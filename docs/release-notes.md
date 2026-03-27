# Release Notes

### Known Issues

For all known issues, please see <https://github.com/Security-Onion-Solutions/securityonion/issues>.

### Release History

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
- FIX: Unable to create detections via Connect API <a href="https://github.com/Security-Onion-Solutions/securityonion/issues/15673">#15673</a>
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
