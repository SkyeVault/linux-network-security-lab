# WAN A — Gateway Setup

Firewall VM: `fw-wan-a`  
Role: `$WAN_A_ROLE` (set in `.env`)

## OPNsense interfaces

| Interface | Assignment | IP |
|---|---|---|
| WAN | `vtnet0` → `vmbr0` | $WAN_A_IPV4 (static) |
| LAN | `vtnet1` → `vmbr2` tagged | trunk carrying VLANs 100-199, 310 |

## IPv6 WAN configuration

- Type: Static IPv6
- Address: First usable in $WAN_A_IPV6_PREFIX (e.g. prefix::1/64)
- Gateway: $WAN_A_IPV6_GATEWAY
- DHCPv6: Disabled (we manage assignments manually)
- RA: Enabled on LAN-facing VLAN interfaces for SLAAC

## Interface VLANs (OPNsense)

Create VLAN sub-interfaces on `vtnet1` for each VLAN in the 100-199 range:

```
vtnet1.100   VLAN 100   10.100.0.1/24    LAN default
vtnet1.110   VLAN 110   10.100.10.1/24   DMZ
vtnet1.120   VLAN 120   10.100.20.1/24   Isolated (no egress)
vtnet1.310   VLAN 310   10.10.20.2/24    OOB monitoring leg
```

## Key firewall rules

```
# WAN inbound (explicit allowlist — everything else is blocked)
pass  in  WAN  proto tcp  to $WAN_A_IPV4  port { <service ports> }   # add as needed
pass  in  WAN6 proto tcp  to $WAN_A_IPV6_PREFIX  port { <...> }

# VLAN 120 — isolated, no internet egress
block out WAN from 10.100.20.0/24

# Monitoring leg
pass  in  VLAN310  from 10.10.20.0/24  to 10.10.20.2  port { 22, 514, 443 }
```

## Logging

Syslog target: `10.10.20.10` (log aggregator on VLAN 310), UDP 514.  
Categories: firewall, authentication, DHCP.
