# Services

Deployed services on the real network. Each subdirectory documents the setup, configuration, and findings for a specific service category.

## Service placement

Services that are internet-exposed must live in a DMZ VLAN (110 or 210). They get no outbound access beyond what they explicitly need. Services that only need inbound from the monitoring segment live in VLAN 310.

| Service | VLAN | Internet-exposed | Subdirectory |
|---|---|---|---|
| OpenCanary honeypot | 110 or 210 (DMZ) | Yes — that's the point | [honeypot/](honeypot/) |
| Wazuh SIEM | 310 (monitoring) | No | [siem/](siem/) |
| Zeek / Suricata | 310 (monitoring) | No | [monitoring/](monitoring/) |

## Adding a new service

1. Decide: which WAN segment does it belong to? Which VLAN?
2. Update the [VM inventory](../vms/README.md)
3. Write the firewall rules following the pattern in [gateway/](../gateway/)
4. Document setup in the relevant subdirectory here
5. Run through the [new-VM runbook](../runbooks/new-vm.md)
