# VM Inventory

Real IPs and hostnames are tracked in a local `inventory.md` file (gitignored). This document defines the naming convention and schema — fill in `inventory.md` from the template below.

## Naming convention

```
<role>-<segment>-<sequence>
```

Examples:
- `fw-wan-a` — firewall for WAN A
- `honey-dmz-a-01` — first honeypot in WAN A DMZ
- `mon-shared-01` — first shared monitoring VM
- `win-dc-a-01` — Windows domain controller in WAN A segment

## Proxmox VMID ranges

| Range | Segment |
|---|---|
| 100–199 | WAN A VMs |
| 200–299 | WAN B VMs |
| 300–399 | Shared / management |
| 900–999 | Templates |

## `inventory.md` template (copy, fill in, keep gitignored)

```markdown
# VM Inventory (PRIVATE — gitignored)

| VMID | Hostname | VLAN | IPv4 | IPv6 | OS | Role | Notes |
|---|---|---|---|---|---|---|---|
| 100 | fw-wan-a | vmbr0/vmbr2 | $WAN_A_IPV4 | $WAN_A_IPV6_PREFIX::1 | OPNsense | Firewall WAN A | |
| 200 | fw-wan-b | vmbr1/vmbr2 | $WAN_B_IPV4 | $WAN_B_IPV6_PREFIX::1 | OPNsense | Firewall WAN B | |
| 300 | mon-wazuh-01 | 310 | 10.10.20.10 | — | Ubuntu 24.04 | Wazuh SIEM | |
| 310 | mon-zeek-01 | 310 | 10.10.20.20 | — | Ubuntu 24.04 | Zeek + Suricata | span port VLAN 310 |
| 320 | mon-log-01 | 320 | 10.10.30.10 | — | Ubuntu 24.04 | Syslog / Loki | |
```

## Snapshot policy

Before any significant change to a production VM:

1. Take a Proxmox snapshot with a descriptive name: `pre-<change>-<date>`
2. Document the change in the relevant runbook
3. Verify functionality after the change
4. Remove snapshots older than 30 days unless they mark a known-good baseline
