# Lab Architecture

A single-node Proxmox virtualization host built for Linux and network-security
practice. The design goal is simple to state and drove every decision:

> **Full access *in* for the operator over an encrypted tunnel; no access *out*
> for the things that shouldn't have it; nothing exposed to the public internet
> except the tunnel itself.**

This document describes what is actually deployed. All public addresses,
hostnames, provider details, and keys are intentionally omitted.

---

## Overview

```
                         Public Internet
                                │
                       (only UDP/51820 open)
                                │
                        ┌───────▼────────┐
                        │   enp5s0       │  public NIC, host IP
                        │  (Proxmox host)│
                        └───┬────────┬───┘
                            │        │
              WireGuard ────┘        └──── NAT masquerade
              wg0  10.99.99.1/24          (for vmbr1 only)
                   │
        ┌──────────┴───────────┐
        │ operator reaches      │
        │ everything via tunnel │
        └──────────┬───────────┘
                   │
        ┌──────────┼──────────────────────┐
        │          │                       │
   ┌────▼────┐ ┌───▼─────┐           ┌─────▼──────┐
   │ host    │ │ vmbr1   │           │  vmbr2     │
   │ SSH/UI  │ │ NAT     │           │  ISOLATED  │
   │ (wg only)│ │10.50.10 │           │  10.50.20  │
   └─────────┘ └───┬─────┘           └─────┬──────┘
                   │ VMs get internet       │ VMs have NO route out
                ┌──▼──┐                  ┌──▼──┐
                │ VMs │                  │ VMs │  (vulnerable targets)
                └─────┘                  └─────┘
```

---

## Physical / host layer

- **Single Proxmox VE node** on dedicated hardware.
- **One physical NIC** (`enp5s0`) carrying the host's public IP directly
  (static config). The provider uses a MAC-locked uplink, so VMs **cannot**
  bridge straight onto the public interface — this is why all VM connectivity
  is done via internal bridges with NAT rather than a public bridge.
- **Storage:** two NVMe drives in software RAID-1 (mirror) across EFI, `/boot`,
  swap, and root — single-disk failure does not take the host down.
- **Hypervisor:** Proxmox VE on a Debian base.

There is intentionally **no `vmbr0`**: the installer left the public IP on the
physical NIC, and rather than reconfigure that (and risk public connectivity),
the lab bridges were added alongside it as internal-only interfaces.

---

## Network bridges

Two software bridges, both **portless** (not attached to the physical NIC).
No VLAN tagging is used — segmentation here is by separate bridges and routing
policy, which is sufficient and simpler for a single-node lab.

| Bridge  | Subnet          | Gateway      | Internet egress | Purpose                                   |
|---------|-----------------|--------------|-----------------|-------------------------------------------|
| `vmbr1` | `10.50.10.0/24` | `10.50.10.1` | **Yes** (NAT)   | General lab VMs that need updates/packages |
| `vmbr2` | `10.50.20.0/24` | `10.50.20.1` | **No** (by design) | Isolated pentest range — vulnerable targets |

### `vmbr1` — NAT bridge

The host acts as gateway (`10.50.10.1`) and masquerades VM traffic out through
the public NIC:

```
iptables -t nat -A POSTROUTING -s 10.50.10.0/24 -o enp5s0 -j MASQUERADE
iptables -A FORWARD -i vmbr1 -o enp5s0 -j ACCEPT
iptables -A FORWARD -i enp5s0 -o vmbr1 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

VMs on this bridge can reach **out** to the internet (for `apt`, image pulls,
tooling). The outside cannot reach **in** unless a port is deliberately
forwarded. This is the segment for anything that needs to be online.

### `vmbr2` — isolated bridge

Deliberately has **no NAT rule and no internet gateway.** VMs here can talk to
each other and to the operator (over WireGuard), but have **no route off the
host.** This is the air-gapped attack range: deliberately vulnerable machines
run here knowing they cannot phone home, cannot be reached from the internet,
and cannot become a pivot point to anything outside the box.

The isolation is structural, not rule-based — there is simply no masquerade
entry for `10.50.20.0/24`, so even if firewall policy were relaxed, this segment
still has nowhere to route.

IP forwarding is enabled host-wide (`net.ipv4.ip_forward=1`) for `vmbr1` NAT and
WireGuard routing; the isolated bridge's lack of egress comes from the absence
of a NAT path, not from forwarding being off.

---

## Access: WireGuard tunnel

All operator access goes through a WireGuard tunnel. The management plane is
**not** exposed to the public internet.

- **Server interface:** `wg0`, `10.99.99.1/24`, listening on a single UDP port
  (the only port open to the public internet).
- **Client:** split-tunnel — only the lab subnets route through the tunnel;
  normal internet traffic on the operator's workstation stays direct.
- **Client `AllowedIPs`:** `10.99.99.0/24, 10.50.10.0/24, 10.50.20.0/24` — so
  the operator can reach the tunnel network **and both lab bridges**, including
  the isolated one.
- **Auth:** public-key handshake plus a per-device preshared key (defense in
  depth). Keys are per-device; the lab tunnel uses a dedicated identity, not a
  reused one.
- **Persistence:** the server-side tunnel is enabled as a boot service and
  comes up automatically on every reboot.

### The access asymmetry (the core idea)

```
operator ──(WireGuard)──► host, vmbr1 VMs, vmbr2 VMs     ✅ reachable
vmbr2 VMs ──────────────► internet                        ❌ no route
internet  ──────────────► host SSH / Proxmox UI           ❌ tunnel-only
internet  ──────────────► WireGuard UDP port              ✅ (reveals nothing
                                                              without a key)
```

The operator reaches *everything* through the tunnel — including the isolated
targets — while the isolated targets can reach *nothing*. Vulnerable machines
are accessible for testing but sealed from the outside world.

---

## Host hardening & public surface

- **SSH:** key-only. Password authentication disabled; root permitted by key
  only. Applied via a drop-in config so it survives package updates.
- **Firewall (`ufw`):** default-deny incoming. Forward policy set to `ACCEPT`
  so `vmbr1` NAT traffic passes (the isolated bridge stays sealed regardless,
  having no NAT path).
- **Public surface — fully clamped:**
  - SSH (22) and the Proxmox UI (8006) are allowed **only on the `wg0`
    interface**.
  - The plain public rules for 22 and 8006 are removed.
  - The **only** thing the public internet can reach is the WireGuard UDP port.
- **Management URLs** are reached at the tunnel address (e.g. the Proxmox UI at
  `https://10.99.99.1:8006`), never the public IP.

### Verifying the clamp

With the tunnel **down**, an attempt to SSH to the host's public IP times out /
is refused. With the tunnel **up**, SSH to the tunnel address connects. That
asymmetry is the proof the management plane is dark to the internet.

Recovery paths if the tunnel ever breaks: the provider's out-of-band KVM/rescue
console remains available independent of any firewall or network state, so the
clamp is not a one-way door.

---

## Design rationale (for the curious / for review)

**Why bare metal, not a managed VPS?** Running your own hypervisor and carving
out genuinely isolated networks requires owning the machine, not renting a slice
of someone else's virtualized host. Nested virtualization on a VPS is typically
unsupported, and a managed environment doesn't give the root-level control a
security lab needs.

**Why NAT bridges instead of a public bridge?** The provider MAC-locks the
uplink; VMs with unrecognized MACs on a public bridge get dropped and can
trigger abuse handling. NAT bridging is the correct pattern for this network —
and it happens to give exactly the isolation posture the lab wants.

**Why a tunnel instead of just firewalling the admin ports?** An exposed admin
port — even a well-secured one — is a standing liability and a target for
constant background scanning. A tunnel removes the management plane from the
public internet entirely; an attacker can't attack what they can't see, and the
one open port (WireGuard) gives nothing away without a valid key.

**Why separate the isolated range structurally rather than by firewall rule?**
Rules can be misconfigured, disabled, or bypassed. The isolated bridge has no
NAT path at all, so its lack of egress is a property of the routing topology,
not of a rule that has to stay correct. Defense that doesn't depend on a rule
staying right is more durable.

---

## Current state & next steps

**Built:** RAID-mirrored Proxmox host, hardened SSH, host firewall, NAT bridge,
isolated bridge, WireGuard access, full public-surface clamp.

**Next:**
- Provision base VM templates and an ISO/template store.
- Stand up first VMs on the appropriate bridges (online tooling on `vmbr1`,
  targets on `vmbr2`).
- Add per-VM Proxmox firewall rules for finer control within bridges.
- **Key rotation:** rotate any keys generated during interactive setup; keep
  real key material only on the machines that need it.

---

## Conventions

- `10.99.99.0/24` — WireGuard tunnel network
- `10.50.10.0/24` — NAT lab segment (internet-connected)
- `10.50.20.0/24` — isolated pentest segment (no egress)
- Public IPs, provider, domain, and keys are omitted from all committed docs.
