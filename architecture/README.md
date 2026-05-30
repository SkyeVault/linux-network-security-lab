# Lab Architecture

## Network Topology

```
                        ┌─────────────────────────────────────────┐
                        │              Proxmox Host                │
                        │                                          │
  ┌──────────────┐      │  ┌──────────┐   ┌──────────┐            │
  │   Internet   │      │  │ Attacker │   │Monitoring│            │
  │  (optional)  │      │  │  VM      │   │   VM     │            │
  └──────┬───────┘      │  │ Kali     │   │  Wazuh   │            │
         │              │  │ VLAN 10  │   │  VLAN 30 │            │
  ┌──────┴───────┐      │  └────┬─────┘   └────┬─────┘            │
  │  OPNsense/   │      │       │               │                  │
  │  pfSense     │◄─────┤  ─────┴───────────────┘                 │
  │  Firewall    │      │       │  VLAN trunk                      │
  └──────────────┘      │  ┌────┴─────────────────┐               │
                        │  │      Victim VLAN 20   │               │
                        │  │  DVWA  Metro  VulnHub │               │
                        │  └──────────────────────┘               │
                        └─────────────────────────────────────────┘
```

## VLAN Assignments

| VLAN | Subnet | Purpose |
|---|---|---|
| 10 | 10.0.10.0/24 | Attacker (Kali Linux) |
| 20 | 10.0.20.0/24 | Victims (DVWA, Metasploitable, VulnHub) |
| 30 | 10.0.30.0/24 | Monitoring (Wazuh, Zeek/Suricata) |

## Firewall Rules

- VLAN 10 → VLAN 20: All traffic allowed (lab attacks)
- VLAN 10 → VLAN 30: Blocked (attacker cannot touch monitoring)
- VLAN 20 → VLAN 30: Mirrored/span port for traffic capture
- All VLANs → Internet: Blocked by default (air-gapped lab)

## VM Inventory

| VM | OS | VLAN | Role |
|---|---|---|---|
| kali | Kali Linux | 10 | Attacker workstation |
| dvwa | Ubuntu + DVWA | 20 | Web app target |
| metasploitable | Metasploitable 2 | 20 | Network services target |
| wazuh | Ubuntu + Wazuh | 30 | SIEM |
| zeek | Ubuntu + Zeek | 30 | Network monitoring |
| opencanary | Ubuntu + OpenCanary | 20 | Honeypot |
