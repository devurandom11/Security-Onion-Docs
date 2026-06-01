# Security Onion 3.x Distributed Deployment — Proxmox Build Specification

> **Purpose of this document.** This is a complete, implementation-ready requirements
> specification for building a Security Onion 3.x distributed grid as virtual machines on a
> fresh 5-node Proxmox VE cluster. It is written so that an engineer (or an AI agent) can
> author Ansible playbooks to **clone** VMs from prebuilt Proxmox templates, perform all
> **Proxmox-layer configuration** (OVS virtual-NIC mirroring, CPU pinning, NUMA), drive
> **Security Onion role setup and grid join**, and **configure/operate** the grid — without
> needing to consult any other source first.
>
> **It deliberately contains no Ansible.** It describes *what* must be true, in what *order*,
> with *which values*, and *why* — leaving the *how* (modules, roles, idempotency) to the
> playbook author.
>
> **Operating model (read first):** Security Onion is **installed once into a single
> generalized Proxmox template** (OS + SO ISO content, `so-setup` NOT yet run). That OS/package
> *installation* step is **out of scope for Ansible**. **Everything from clone onward is in
> scope for Ansible** — including driving `so-setup` itself (see §6), because `so-setup` binds
> hostname/IP/TLS-certs/Salt-identity permanently and therefore **cannot** be pre-baked into a
> reusable template for multi-instance roles (e.g., two search nodes). See §0.1 for the exact
> scope split.
>
> All Security Onion facts below are drawn from the official 3.x documentation
> (`docs/hardware.md`, `docs/architecture.md`, `docs/proxmox.md`, `docs/installation.md`,
> `docs/firewall.md`, `docs/configuration.md`, `docs/elasticsearch.md`,
> `docs/full-packet-capture.md`, `docs/suricata.md`, `docs/salt.md`, `docs/soup.md`,
> `docs/elastic-agent.md`, `docs/elastic-fleet.md`, `docs/ntp.md`, `docs/network-installation.md`)
> and from the Security Onion `setup/so-setup` + `setup/so-whiptail` source (branch `3/main`).

---

## 0. Workload summary (the inputs that drove the sizing)

| Parameter | Value |
|---|---|
| Deployment type | Distributed (dedicated manager + search nodes + sensor + fleet + receiver) |
| Analysts (concurrent) | 20 |
| Endpoints (peak, simultaneous) | 100, with Elastic Agent (logs + Osquery) |
| Monitoring scope | Network monitoring **and** endpoint monitoring |
| Simulated traffic profile | Per endpoint ≈ one user browsing ~2 hrs/day ≈ **~1 GB/day/endpoint** |
| Aggregate traffic | ≈ **100 GB/day**, **~100 Mbps peak** if all endpoints active at once |
| PCAP / log retention | **14 days** |
| Hypervisor | Proxmox VE cluster, **5 physical hosts** |
| NIC passthrough available? | **No** — monitoring must use **virtual NICs** (accepted tradeoff) |

These inputs make this a **light-throughput grid**. Sizing is driven by **analyst search
concurrency (20)** and **endpoint agent management (100)**, *not* by packet volume.

---

## 0.1 Scope split — Template build vs. Ansible

| Concern | Owner | Notes |
|---|---|---|
| OS install (Oracle Linux 9) | **Template (manual, one-time)** | Baked into the golden base template |
| Security Onion ISO content / packages on disk | **Template (manual, one-time)** | `so-setup` **NOT** run in the template |
| Template generalization (clear `machine-id`, SSH host keys, salt minion id; enable cloud-init) | **Template (manual, one-time)** | Mandatory so clones get unique identity (§5, §6.1) |
| **Clone** VM from template | **Ansible (Proxmox API)** | §6.2 |
| Per-clone personalization: hostname, static IP/mask/gateway, DNS, NTP | **Ansible (cloud-init)** | Must be final **before** `so-setup` (§5, §6.2) |
| Proxmox CPU type=`host`, cores, RAM, disks, NIC model/count | **Ansible (Proxmox API)** | §4 |
| **OVS virtual-NIC creation + port mirror** for the sensor | **Ansible (Proxmox host)** | §7 |
| **CPU pinning / NUMA / vhost-net pinning** | **Ansible (Proxmox host)** | §8 |
| Host-side **NIC offload disable** (sniff path) | **Ansible (Proxmox host)** | §7.3 |
| Run **`so-setup`** (role bind + grid join) | **Ansible (in-guest, via `expect`)** | §6.3 — the one nuance; see §6 |
| Manager-side **node acceptance** (salt-key / firewall hostgroup / API) | **Ansible (manager)** | §11 |
| **Create users** (SOC/Elastic analyst accounts ×20) | **Ansible** | §9.8 |
| **Check `so-status`** / grid health | **Ansible** | §16 |
| **Configure SO settings** (ILM/retention, PCAP maxsize, Suricata threads, firewall, Fleet) | **Ansible** | §9, §10, §12, §15 |
| **Updates** (`soup`) | **Ansible (day-2)** | §17 |
| Endpoint Elastic Agent rollout (the 100 endpoints) | **Ansible (or MDM)** | §15 |

> **Bottom line:** the template stops at "OS + SO bits on disk, generalized." Ansible does the
> clone, all Proxmox plumbing, the `so-setup` run, grid acceptance, and all day-1/day-2 config.

---

## 1. Grid topology — 7 VMs

| # | Hostname (suggested) | Role | Proxmox host (anti-affinity) |
|---|---|---|---|
| 1 | `so-manager` | Manager (dedicated; **New Deployment → MANAGER**) | Host A |
| 2 | `so-search01` | Search node | Host B |
| 3 | `so-search02` | Search node | Host C |
| 4 | `so-sensor01` | Sensor (forward node: Suricata + Zeek + PCAP) | Host D |
| 5 | `so-fleet01` | Elastic Fleet (standalone) node | Host E |
| 6 | `so-receiver01` | Receiver node (Logstash + Redis pipeline redundancy) | Host E |
| 7 | *(optional)* `so-idh01` | Intrusion Detection Honeypot | Host A or B |

**Rationale (per `docs/architecture.md`):**
- A **dedicated manager MUST be paired with one or more search nodes** — otherwise logs queue
  on the manager with nowhere to land. We use **two** search nodes for 20-analyst query
  concurrency and to permit an Elasticsearch replica (resilience).
- A **dedicated Fleet node** is recommended when managing a large number of Elastic Agents;
  it offloads agent management (ports 8220/5055) from the manager.
- A **receiver node** provides pipeline redundancy: if the manager reboots, agents fail over
  to the receiver and search nodes keep draining its Redis queue.
- **IDH** is optional; include only if honeypot detection is in scope.

> **Do NOT** make the manager a sensor. `docs/configuration.md`: "the manager node should not
> do any network sniffing, that should be handled by dedicated sensor nodes."

---

## 2. Per-VM resource specification

Two columns: **Minimum** = Security Onion's documented bare minimum (`docs/hardware.md`);
**Recommended** = what to actually allocate for *this* workload. Provision the **Recommended**
values.

| Node | Min vCPU | Min RAM | Min Disk | **Rec vCPU** | **Rec RAM** | **Rec Disk** | NICs |
|---|---|---|---|---|---|---|---|
| Manager | 4 | 16 GB | 200 GB | **8** | **32 GB** | **250 GB SSD** | 1 (mgmt) |
| Search ×2 | 4 | 16 GB | 200 GB | **8** | **32 GB** | **1.5–2 TB SSD (RAID 10 backing)** each | 1 (mgmt) |
| Sensor | 4 | 12 GB | 200 GB | **8** | **16 GB** | **2 TB** (200 GB OS + ~1.8 TB `/nsm` PCAP) | 2 (mgmt + sniff) |
| Fleet | 4 | 4 GB | 200 GB | **6** | **8 GB** | **200 GB** | 1 (mgmt) |
| Receiver | 2 | 8 GB | 200 GB | **4** | **16 GB** | **200 GB** | 1 (mgmt) |
| IDH (opt) | 2 | 1 GB | 12 GB | **2** | **2 GB** | **100 GB** | 1 (mgmt) |

**Cluster totals (recommended, excluding optional IDH):** ~42 vCPU · ~136 GB RAM · ~6 TB storage.

### 2.1 Storage sizing math (must be reproduced in playbook variables)

- **Sensor PCAP** (`docs/hardware.md` formula): 50 Mbps ≈ 540 GB/day. Here ≈100 GB/day →
  **~1.4 TB for 14 days**; provision **~1.8 TB usable `/nsm`** for headroom. Configure
  Suricata `maxsize` (total PCAP GB) accordingly — see §10.2.
- **Search-node Elasticsearch** (`/nsm/elasticsearch`): dominated by **endpoint/Osquery logs**
  from 100 agents (~30–50 GB/day indexed), not network metadata. 14 days hot + 1 replica
  across 2 nodes ≈ **~1–1.4 TB total** → **1.5–2 TB per search node**. Account for the
  Elasticsearch **80% watermark** (keep ≥20% free) and ILM (see §9).
- **Manager / Fleet / Receiver**: mostly OS + Docker images + Redis queue; 200–250 GB each.

### 2.2 Partition layout per VM (`/nsm` is where the data lives)

The **golden base template carries the partition layout** (the SO ISO auto-partitions when the
template is built — §6.1). Ansible's job at clone time is to **size/attach the role `/nsm` disk**
and **grow the `/nsm` LVM volume** to match §2 (e.g., 1.8 TB on the sensor, 1.5–2 TB per search
node). The layout the template should embody (`docs/network-installation.md`):

- `/boot` — ≥1 GB at the start of the disk.
- `/` (root) — ≥100 GB (holds `/var/lib/docker`; current image set ≈30 GB, plus a second set
  for in-place updates).
- `/nsm` — the **vast majority** of the disk (Elasticsearch on search nodes, PCAP on sensor).
- `swap` — ~8 GB (over-provision RAM so swap is effectively unused; `docs/hardware.md`).
- `/tmp` — ~2 GB.
- Use **LVM** so partitions can be grown later.

> **Hard constraint (`docs/hardware.md`):** **Local storage ONLY.** SAN/iSCSI/FC/NFS for `/nsm`
> is **unsupported** and harms performance/reliability. On a Proxmox *cluster*, do **NOT** back
> `/nsm` disks (search + sensor especially) with Ceph or shared/NFS storage. Pin those VM disks
> to **local SSD/NVMe** (RAID 10 recommended for Elasticsearch). This also constrains live
> migration — treat search and sensor VMs as **non-migratable**.

---

## 3. Proxmox cluster prerequisites (provisioning layer)

The Ansible author must ensure these are true on the Proxmox hosts before/while creating VMs.

1. **Cluster**: 5 PVE nodes already joined into one cluster (quorum healthy). Document each
   host's CPU core count and NUMA topology (see §8).
2. **Local datastores**: each host has a **local** SSD/NVMe-backed storage target for VM disks
   (e.g., `local-lvm` or a local ZFS pool). Search and sensor disks **must** land here, not on
   shared storage.
3. **VM CPU type = `host`** (`docs/proxmox.md`): the default emulated CPU hides host features;
   set CPU type to `host` on every SO VM so Suricata/Zeek/Elasticsearch get full instruction sets.
4. **Open vSwitch** installed on at least the host(s) participating in traffic mirroring
   (`apt install openvswitch-switch`) — required for the monitoring design in §7.
5. **Network bridges** defined (see §7): a management bridge (e.g., `vmbr0`, has gateway) and a
   **monitoring/mirror bridge** (e.g., OVS `vmbr1`, no IP).
6. **Golden base template present**: the generalized, pre-`so-setup` Security Onion template
   (built once from the verified ISO — `docs/download.md`, §6.1) exists on the cluster and is
   reachable for cloning. **Always verify the ISO checksum** when (re)building the template.
7. **Time sync on the hosts themselves** (NTP) — distributed SO is clock-sensitive (§13).
8. **x86-64 only** (`docs/hardware.md`): ARM and other architectures are unsupported. (Native
   for Proxmox; just don't enable CPU model emulation tricks that break this.)

---

## 4. Per-VM Proxmox hardware definition (what to set when creating each VM)

For **every** SO VM:
- **CPU type**: `host`.
- **Cores/sockets**: set `cores` to the Recommended vCPU count from §2; keep a single socket
  unless aligning to multiple physical NUMA nodes (see §8). Enable `numa: 1` on the larger VMs
  (manager, search, sensor).
- **Memory**: fixed (no ballooning) for search and sensor nodes to keep Elasticsearch
  heap/Lucene cache and capture buffers stable. Disable the balloon device on those.
- **Disk bus**: VirtIO SCSI (`virtio-scsi-single`), `cache=none` (or `writeback` only on
  battery/UPS-backed storage), `discard=on`, `iothread=1` for the data disks on search/sensor.
- **NIC model**: **VirtIO (paravirtualized)** for both management and sniffing interfaces.
- **Boot order**: disk-first (clones boot the template's installed OS; no ISO step at clone time).
- **cloud-init drive**: attach a cloud-init drive so Ansible can inject hostname, static IP,
  DNS, NTP, and SSH keys at first boot (§6.2). Requires cloud-init in the template image.
- **Display**: default is fine for headless nodes. Only if a node will run Mono apps
  (NetworkMiner) set Display = `VMware compatible (vmware)` per `docs/proxmox.md` (applies to a
  desktop/analyst VM, not the grid nodes).
- **QEMU guest agent**: enable (helps Ansible/Proxmox query IPs and do graceful shutdown).
- **UPS/graceful shutdown**: production deployments should sit behind a UPS (`docs/hardware.md`).

NIC count by role:
- **Sensor**: **2 vNICs** — `net0` management (VirtIO, on `vmbr0`, has IP), `net1` sniffing
  (VirtIO, on the OVS mirror bridge `vmbr1`, **no IP**, firewall disabled). See §7.
- **All other nodes**: **1 vNIC** management (VirtIO, on `vmbr0`, has IP).

> **Per-VM hardware is immutable in some flows** — and SO does **not** support changing
> hostname or IP after `so-setup` (§5). Get cores/RAM/disk/IP/hostname right *before* running
> `so-setup`.

---

## 5. Hard SO constraints the playbooks must respect (read before designing)

1. **Hostname is permanent.** `so-setup` generates TLS certs and the Salt minion identity from
   the hostname; **changing the hostname after setup is unsupported** (`docs/installation.md`,
   `docs/hostname.md`). The hostname must be set (via cloud-init) **before** `so-setup` runs.
2. **IP address is effectively permanent.** `so-ip-update` is experimental and **standalone-only**
   (`docs/ip-address.md`). Use **static IPs** assigned (via cloud-init) before `so-setup`; never
   DHCP-churn a grid node.
3. **Templates MUST be generalized.** Because of (1) and (2), a template that has already run
   `so-setup` **cannot be cloned** into more than one node of a role (the two search nodes would
   share certs/minion-id and the join breaks). The template must therefore contain **OS + SO
   bits but NOT a completed `so-setup`**, and must be generalized before being converted to a
   template: clear `/etc/machine-id`, regenerate/remove SSH host keys on first boot, remove any
   salt minion id/keys, and enable cloud-init. (See §6.1.)
4. **Manager must be built first**, fully, before any other node joins (`docs/configuration.md`).
5. **Node join is a two-party handshake** requiring manager-side acceptance (firewall hostgroup
   + Grid Members "Accept") — see §11. Playbooks must orchestrate both sides.
6. **`so-setup` is interactive — but automatable.** It is a whiptail TUI with **no documented**
   unattended flag. Two automation paths exist (see §6.3): (a) **Ansible `expect`** driving the
   real prompts with production values (uses the supported path; pin to a version), or (b) the
   **built-in `TESTING` bypass** (pass a second "profile" arg → every prompt short-circuits) —
   but the stock profiles **hardcode test-grade values** (DHCP, default creds, `10.0.0.0/8`,
   `0.0.0.0/0`, throwaway hostnames) and are **undocumented/unsupported for production**.
   **Recommended: path (a).** This is the single biggest automation risk — budget for it.
7. **Multi-node Elasticsearch deletion is ILM-only.** `so-elasticsearch-indices-delete` is
   **disabled on multi-node deployments**; retention MUST be enforced via ILM `min_age` (§9).
8. **Do not hand-edit iptables.** The host firewall is **Salt-managed**; all firewall changes go
   through SOC Configuration → firewall (or the documented `so-firewall`/grid sync), never raw
   `iptables` (`docs/firewall.md`).

---

## 6. Template model, cloning, and `so-setup` automation

### 6.1 Golden base template (built ONCE, manually — out of Ansible scope)

- Install from the **official Security Onion ISO** (`docs/installation.md`). For distributed
  deployments the ISO is the **only supported** install method; **network installs are NOT
  supported for distributed** (`docs/network-installation.md`). Base OS is **Oracle Linux 9**;
  the ISO auto-partitions (`/boot`, LVM with `/nsm`, `/`, `swap`, `/tmp` — see §2.2).
- **Stop before `so-setup` completes.** The template should contain the installed OS and the
  staged SO content (the ISO drops the SecurityOnion tree under `/root/SecurityOnion`, and
  `so-setup` can be re-run with `sudo SecurityOnion/setup/so-setup iso`) but must **not** have a
  finished `so-setup` (no role, no certs, no grid identity).
- **Generalize**, then convert to a Proxmox template:
  - truncate `/etc/machine-id`; arrange SSH host-key regeneration on first boot;
  - remove any salt minion id / keys and prior setup artifacts (e.g., `/root/accept_changes`,
    `/root/manager_setup`, setup logs) so a clone is detected as a fresh install;
  - install/enable **cloud-init** and the **QEMU guest agent**;
  - leave a known bootstrap credential / SSH key for Ansible to connect.
- One single base template covers all roles (role is chosen later by `so-setup`). A per-role
  template set is **not** recommended — it provides no benefit and reintroduces the clone
  identity problem for multi-instance roles.

> **Disk sizing at clone time:** the template's base disk is small; Ansible should **resize the
> boot disk and/or attach the role-specific `/nsm` data disk** (per §2) on the clone before
> first boot, and grow the `/nsm` LVM volume in-guest via cloud-init.

### 6.2 Clone + personalize (Ansible)

Per node, before any SO setup:
1. **Clone** the base template to a new VMID on the target Proxmox host (full clone onto local
   SSD/NVMe storage — §3, §5.3).
2. Apply **hardware** per §4 (cores, RAM, disks, NIC count/model, CPU type `host`, NUMA).
3. Inject via **cloud-init**: final **hostname**, **static IP/mask/gateway**, **DNS**, **NTP**,
   Ansible SSH key. These are immutable post-`so-setup` (§5) so they must be correct now.
4. First boot; confirm guest agent + SSH reachability; grow `/nsm` if needed.

### 6.3 Driving `so-setup` non-interactively (Ansible)

`so-setup`'s prompts (whiptail) each short-circuit when `TESTING` is set, otherwise they collect
the variables below. Whichever path is chosen, the **answer set** is the same:

| Variable | Meaning | Example / source |
|---|---|---|
| `setup_type` (arg 1) | `iso` (template was built from ISO) | `iso` |
| `install_type` | Role | `MANAGER`, `SEARCHNODE`, `SENSOR`, `FLEET`, `RECEIVER`, `IDH` |
| `HOSTNAME` | Node hostname | must match the cloud-init hostname |
| `address_type` | `STATIC` or `DHCP` | **STATIC** |
| `MNIC` | Management NIC name | e.g. `eth0` |
| IP / mask | Management address | per inventory |
| `MGATEWAY` | Default gateway | per inventory |
| `MDNS` | DNS server(s) | per inventory |
| `MSRV` / `MSRVIP` | Manager hostname / IP (joining nodes only) | the manager |
| `BNICS` | Sniffing NIC(s) (**sensor only**, no IP) | e.g. `eth1` |
| NTP servers | Time sources | per inventory (§14) |
| `WEBUSER` / `WEBPASSWD1` / `WEBPASSWD2` | Initial SOC/Elastic admin (manager) | from vault |
| `NODE_DESCRIPTION` | Free-text node label | e.g. `so-search01 - SEARCHNODE` |
| `ALLOW_CIDR` / `ALLOW_ROLE` | Initial allowed analyst CIDR/role | analyst mgmt subnet |
| telemetry accept | Accept/decline telemetry prompt | per policy |

**Path (a) — Ansible `expect` (recommended, supported flow).** Use `ansible.builtin.expect`
(or a pexpect wrapper) to run `sudo SecurityOnion/setup/so-setup iso` and respond to each
whiptail prompt with the values above. Pin the SO version, and keep a prompt→response map that
is version-tested (`docs`/UI text can change between releases).

**Path (b) — built-in `TESTING` bypass (fast, UNSUPPORTED for prod).** Passing a second
positional "profile" arg (e.g. `distributed-search-iso`) sets `TESTING=true`, skipping all
prompts. **But** the stock profile block hardcodes `address_type=DHCP`, default web creds, a
`/8` minion CIDR, `ALLOW_CIDR=0.0.0.0/0`, and throwaway hostnames — i.e. **not production-safe**
and undocumented. Only consider this if you also override those hardcoded values (effectively
forking the setup flow) and you accept it is unsupported and version-fragile.

**Run order:** MANAGER first (New Deployment), then SEARCHNODE ×2, then RECEIVER, FLEET,
SENSOR, (IDH) as Existing-Deployment joins pointing at `MSRV`/`MSRVIP` — see §11.

> **Network-install fallback** (unsupported; only if the ISO truly cannot be used to build the
> template): install Oracle Linux 9, `sudo dnf -y install git`,
> `git clone -b 3/main https://github.com/Security-Onion-Solutions/securityonion`,
> `cd securityonion`, `sudo bash so-setup-network`. **Do not use for the production grid.**

---

## 7. Network architecture

### 7.1 Networks / bridges

| Network | Proxmox bridge | IP? | Purpose |
|---|---|---|---|
| Management | `vmbr0` (Linux or OVS) | Yes (static per node) | SO node↔node, Salt, SOC/Kibana, agent ingest, analyst access |
| Monitoring/mirror | `vmbr1` (**OVS**) | **No** | Carries mirrored endpoint traffic to the sensor's sniff NIC |

- All 7 nodes have a management interface on `vmbr0` with a **static IP** and the correct
  **DNS** and **gateway**.
- Only the **sensor** has a second interface on `vmbr1` (no IP) for sniffing.

### 7.2 Getting traffic to the sensor with **virtual NICs** (no passthrough)

Because the simulated endpoints are VMs and NIC passthrough is unavailable, mirror endpoint
traffic to the sensor's sniffing vNIC using **Open vSwitch port mirroring** (`docs/proxmox.md`
sanctions virtual-NIC sniffing; OVS provides the clean mirror primitive).

**Chosen pattern — single chokepoint mirror (recommended for this lab):**
1. Funnel all simulated endpoint traffic through **one bridge / one virtual gateway VM** so
   there is a single place where all traffic is visible. Endpoints may run on any of the 5
   hosts, but their traffic transits this chokepoint host.
2. Place the **sensor's sniff vNIC** on the same OVS bridge (`vmbr1`) at that chokepoint.
3. Configure an OVS mirror that copies **all** bridge traffic to the sensor's sniff port. The
   sensor's sniff tap appears on the host as `tap<SENSOR_VMID>i1`. Conceptually:
   - create a Mirror on `vmbr1` with `select-all=true` and `output-port` = the sensor sniff port.
4. The sniff vNIC inside the sensor VM has **no IP** and SO uses **AF-PACKET** to capture
   (`docs/af-packet.md`).

**Alternative — per-host mirror + tunnel (only if a chokepoint isn't feasible):** mirror on each
host's OVS bridge and forward copies to the sensor host via **OVS ERSPAN/GRE**. More complex;
document only as a fallback.

### 7.3 NIC offloading MUST be disabled on the host sniffing path

This is the #1 correctness issue for virtual-NIC capture. VirtIO offloads (GRO/GSO/TSO/LRO)
coalesce packets into super-frames larger than MTU, which corrupts what Zeek/Suricata see.
Security Onion disables offload **inside** the VM on the sniff interface automatically, **but
the playbook must also disable it on the Proxmox host** for the physical sniffing NIC and its
bridge (`docs/proxmox.md`):

- Add `post-up` commands in `/etc/network/interfaces` on the relevant Proxmox host(s) for the
  physical sniff interface **and** its bridge, e.g. (doc-provided form):
  - on the physical iface (e.g. `enp2s0`): `post-up for i in rx tx sg tso ufo gso gro lro; do ethtool -K enp2s0 $i off; done`
  - on the bridge (e.g. `vmbr1`): `post-up for i in rx tx sg tso ufo gso gro lro; do ethtool -K vmbr1 $i off; done`
- Also disable offloads on the **sensor's sniff tap** (`tap<VMID>i1`) on each VM start — taps are
  recreated per boot, so make this persistent (Proxmox **hookscript** on `post-start`, or a
  systemd unit).
- **Proxmox 9+**: additionally set `mtu 9000` on the physical sniff interface and its bridge
  (`docs/proxmox.md`).
- If the Grid later reports **Capture Loss but no Zeek Loss**, the host NIC channels may need
  pinning: add `ethtool -L <iface> combined 1` in the same `post-up` (`docs/proxmox.md`).

> **Accepted tradeoff:** even with offloads disabled, virtual-NIC capture may drop some packets
> under bursts. Acceptable at ~100 Mbps; document it as a known limitation.

### 7.4 Multiqueue (optional, helps Zeek/Suricata parallelism)

Optionally set the sniff vNIC to multiqueue (e.g., `queues=4`) so RX work spreads across vCPUs,
and enable RPS/RSS inside the VM. **Optional at ~100 Mbps**; Suricata/Zeek worker counts (§10)
matter more.

---

## 8. CPU pinning & NUMA (host layer)

`docs/suricata.md`: pin sniffing processes to a CPU in the **same NUMA domain** as the sniffing
NIC; cross-NUMA access is slower. Requirements for the playbook:

1. **Discover topology per host**: capture `lscpu`, `lscpu | grep NUMA`, and `numactl --hardware`
   so each VM's pinned cores stay within one NUMA node.
2. **Pin each VM's vCPUs to host cores** using Proxmox **CPU affinity** (PVE 7.3+):
   - CLI: `qm set <VMID> --affinity "<cpuset>"` (e.g., `--affinity 2-9`).
   - GUI: VM → Hardware → Processors → Advanced → **affinity** (cpuset list like `2-9` or `2,4,6,8`).
3. **Reserve cores for the hypervisor**: never pin VMs onto cores 0–1; leave them for the PVE
   host, storage (ZFS), and networking.
4. **Give the sensor exclusive cores** (no overlap with other VMs) so packet processing is not
   preempted. Align the sensor's cores to the **same NUMA node** as `vmbr1`/the sniff path.
5. **Enable guest NUMA** (`qm set <VMID> --numa 1`) on manager/search/sensor and keep their
   affinity within a single node.
6. Optional softer controls: `cpuunits` (relative weight), `cpulimit` (cap). **Affinity is the
   hard pin.**

**Example core map for the sensor host (16-core example):**

| Host cores | Assigned to |
|---|---|
| 0–1 | Proxmox host / storage / networking (unpinned) |
| 2–9 | Sensor VM vCPUs (`--affinity 2-9`) |
| 10–11 | Sensor sniff-vNIC `vhost-net` threads (see §8.1) |
| 12–15 | Other VMs on this host |

### 8.1 Pinning the virtual NIC processing (optional polish)

VirtIO NIC packets are serviced by host **`vhost-net`** kernel threads (one per queue) plus the
QEMU emulator thread. To pin them:
- Identify threads: `ps -eLo pid,comm | grep vhost`.
- Pin to reserved cores: `taskset -pc 10,11 <vhost_tid>`.
- Threads are recreated on VM start → wrap in a Proxmox **hookscript** (`post-start`).

> At ~100 Mbps this is **optional**. Priorities are: (1) offload-off on the host (§7.3),
> (2) exclusive pinned vCPUs for the sensor. Do those first.

---

## 9. Elasticsearch / data-tier configuration (search nodes + manager)

All values below are set in **SOC → Administration → Configuration → Elasticsearch** (many
require enabling "Show advanced settings"). The playbook should treat these as
post-deployment configuration tasks (ideally via the SO config mechanism / Salt pillars, not
hand-edited files).

1. **Heap size** (`docs/elasticsearch.md`): Setup auto-sets heap to **33% of RAM, capped at 25 GB**
   when RAM ≥8 GB. With 32 GB search nodes, heap ≈ ~10–11 GB — acceptable. Adjust under
   `Elasticsearch → esheap` only if performance requires.
2. **Replicas**: with **2 search nodes** you can run **1 replica** for resilience (this doubles
   index storage — already accounted for in §2.1). Note the single-node replica pitfall in the
   docs does **not** apply here because there are 2 data nodes.
3. **Retention enforcement = ILM `min_age`** (mandatory on multi-node):
   - `so-elasticsearch-indices-delete` is **disabled** on multi-node grids — do **not** rely on it.
   - Set global ILM delete: `Elasticsearch → index_settings → global_overrides → policy → phases
     → delete → min_age` = **14d** (matches the 14-day requirement). Remember `min_age` is
     relative to **rollover date, not creation date**.
   - Goal: delete indices **before** hitting the Elasticsearch **80% watermark**; keep ≥20% free.
4. **`retention_pct`** (`Elasticsearch → retention → retention_pct`): default **50%** is tuned for
   standalone (where PCAP shares the disk). For **dedicated search nodes** where Elasticsearch is
   the main disk consumer, **increase** this (e.g., toward 80% but staying under the watermark).
   Note this is size-based and takes priority over time-based ILM deletion.
5. **Shards** (`docs/elasticsearch.md`): keep shard size **< 50 GB**; aim 20–40 GB. Keep
   **< 20–25 shards per GB of heap** per node. At this data volume, default shard counts are
   fine; only raise `number_of_shards` for an index if it averages >50 GB/shard.
6. **Validation utilities** the playbook can call for verification:
   `sudo so-elasticsearch-indices-list`, `so-elasticsearch-shards-list`,
   `so-elasticsearch-indices-growth`, `so-elasticsearch-retention-estimate`.
7. **GeoIP** (optional): enabling Elastic GeoIP updates requires egress to `geoip.elastic.co`
   and `storage.googleapis.com` (§12).

### 9.8 User account creation (the 20 analysts)

- The **initial admin** account is created during the manager's `so-setup` run
  (`WEBUSER`/`WEBPASSWD`, §6.3).
- **Additional analyst accounts** (×20) are created post-deploy. Creating an account through
  Security Onion provisions it in **both SOC and Elasticsearch at once** (`docs/adding-accounts.md`,
  `docs/elasticsearch.md`). **Do not** create accounts directly in Elastic — those get Elastic
  access only, not SOC.
- The supported CLI utility is **`so-user`** (`docs/so-user.md`) — Ansible should drive
  `so-user` on the manager (e.g., add/list/enable/disable) to create the analyst accounts
  idempotently, sourcing usernames and initial passwords from a vault.
- Consider **RBAC roles** (`docs/rbac.md`) and **MFA** (`docs/mfa.md`) for analyst accounts; both
  are SO-managed and configurable post-deploy. Optionally integrate **OIDC** (`docs/oidc.md`) if
  using an external IdP for the 20 analysts.
- Password/lockout policy lives under SOC Configuration (`docs/passwords.md`).

---

## 10. Sensor capture configuration (Suricata / Zeek / PCAP)

### 10.1 Engine worker counts (AF-PACKET threads)

- Capture uses **AF-PACKET** with fanout load-balancing (`docs/af-packet.md`).
- Sizing rule (`docs/hardware.md`): ~**200 Mbps per Suricata worker and per Zeek worker**. At
  **~100 Mbps peak**, a **single worker per engine** technically suffices; set a small worker
  count (e.g., **2 each**) for headroom.
- Configure Suricata AF-PACKET threads: `Configuration → Suricata → config → af-packet → threads`.
- Pin Suricata/Zeek workers to the sensor's NUMA-local cores (§8).
- Optional CPU-saving alternative (`docs/suricata.md`): run **Suricata for both alerts and
  metadata** and disable Zeek. Not necessary at this scale; keep **Zeek** for richer protocol
  metadata unless CPU-constrained.

### 10.2 Full packet capture retention (`docs/full-packet-capture.md`)

PCAP is written by Suricata. Configure under `Configuration → Suricata`:
- **`maxsize`** — max **total** disk (GB) for all PCAP. Set to fit the ~1.8 TB `/nsm` budget
  (e.g., ~1500–1700 GB, leaving room for logs/overhead) so ~14 days of ~100 GB/day fits.
- **`filesize`** — per-file PCAP size (larger = better write perf, slower retrieval).
- **`compression`** / **`lz4-level`** — `none` (default) or `lz4` (more CPU). Leave `none` at
  this volume.
- **`use-stream-depth`** — `no` to capture full flows, `yes` to truncate by stream depth.

> PCAP storage is **size-capped** by `maxsize`, independent of Elasticsearch ILM. Keep both the
> PCAP cap and the ES retention aligned to 14 days.

---

## 11. Deployment sequence (strict ordering — the heart of the automation)

Security Onion's distributed join is **manager-first** and requires **manager-side acceptance**
of each joining node (`docs/configuration.md`). Every node follows the same **per-node task
chain**; the *order across nodes* is fixed (manager → search → receiver/fleet/sensor → IDH).

### 11.0 Per-node task chain (applies to every clone)

1. **Clone** the golden base template → new VMID on the assigned host (§6.2, anti-affinity §1).
2. **Set hardware** (§4): cores, RAM, role disks, NIC count/model, CPU type `host`, NUMA.
3. **cloud-init personalize** (§6.2): final hostname, static IP/mask/gateway, DNS, NTP, SSH key.
4. **Proxmox host plumbing** *(sensor especially)*: OVS sniff vNIC + mirror (§7), offload-disable
   hookscript (§7.3), CPU affinity / NUMA pin (§8), vhost pin (§8.1).
5. **First boot**; wait for guest agent + SSH; grow `/nsm` LVM if required.
6. **Run `so-setup`** non-interactively (§6.3, `expect`) with the role's answer set.
7. **Accept on the manager** (§11.6) — firewall hostgroup + grid membership.
8. **Verify** (`so-status`, grid green) before moving to the next node (§16).

### 11.1 Manager (must complete fully first)
- Task chain with `install_type=MANAGER`, **New Deployment**.
  (`MANAGERSEARCH` would make separate search nodes optional, but for 20 analysts the
  **dedicated MANAGER + separate search nodes** design is preferred — §1.)
- Supply the initial admin `WEBUSER`/`WEBPASSWD` (from vault).
- Gate: `sudo so-status` all green before proceeding (a new node may briefly show red Fault).

### 11.2 Search nodes ×2 (before any other node type)
- Task chain with `install_type=SEARCHNODE`, **Existing Deployment**, `MSRV`/`MSRVIP`=manager.
- Accept each on the manager, then confirm it joins the Elastic cluster and indexes.
- Search nodes must reach the manager on **443, 4505, 4506, 5000, 5055, 8086, 8220, 8443, 9696**
  and participate in **9200/9300** cluster traffic (§12).

### 11.3 Receiver node
- Task chain with `install_type=RECEIVER`, Existing Deployment; accept on manager.

### 11.4 Fleet node
- Task chain with `install_type=FLEET`, Existing Deployment; accept on manager.

### 11.5 Sensor
- Task chain with `install_type=SENSOR`, Existing Deployment; **`BNICS`** = the sniff NIC, which
  must have **no IP** (the management NIC holds the only IP). `docs/configuration.md` warns if a
  non-management interface has an IP (usually a sniff NIC wrongly on a DHCP port). Ensure the
  Proxmox plumbing in step 4 (OVS mirror + offload-off) is in place **before** setup.
- *(optional)* **IDH**: `install_type=IDH`, Existing Deployment; manager reaches it on **2222/tcp**.

### 11.6 Node acceptance automation (manager side)
The UI "Accept" (SOC → Administration → **Grid Members** → Pending → Review → Accept) corresponds
to **Salt key acceptance + grid membership** on the manager, plus adding the node's IP to the
correct **firewall hostgroup** (SOC → Configuration → firewall → hostgroups). For full
automation, the playbook should drive these on the manager rather than the GUI:
- Salt key handling via `salt-key` on the manager (accept the new minion).
- Firewall hostgroup membership via the SO config mechanism / `so-firewall`.
- Investigate the **SO REST API** (`docs/connect-api.md`) for grid-member acceptance and
  hostgroup updates. **Treat the exact acceptance API/CLI as a verification item** during build.

### 11.7 Post-deploy configuration (then §9, §10, §12–§15)
ILM `min_age`=14d + `retention_pct` (§9), PCAP `maxsize` (§10.2), Suricata AF-PACKET threads
(§10.1), analyst firewall hostgroup + analyst accounts ×20 (§9.8, §12.3), NTP (§14), Fleet
enrollment for the 100 endpoints (§15).

---

## 12. Firewall / port matrix (network firewalls + SO host firewall)

The SO **host firewall is Salt-managed**; open node-to-node access by adding IPs to **hostgroups**
(SOC → Configuration → firewall → hostgroups), never via raw iptables. Any **network** firewalls
between VLANs/hosts must also permit the following (`docs/firewall.md`):

### 12.1 Intra-grid (node-to-node) — TCP

| Source | Destination | Ports (TCP) | Purpose |
|---|---|---|---|
| All grid nodes | Manager | 443, 4505, 4506, 5000, 5055, 8086, 8220, 8443 | Mgmt, registry, Salt, updates |
| All grid nodes | Fleet node | 5055, 8220 | Elastic Agent data + management |
| All grid nodes | Receiver node | 5055 | Elastic Agent data |
| Search nodes | Manager | 443, 4505, 4506, 5000, 5055, 8086, 8220, 8443, **9696** | + Redis |
| Elastic cluster nodes | Elastic cluster nodes | **9200, 9300** | Logstash→ES and ES node-to-node |
| Fleet node | Receiver node | 5056 | Logstash-to-Logstash |
| Fleet node | Manager | 5056, 9200 | Logstash-to-Logstash + ES |
| Manager | IDH node | 2222 | SSH management |

> "Elastic cluster nodes" = the search nodes **and** the manager. Salt requires **4505/4506**
> from every minion to the manager (`docs/salt.md`).

### 12.2 Endpoint Elastic Agents (the 100 endpoints) — TCP

| Source | Destination | Ports (TCP) | Purpose |
|---|---|---|---|
| Endpoint agents | Manager | 8220, 8443, 5055 | Agent mgmt, binary updates, data |
| Endpoint agents | Fleet node | 5055, 8220 | Agent mgmt + data |
| Endpoint agents | Receiver node | 5055 | Agent data |

### 12.3 Analyst access

- The 20 analysts reach SOC/Kibana over the manager's **nginx (80/443)**. Add analyst source IPs
  to the **`analyst`** hostgroup (SOC → Configuration → firewall → hostgroups). Emergency CLI:
  `sudo so-firewall includehost analyst <IP>`.

---

## 13. External connectivity (egress) — non-airgap

Ensure the grid (primarily the manager) can reach **outbound TCP/443** to (`docs/firewall.md`):

- `raw.githubusercontent.com`, `sigs.securityonion.net`, `ghcr.io`,
  `pkg-containers.githubusercontent.com` — keys, signatures, container downloads.
- `rules.emergingthreatspro.com`, `rules.emergingthreats.net` — IDS rules.
- `github.com`, `objects.githubusercontent.com` — Strelka/Sigma rules.
- ISO-based OS package updates: `repo.securityonion.net`, `repo-alt.securityonion.net`,
  `so-repo-east.s3.us-east-005.backblazeb2.com`.
- Optional GeoIP: `geoip.elastic.co`, `storage.googleapis.com`.
- Optional Snort Talos ruleset: `www.snort.org`.

> If the environment is airgapped instead, follow the **Airgap** process (different repo/source
> handling) — out of scope for this spec but note it as a branch point.

---

## 14. Cross-cutting prerequisites

1. **NTP (critical):** all nodes' clocks **must** be synchronized or data appears missing
   (`docs/ntp.md`). Set NTP at OS install and/or via SOC → Configuration → ntp. Also sync the
   **Proxmox hosts**.
2. **DNS:** configured at install; nodes (esp. manager↔nodes) should resolve each other by name,
   or add `/etc/hosts` entries (the docs call this out for manager→node reachability). Use static
   IPs.
3. **Hostnames:** final, correct, **before** Setup (certs depend on them; immutable after — §5).
4. **SSH:** available for break-glass admin (`docs/post-installation.md`), but most admin is via
   SOC. Manager→IDH SSH uses **2222**.

---

## 15. Endpoint agent rollout (the 100 endpoints)

- Deploy **Elastic Agent** from SOC → **Downloads** (per-OS installer); **do not** use Fleet's
  "Add Agent" button (more manual config) — `docs/elastic-agent.md`.
- Enrollment uses a **token** (from Fleet → Enrollment Tokens) and a **Fleet host**. Installer
  flags: non-MSI `-token=$TOKEN -fleet=$FLEETHOST` (and `-delay-enroll`, `-timeout`); MSI
  `TOKEN=$TOKEN FLEET=$FLEETHOST DELAYENROLL=...`.
- Point endpoint agents at the **Fleet node** (8220 mgmt) and ship data to Fleet/Receiver (5055);
  open the **`elastic_agent_endpoint`** firewall option / hostgroup on the manager
  (`docs/elastic-agent.md`).
- **Agent version must match the Elastic stack version** running in the grid (`docs/elastic-fleet.md`).
- **Agent monitoring:** enable offline/degraded alerting via SOC → Configuration → manager →
  `agent_monitoring` (default threshold 5h) for the 100 endpoints.
- Endpoint/Osquery log volume is the main driver of search-node disk (§2.1) — keep ILM at 14d.

---

## 16. Validation / acceptance tests (post-build)

The playbook should verify, at minimum:
1. `sudo so-status` on every node → all services **OK** (allow a few minutes / `so-checkin`).
2. Grid page (SOC → Grid) → all 7 members **green OK**, no Pending/Fault.
3. Salt connectivity: `salt '*' test.ping` from the manager returns all minions.
4. Elasticsearch: search nodes joined the cluster; `so-elasticsearch-indices-list` shows shards
   on both search nodes; no unassigned (`UN`) shards (`so-elasticsearch-query _cat/shards | grep UN`).
5. Ingest: Suricata alerts + Zeek metadata appear in Dashboards/Hunt from the sensor; PCAP
   retrievable via the PCAP interface.
6. Endpoint data: a test endpoint's Elastic Agent shows **Healthy** in Fleet and its logs/Osquery
   results appear in Hunt/Kibana.
7. Capture quality: Grid shows acceptable Capture/Zeek loss; if "Capture Loss, no Zeek Loss",
   revisit host NIC offload/channels (§7.3).
8. Retention: `so-elasticsearch-retention-estimate` ≈ ≥14 days at projected ingest; ILM delete
   `min_age` = 14d confirmed.
9. Failover smoke test: stop the manager briefly → confirm agents fail over to the receiver and
   search nodes keep draining the receiver's Redis queue (then restart manager).

---

## 17. Maintenance / update model (for the playbooks' day-2 design)

- Updates are applied with **`soup`** (`sudo soup`) — handles SO version updates, hotfixes, and
  OS updates; may self-update and ask to re-run (`docs/soup.md`).
- **Update the manager first**, then nodes (manager controls Salt code/state for the grid).
- After `soup`/reboot, allow a few minutes for the on-boot Salt highstate; force with
  `sudo so-checkin` if services aren't OK within ~15 min.
- Keep ≥1 extra set of Docker image space on `/` (§2.2); old images auto-prune at next version
  update (or `docker system prune -a` after verifying health).
- **Backups:** see `docs/backup.md` (out of scope here) — at minimum protect manager config/Salt
  pillars.

---

## 18. Known limitations & explicit tradeoffs (call these out to stakeholders)

1. **Virtual-NIC sniffing instead of passthrough** → possible packet loss under bursts even with
   offloads disabled. Mitigations in §7.3 reduce but don't eliminate it. Acceptable at ~100 Mbps.
2. **Local-storage-only for `/nsm`** → search and sensor VMs are effectively **non-migratable** and
   **must not** use Ceph/NFS/shared storage. Plan HA accordingly (rebuild-from-config, not live
   migrate, for those two roles).
3. **`so-setup` is interactive** → the single biggest automation gap; it is **driven by Ansible
   via `expect`** (§6.3), pinned to a specific SO version. The undocumented `TESTING` bypass is
   **not** production-safe. Node acceptance is automated manager-side via salt-key/API (§11.6).
4. **Templates must be pre-`so-setup` and generalized** → a post-setup template cannot be cloned
   for multi-instance roles (the two search nodes). `so-setup` therefore runs **per clone**, in
   Ansible's scope, not in the template (§0.1, §5.3, §6).
5. **Hostname/IP are immutable post-setup** → static addressing and final hostnames are mandatory
   cloud-init inputs **before** `so-setup`.
6. **Multi-node ES deletion is ILM-only** → misconfigured ILM can fill disks and stop ingest at the
   80% watermark.

---

## 19. Parameters the Ansible author should externalize (inventory/vars)

Minimum variable set to template the whole build:

- Template/clone: `base_template_id` (or name), `so_version` (pin for `so-setup` `expect`),
  `setup_method` (`expect` | `testing_bypass`), `ci_user`/`ci_ssh_key` (cloud-init bootstrap).
- Per node: `hostname`, `mgmt_ip`, `cidr`, `gateway`, `dns_servers`, `ntp_servers`,
  `proxmox_host`, `vcpus`, `ram_gb`, `root_disk_gb`, `nsm_disk_gb`, `role` (=`install_type`),
  `numa_node`, `cpu_affinity` (cpuset), `node_description`, `mac` (optional).
- `so-setup` answer set (per node, §6.3): `install_type`, `address_type=STATIC`, `mnic`,
  `mgateway`, `mdns`, `msrv`/`msrvip` (joiners), `bnics` (sensor), `allow_cidr`/`allow_role`,
  telemetry choice; manager-only: `web_user`/`web_pass` (vault).
- Sensor extras: `sniff_bridge` (`vmbr1`), `sniff_queues`, `vhost_affinity`, `bnics`.
- Proxmox: `local_storage_target` per host, `mgmt_bridge`, `mirror_bridge` (OVS), mirror
  chokepoint host, host NUMA/core maps, offload-disable iface list.
- SO config: `elastic_replicas`, `ilm_delete_min_age` (=14d), `retention_pct`, `esheap_gb`
  (override or auto), Suricata `af_packet_threads`, `pcap_maxsize_gb`, `pcap_filesize`,
  Zeek-vs-Suricata-metadata toggle.
- Users: `analyst_accounts` (list: username + vaulted initial password + RBAC role), MFA/OIDC
  toggles.
- Fleet/endpoints: `fleet_host`, `enrollment_token`, `agent_version` (must match stack),
  `agent_monitoring_threshold`.
- Firewall: `analyst_source_ips`, `endpoint_agent_subnets`, hostgroup→IP mappings.
- Egress allowlist (§13) for the upstream network firewall.

---

*End of specification.*
