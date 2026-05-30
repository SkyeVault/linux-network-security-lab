# Gateway / Firewall

Two OPNsense VMs — one per WAN. Each is the sole path between its public IP and the internal VLANs it owns. Neither has visibility into the other WAN's traffic.

## Firewall VMs

| VM | Hostname | WAN IP | WAN IPv6 | Internal VLANs |
|---|---|---|---|---|
| fw-wan-a | fw-wan-a.lab | $WAN_A_IPV4 | $WAN_A_IPV6_PREFIX | 100-199 |
| fw-wan-b | fw-wan-b.lab | $WAN_B_IPV4 | $WAN_B_IPV6_PREFIX | 200-299 |

Both OPNsense instances also have a leg on VLAN 310 (out-of-band monitoring) so Wazuh can collect firewall logs from both without giving the monitoring VM a public IP.

## OPNsense VM specs (Proxmox)

| Setting | Value |
|---|---|
| vCPUs | 4 |
| RAM | 4 GB |
| Disk | 32 GB |
| WAN NIC | `vmbr0` (fw-wan-a) / `vmbr1` (fw-wan-b) |
| LAN/trunk NIC | `vmbr2` with VLAN tagging |

## Firewall rule philosophy

- Default deny inbound on WAN — explicit allowlist only
- Default permit outbound from LAN VLANs — restricted per VLAN (isolated VLANs have no egress)
- No cross-WAN routing in firewall rules — A and B are independent
- Management VLAN (300) allow: SSH to firewall admin interface from VLAN 300 only
- Logging: all WAN deny rules logged; log shipped to VLAN 310 syslog collector

## Per-WAN config

- [WAN A setup](wan-a/)
- [WAN B setup](wan-b/)
