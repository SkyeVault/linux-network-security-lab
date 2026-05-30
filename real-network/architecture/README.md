# Architecture

## Physical Layer

Single Proxmox node. Two physical NICs (or two IPs on one NIC via your upstream provider's routing):

| Interface | Carries | Connected to |
|---|---|---|
| `ens18` (or equiv) | WAN A public traffic | Upstream router |
| `ens19` (or equiv) | WAN B public traffic | Upstream router |
| `ens20` (or equiv) | Proxmox management | Out-of-band or same upstream |

All internal segmentation is done in software via Linux bridges and 802.1Q VLAN tagging on Proxmox. No physical switches required for internal segmentation.

## Bridges

| Bridge | Purpose | Tagged VLANs |
|---|---|---|
| `vmbr0` | WAN A uplink — firewall WAN port only | None (untagged WAN A IP) |
| `vmbr1` | WAN B uplink — firewall WAN port only | None (untagged WAN B IP) |
| `vmbr2` | Internal trunk — all VM traffic | 100-399 |

## VLAN Plan

### WAN A segment (100-199)

| VLAN | Subnet (IPv4) | IPv6 | Purpose |
|---|---|---|---|
| 100 | 10.100.0.0/24 | $WAN_A_IPV6_PREFIX:100::/112 | WAN A LAN — default segment |
| 110 | 10.100.10.0/24 | $WAN_A_IPV6_PREFIX:110::/112 | WAN A DMZ — internet-exposed services |
| 120 | 10.100.20.0/24 | $WAN_A_IPV6_PREFIX:120::/112 | WAN A isolated — high-risk VMs, no egress |

### WAN B segment (200-299)

| VLAN | Subnet (IPv4) | IPv6 | Purpose |
|---|---|---|---|
| 200 | 10.200.0.0/24 | $WAN_B_IPV6_PREFIX:200::/112 | WAN B LAN — default segment |
| 210 | 10.200.10.0/24 | $WAN_B_IPV6_PREFIX:210::/112 | WAN B DMZ — internet-exposed services |
| 220 | 10.200.20.0/24 | $WAN_B_IPV6_PREFIX:220::/112 | WAN B isolated — high-risk VMs |

### Shared / Management (300-399)

| VLAN | Subnet | Purpose |
|---|---|---|
| 300 | 10.10.10.0/24 | Proxmox management — PVE UI, SSH to host |
| 310 | 10.10.20.0/24 | Out-of-band monitoring — Wazuh, Zeek, SIEM reach both segments |
| 320 | 10.10.30.0/24 | Logging aggregation — syslog, metrics, exporters |

## IPv6 Addressing

Each WAN provides a /64 prefix. Hosts use SLAAC or static assignment within that prefix. The /64 gives 2^64 addresses — enough to assign a unique IPv6 to every VM without NAT.

Internal VLANs that need IPv6 reachability get a /112 slice carved from the parent /64 (chosen VLAN ID encoded in the address for readability). Traffic still exits via the respective WAN A or WAN B firewall.

## Inter-segment routing

WAN A and WAN B segments are isolated by default — no direct routing between VLAN 100-199 and VLAN 200-299. The management segment (300-399) can reach both, but only via explicit firewall rules on each OPNsense instance.

This means compromising a VM on one WAN segment does not give lateral access to the other.
