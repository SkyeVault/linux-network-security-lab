# WAN B — Gateway Setup

Firewall VM: `fw-wan-b`  
Role: `$WAN_B_ROLE` (set in `.env`)

## OPNsense interfaces

| Interface | Assignment | IP |
|---|---|---|
| WAN | `vtnet0` → `vmbr1` | $WAN_B_IPV4 (static) |
| LAN | `vtnet1` → `vmbr2` tagged | trunk carrying VLANs 200-299, 310 |

## IPv6 WAN configuration

- Type: Static IPv6
- Address: First usable in $WAN_B_IPV6_PREFIX (e.g. prefix::1/64)
- Gateway: $WAN_B_IPV6_GATEWAY
- DHCPv6: Disabled
- RA: Enabled on LAN-facing VLAN interfaces for SLAAC

## Interface VLANs (OPNsense)

```
vtnet1.200   VLAN 200   10.200.0.1/24    LAN default
vtnet1.210   VLAN 210   10.200.10.1/24   DMZ
vtnet1.220   VLAN 220   10.200.20.1/24   Isolated (no egress)
vtnet1.310   VLAN 310   10.10.20.3/24    OOB monitoring leg
```

## Key firewall rules

```
# WAN inbound (explicit allowlist)
pass  in  WAN  proto tcp  to $WAN_B_IPV4  port { <service ports> }
pass  in  WAN6 proto tcp  to $WAN_B_IPV6_PREFIX  port { <...> }

# VLAN 220 — isolated
block out WAN from 10.200.20.0/24

# Monitoring leg
pass  in  VLAN310  from 10.10.20.0/24  to 10.10.20.3  port { 22, 514, 443 }
```

## Logging

Syslog target: `10.10.20.10` (log aggregator on VLAN 310), UDP 514.
