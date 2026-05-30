# Internet-Facing Honeypot — OpenCanary

Unlike the virtual lab honeypot, this instance is actually reachable from the internet via its WAN IP. It logs real, unsolicited attack traffic.

## Placement

- VLAN: 110 (WAN A DMZ) or 210 (WAN B DMZ) — document which one in `.env`
- IPv4: assigned from the DMZ subnet, NAT'd to a port on $WAN_A_IPV4 or $WAN_B_IPV4
- IPv6: assigned a static address from the WAN /64 — directly reachable, no NAT

The IPv6 address gets its own firewall allowlist on the OPNsense WAN rule. Port-for-port: whatever OpenCanary listens on is opened on the firewall.

## Why IPv6 matters here

Scanners increasingly do IPv6. A honeypot with a live /64 address (no NAT) sees traffic that a NAT'd IPv4 honeypot misses. Log both stacks separately to compare attacker behavior.

## Setup

See [setup notes](setup.md) once written.

## Data collected

- Source IP (v4 and v6)
- Target port / service
- Credentials attempted (SSH, HTTP auth, FTP)
- Timing and frequency patterns
- Geographic distribution

All logs forward to Wazuh on VLAN 310 via syslog over the monitoring leg.
