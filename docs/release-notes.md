# Release Notes

### Known Issues

For all known issues, please see <https://github.com/Security-Onion-Solutions/securityonion/issues>.

### Release History

3.0.0 [20260331] Changes
----------------------

- FEATURE: Configurable Elasticsearch vm.max_map_count setting
- FEATURE: Dynamically load Zeek plugins on zeek startup `#15546 <https://github.com/Security-Onion-Solutions/securityonion/issues/15546>`_
- FEATURE: Enable JA4+ License Acceptance `#15560 <https://github.com/Security-Onion-Solutions/securityonion/issues/15560>`_
- FEATURE: Parsing for Zeek websockets logs `#15657 <https://github.com/Security-Onion-Solutions/securityonion/issues/15657>`_
- FEATURE: Refresh login page with updated look
- FEATURE: Refresh SOC UI with updated look
- FEATURE: Support additional alt names in web cert
- FEATURE: Support docker ulimit customization `#15581 <https://github.com/Security-Onion-Solutions/securityonion/issues/15581>`_
- FEATURE: Suricata PCAP replacing Stenographer
- FIX: API 401 errors will no longer redirect `#15611 <https://github.com/Security-Onion-Solutions/securityonion/issues/15611>`_
- FIX: Cleanup file.absent and cron.absent
- FIX: Detections - Intermittent "error closing scroll" `#14216 <https://github.com/Security-Onion-Solutions/securityonion/issues/14216>`_
- FIX: Enabled / Disabled Buttons for SOC Grid Configuration Options `#15649 <https://github.com/Security-Onion-Solutions/securityonion/issues/15649>`_
- FIX: Fix rule validators in SOC `#15533 <https://github.com/Security-Onion-Solutions/securityonion/issues/15533>`_
- FIX: Global override configs should not apply to certain indices `#15601 <https://github.com/Security-Onion-Solutions/securityonion/issues/15601>`_
- FIX: Network Transport for suricata alerts should be lowercase `#15668 <https://github.com/Security-Onion-Solutions/securityonion/issues/15668>`_
- FIX: Sensors are not checking in while processing long jobs `#15650 <https://github.com/Security-Onion-Solutions/securityonion/issues/15650>`_
- FIX: so-suricata-testrule script `#15396 <https://github.com/Security-Onion-Solutions/securityonion/issues/15396>`_
- FIX: STIG V1R3
- FIX: Suricata address-groups vars allow negation `#15664 <https://github.com/Security-Onion-Solutions/securityonion/issues/15664>`_
- FIX: Unable to create detections via Connect API `#15673 <https://github.com/Security-Onion-Solutions/securityonion/issues/15673>`_
- UPGRADE: All frontend 3rd party deps
- UPGRADE: ATTACK Navigator to 5.3.0 `#15680 <https://github.com/Security-Onion-Solutions/securityonion/issues/15680>`_
- UPGRADE: CyberChef to 10.22.1 `#15681 <https://github.com/Security-Onion-Solutions/securityonion/issues/15681>`_
- UPGRADE: ElastAlert2 to 2.28.0 `#15685 <https://github.com/Security-Onion-Solutions/securityonion/issues/15685>`_
- UPGRADE: Golang 3rd party deps `#15647 <https://github.com/Security-Onion-Solutions/securityonion/issues/15647>`_
- UPGRADE: Golang to 1.26.1 `#15580 <https://github.com/Security-Onion-Solutions/securityonion/issues/15580>`_
- UPGRADE: Hydra to 25.4.0 `#15678 <https://github.com/Security-Onion-Solutions/securityonion/issues/15678>`_
- UPGRADE: Kafka to 3.9.2 `#15684 <https://github.com/Security-Onion-Solutions/securityonion/issues/15684>`_
- UPGRADE: Kratos to 25.4.0 `#15677 <https://github.com/Security-Onion-Solutions/securityonion/issues/15677>`_
- UPGRADE: Nginx to 1.29.6 `#15686 <https://github.com/Security-Onion-Solutions/securityonion/issues/15686>`_
- UPGRADE: OpenCanary to 0.9.7 `#15679 <https://github.com/Security-Onion-Solutions/securityonion/issues/15679>`_
- UPGRADE: Redis to 7.2.13 `#15682 <https://github.com/Security-Onion-Solutions/securityonion/issues/15682>`_
- UPGRADE: Suricata to 8.0.4 `#15625 <https://github.com/Security-Onion-Solutions/securityonion/issues/15625>`_
- UPGRADE: Telegraf to 1.38.0 `#15683 <https://github.com/Security-Onion-Solutions/securityonion/issues/15683>`_
- UPGRADE: Update Docker base images
