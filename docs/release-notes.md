# Release Notes

### Known Issues

An issue was discovered with deployments with multiple heavy nodes that prevents them from updating to version 3.1.0. We plan to address this in an upcoming hotfix:

<https://github.com/Security-Onion-Solutions/securityonion/discussions/15920>

For all other known issues, please see <https://github.com/Security-Onion-Solutions/securityonion/issues>.

### Release History

3.1.0 [20260521] Changes
------------------------

- FEATURE: Add Postgres support for future features
- FEATURE: Add bonded NIC support for management interfaces <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15548">#15548</a>
- FEATURE: Add ingest latency metric
- FEATURE: Allow the setup of bond1 for management for ISO installs <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15865">#15865</a>
- FEATURE: Elastic Fleet continuously validate output policy
- FEATURE: RAID monitoring for hypervisor VMs <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15809">#15809</a>
- FEATURE: Restore Suricata Overrides from backup <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15881">#15881</a>
- FEATURE: Sigma mappings - M365 & Fortigate <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15882">#15882</a>
- FEATURE: Simplified Onion AI setup for regions outside US <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15773">#15773</a>
- FEATURE: Support Azure OpenAI endpoints <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15841">#15841</a>
- FIX: 'Investigate' using inaccessible local model shows "insufficient credits"
- FIX: Add options selection to annotations <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15744">#15744</a>
- FIX: Appliance images in SOC grid misaligned <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15713">#15713</a>
- FIX: Consider setting Elastic Agent output level to warning only <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15431">#15431</a>
- FIX: Deterministically sort threshold.conf <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15815">#15815</a>
- FIX: Improve elastic agent install outcome to check that the installation is healthy
- FIX: Improve lucene and elastic query param validation <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15860">#15860</a>
- FIX: Improve reverse DNS lookups success rate <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15760">#15760</a>
- FIX: Improve usability for visually impaired users
- FIX: JA4+ license hyperlink <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15717">#15717</a>
- FIX: Make SOC and Kratos enabled annoations readonly <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15827">#15827</a>
- FIX: Modifying detection templates in config causes SOC to crash loop <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15798">#15798</a>
- FIX: Need better user feedback when attaching assistant chat to a case <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15689">#15689</a>
- FIX: Node descriptions containing both spaces and numbers prevent pillar creation <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15540">#15540</a>
- FIX: Prevent excessive OnionAI query length
- FIX: Reactor sominion_setup <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15834">#15834</a>
- FIX: Refactor Detections backup <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/14992">#14992</a>
- FIX: Reinstall <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15811">#15811</a>
- FIX: SOUP verify all Elasticsearch nodes are compatible with the next Elasticsearch version <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15908">#15908</a>
- FIX: Suricata pcap-log max-files rounds to 0 when calculated value is between 0 and 1 <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15740">#15740</a>
- FIX: UI should show the name of the current Dashboard <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15703">#15703</a>
- FIX: Use hunt action link for case observable hunt pivots <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15752">#15752</a>
- FIX: Use safeload for loading filecheck config <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15859">#15859</a>
- FIX: Zeek ingest pipeline for JA4d.log <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15886">#15886</a>
- UPGRADE: Axios to 1.15.0 in SOC <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15774">#15774</a>
- UPGRADE: CyberChef to 11.0.0 <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15890">#15890</a>
- UPGRADE: Elasticsearch to 9.3.3
- UPGRADE: Kratos and Hydra 26.2.0+pgx <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15796">#15796</a>
- UPGRADE: SOC Go dependencies <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15795">#15795</a>
- UPGRADE: SOC frontend dependency libs <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15848">#15848</a>
- UPGRADE: Suricata to 8.0.5 <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15903">#15903</a>
- UPGRADE: Zeek to 8.0.8 <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15794">#15794</a>
- UPGRADE: nginx to 1.30.1 <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15891">#15891</a>

3.0.0 [20260331] Changes
----------------------

- FEATURE: Configurable Elasticsearch vm.max_map_count setting
- FEATURE: Dynamically load Zeek plugins on zeek startup <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15546">#15546</a>
- FEATURE: Enable JA4+ License Acceptance <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15560">#15560</a>
- FEATURE: Parsing for Zeek websockets logs <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15657">#15657</a>
- FEATURE: Refresh login page with updated look
- FEATURE: Refresh SOC UI with updated look
- FEATURE: Support additional alt names in web cert
- FEATURE: Support docker ulimit customization <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15581">#15581</a>
- FEATURE: Suricata PCAP replacing Stenographer
- FIX: API 401 errors will no longer redirect <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15611">#15611</a>
- FIX: Cleanup file.absent and cron.absent
- FIX: Detections - Intermittent "error closing scroll" <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/14216">#14216</a>
- FIX: Duplicated user roles when refreshing frontend at Administration > Users <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15688">#15688</a>
- FIX: Enabled / Disabled Buttons for SOC Grid Configuration Options <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15649">#15649</a>
- FIX: Fix rule validators in SOC <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15533">#15533</a>
- FIX: Global override configs should not apply to certain indices <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15601">#15601</a>
- FIX: Network Transport for suricata alerts should be lowercase <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15668">#15668</a>
- FIX: Sensors are not checking in while processing long jobs <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15650">#15650</a>
- FIX: so-suricata-testrule script <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15396">#15396</a>
- FIX: STIG V1R3
- FIX: Suricata address-groups vars allow negation <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15664">#15664</a>
- FIX: Unable to create detections via Security Onion API <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15673">#15673</a>
- UPGRADE: All frontend 3rd party deps
- UPGRADE: ATTACK Navigator to 5.3.0 <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15680">#15680</a>
- UPGRADE: CyberChef to 10.22.1 <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15681">#15681</a>
- UPGRADE: ElastAlert2 to 2.28.0 <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15685">#15685</a>
- UPGRADE: Golang 3rd party deps <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15647">#15647</a>
- UPGRADE: Golang to 1.26.1 <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15580">#15580</a>
- UPGRADE: Hydra to 25.4.0 <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15678">#15678</a>
- UPGRADE: Kafka to 3.9.2 <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15684">#15684</a>
- UPGRADE: Kratos to 25.4.0 <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15677">#15677</a>
- UPGRADE: Nginx to 1.29.6 <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15686">#15686</a>
- UPGRADE: OpenCanary to 0.9.7 <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15679">#15679</a>
- UPGRADE: Redis to 7.2.13 <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15682">#15682</a>
- UPGRADE: Suricata to 8.0.4 <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15625">#15625</a>
- UPGRADE: Telegraf to 1.38.0 <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15683">#15683</a>
- UPGRADE: Update Docker base images
