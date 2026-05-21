# PostgreSQL

From <https://www.postgresql.org/>:

> PostgreSQL is a powerful, open source object-relational database system with over 35 years of active development that has earned it a strong reputation for reliability, feature robustness, and performance.

Security Onion uses PostgreSQL as its central data platform, running on manager nodes. It backs the [Onion AI](onion-ai.md) conversation store and receives [Telegraf](influxdb.md) metrics alongside [InfluxDB](influxdb.md).

## Storage

- Data: `/nsm/postgres/`
- Configuration: `/opt/so/conf/postgres/`
- Secrets (mounted 0600 into the container): `/opt/so/conf/postgres/secrets/`
- Logs: `/opt/so/log/postgres/`

!!! warning

    Do not manually modify files in `/nsm/postgres/` as this could corrupt the database.

## Authentication

PostgreSQL uses a three-tier authentication model:

- **`postgres` superuser** — administrative operations only (schema init, role management during upgrade). The password is auto-generated and delivered to the container via the file mounted at `/run/secrets/postgres_password` using `POSTGRES_PASSWORD_FILE`.
- **`so_postgres` application user** — used by the Security Onion platform (the SOC assistant store) for day-to-day database access. The password is auto-generated and delivered via `/run/secrets/so_postgres_pass` using `SO_POSTGRES_PASS_FILE`.
- **`so_telegraf_<minion>` per-minion users** — one login role per grid minion, all members of the shared `so_telegraf` group. Each minion only sees its own credential via its per-minion pillar file.

All credentials are auto-generated and managed by Salt. Passwords are delivered to the container as mounted files rather than plaintext environment variables so they do not appear in `docker inspect` output.

## Encryption

All TCP connections to PostgreSQL are encrypted with TLS using a certificate signed by the Security Onion CA at `/etc/pki/postgres.crt` / `/etc/pki/postgres.key`.

`pg_hba.conf` only permits:

- `local` Unix-socket connections (used by the container's own admin scripts).
- `hostssl` TLS-wrapped TCP connections with SCRAM-SHA-256 authentication.

Plain-text `host` TCP connections are rejected, so a client using `sslmode=disable` cannot establish a session even if it has valid credentials.

## Configuration

You can configure PostgreSQL by going to [Administration](administration.md) --> Configuration --> postgres.

Commonly adjusted settings:

| Setting | Default | Description |
|---------|---------|-------------|
| `max_connections` | `100` | Maximum concurrent PostgreSQL connections. |
| `shared_buffers` | `256MB` | Memory for the shared buffer cache. Raising this improves read cache hit rate at the cost of RAM. |
| `log_min_messages` | `warning` | Minimum severity written to the PostgreSQL log. |
| `telegraf.retention_days` | `14` | Days of Telegraf metrics retained in the `so_telegraf` database. Older partitions are dropped hourly by pg_partman. |

Infrastructure settings (TLS paths, `hba_file`, `listen_addresses`, preloaded extensions, etc.) are marked **advanced** — they are Salt-managed and should not be changed in normal operation.

### Telegraf output selector

The backend(s) Telegraf writes metrics to is configured under [Administration](administration.md) --> Configuration --> telegraf --> **output**:

| Value | Behavior |
|-------|----------|
| `INFLUXDB` | Telegraf writes only to [InfluxDB](influxdb.md). |
| `POSTGRES` | Telegraf writes only to PostgreSQL (`so_telegraf` database). |
| `BOTH` | Telegraf dual-writes to both (useful during migration validation). |

## Firewall

PostgreSQL is only reachable from the manager itself and — within the docker bridge — from container peers. Access is enforced by the Security Onion [firewall](firewall.md) via the `DOCKER-USER` iptables chain: `postgres` (5432) is granted only to the `manager`, `managerhype`, `managersearch`, `eval`, and `standalone` hostgroups. Sensors, heavynodes, search nodes, receivers, and other roles cannot reach the port.

## Management Commands

Security Onion provides command-line tools for managing PostgreSQL. These tools run `psql` inside the [Docker](docker.md) container so PostgreSQL client binaries do not need to be installed on the host.

### Interactive Shell

Open an interactive `psql` session as the superuser:

```bash
sudo so-postgres-manage shell
```

### Execute SQL

Run a SQL command directly:

```bash
sudo so-postgres-manage sql "SELECT version();"
```

### Execute SQL File

Run a SQL script:

```bash
sudo so-postgres-manage sqlfile /path/to/script.sql
```

### List Databases

```bash
sudo so-postgres-manage dblist
```

### List Database Roles

```bash
sudo so-postgres-manage userlist
```

### Container Lifecycle

```bash
sudo so-postgres-start
sudo so-postgres-stop
sudo so-postgres-restart
```

### Grid Metrics Snapshot

`so-stats-show` prints the most recent CPU, memory, disk, and load metrics for each host reporting to the Telegraf PostgreSQL backend. Useful for verifying that Telegraf is successfully landing metrics before consuming them in a dashboard.

```bash
sudo so-stats-show            # all hosts
sudo so-stats-show <hostname> # a single host
```

This tool requires `telegraf.output` to be `POSTGRES` or `BOTH`.

## Telegraf Metrics

When `telegraf.output` is `POSTGRES` or `BOTH`, each grid minion writes its metrics directly to a shared `so_telegraf` database inside PostgreSQL using its own per-minion login role.

- Tables live in the `telegraf` schema and are partitioned daily by [`pg_partman`](https://github.com/pgpartman/pg_partman).
- Retention is driven by the `postgres.telegraf.retention_days` configuration value; `pg_partman` drops expired partitions hourly via [`pg_cron`](https://github.com/citusdata/pg_cron).
- Tags and fields are stored as `jsonb` so the fixed-column-per-table limit (1600) does not constrain high-cardinality inputs.
- Each TCP connection is TLS-verified against the Security Onion CA (`sslmode=verify-full`).

Per-minion credentials are provisioned automatically when a minion's key is accepted — a Salt reactor creates the database role and writes the password into that minion's own pillar file. No admin intervention is required.

## Extensions

The following PostgreSQL extensions are installed and available:

| Extension | Description |
|-----------|-------------|
| **pgvector** | Vector similarity search for AI/ML embeddings. |
| **pg_cron** | Job scheduling within PostgreSQL. Drives pg_partman maintenance. |
| **pg_partman** | Time-based automated table partitioning. Drives Telegraf metric retention. |
| **pgcrypto** | Cryptographic functions for column-level encryption. |

## Backup

PostgreSQL is automatically backed up daily at 00:05. The backup uses `pg_dumpall` to capture all databases and roles, compressed with gzip, and stored 0600-mode at `/nsm/backup/so-postgres-backup-YYYY_MM_DD.sql.gz` with 7-day retention.

See the [Backup](backup.md) page for more.

To manually create a backup:

```bash
docker exec so-postgres pg_dumpall -U postgres | gzip > /nsm/backup/so-postgres-manual-backup.sql.gz
```

To restore from a backup:

```bash
zcat /nsm/backup/so-postgres-backup-2026_04_21.sql.gz | docker exec -i so-postgres psql -U postgres
```

## Diagnostic Logging

PostgreSQL server logs land at `/opt/so/log/postgres/`. For container-level issues (startup problems, OOM, crashes), also check the [Docker](docker.md) logs:

```bash
sudo docker logs so-postgres
```

## More Information

!!! note

    For more information about PostgreSQL, please see <https://www.postgresql.org/docs/>.
