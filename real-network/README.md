# Real Network Lab

A production-grade security research environment running on a single Proxmox node with two independent public IPv4 addresses, each with a dedicated /64 IPv6 subnet. This is not a simulated home lab — all traffic is live and internet-routable.

## Key differences from the virtual lab

| | [Virtual Lab](../labs/) | Real Network Lab |
|---|---|---|
| IPs | RFC 1918 private | Public IPv4 + native IPv6 |
| Scale | Limited by local hardware | Unlimited Proxmox VMs |
| Internet exposure | Air-gapped | Live — honeypots, sensors, services are reachable |
| IPv6 | Simulated | Native /64 per WAN |
| Change risk | Low | High — changes affect live internet-facing infrastructure |

## Network Overview

Two independent WAN connections on a single Proxmox host, each with its own firewall VM, VLAN range, and IPv6 subnet. Internal VLANs are tagged on the Proxmox Linux bridge; WAN A and WAN B traffic is isolated by firewall policy and separate bridge assignments.

```
                     Proxmox Host (single node)
                     ─────────────────────────────────────────────
  [Internet]
      │
      ├── WAN A ($WAN_A_IPV4 / $WAN_A_IPV6_PREFIX)
      │       │
      │   ┌───┴────────────────────────────────────────┐
      │   │  OPNsense-A (fw-wan-a)                     │
      │   │  VLANs 100-199                             │
      │   │  Role: $WAN_A_ROLE                         │
      │   └───────────────────────────────────────────-┘
      │
      └── WAN B ($WAN_B_IPV4 / $WAN_B_IPV6_PREFIX)
              │
          ┌───┴────────────────────────────────────────┐
          │  OPNsense-B (fw-wan-b)                     │
          │  VLANs 200-299                             │
          │  Role: $WAN_B_ROLE                         │
          └────────────────────────────────────────────┘

  Shared management / monitoring (VLANs 300-399) bridges both segments
  via internal routing only — no direct internet path.
```

See [`architecture/`](architecture/) for full diagrams and VLAN assignments.

## Secrets

All IPs, credentials, and host-specific values live in a local `.env` file (gitignored). See [`.env.example`](.env.example) for the full variable list. Scripts and config templates reference `$VAR` names — never hardcoded values.

## Structure

```
real-network/
├── .env.example        variables template (IPs never committed)
├── architecture/       network diagrams, VLAN plan, IPv6 addressing
├── gateway/
│   ├── wan-a/          OPNsense/firewall config for WAN A
│   └── wan-b/          OPNsense/firewall config for WAN B
├── vms/                VM inventory and provisioning notes
├── services/           deployed services (honeypot, SIEM, monitoring)
└── runbooks/           change management, incident response, new-VM checklist
```

## Sections

- [Architecture](architecture/README.md)
- [Gateway / Firewall](gateway/README.md)
- [VM Inventory](vms/README.md)
- [Services](services/README.md)
- [Runbooks](runbooks/README.md)
