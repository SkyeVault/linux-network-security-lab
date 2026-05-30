# SIEM — Wazuh (Shared Monitoring Segment)

Wazuh runs on VLAN 310 and has agent connectivity to VMs across both WAN A and WAN B segments. It never has a public IP — agents connect inbound to VLAN 310 through firewall rules on each OPNsense instance.

## Architecture

```
fw-wan-a (VLAN 310 leg: 10.10.20.2) ──┐
fw-wan-b (VLAN 310 leg: 10.10.20.3) ──┤──► Wazuh Manager (10.10.20.10)
VMs on VLAN 100-299 (agent traffic) ───┘         │
                                            Wazuh Dashboard
                                            (HTTPS, VLAN 310 only)
```

## Agent coverage targets

- [ ] fw-wan-a (OPNsense — syslog forwarding)
- [ ] fw-wan-b (OPNsense — syslog forwarding)
- [ ] All honeypot VMs
- [ ] All internet-facing DMZ services

## Alert categories to tune

1. Honeypot hits — correlate with firewall deny logs
2. Auth failures across all agents
3. Unusual outbound from non-DMZ VMs (canary for compromise)
4. Firewall rule changes (configuration integrity)
