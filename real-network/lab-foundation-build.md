# Devlog - Lab Foundation Build

**Date:** 2026-06-11
**Phase:** Initial infrastructure stand-up
**Status:** Host + access layer complete; internal networking in progress

---

## Summary

Stood up the dedicated server that will host the Linux network security lab and
built it from bare metal up through a hardened, privately-accessible Proxmox
host. The guiding principle for the day was **harden and isolate before
deploying anything** - no VMs were created until the host, access path, and
firewall posture were in place.

By end of session the box is a stable, reboot-persistent Proxmox hypervisor
reachable only over an encrypted tunnel, with public-facing management access
still open as a fallback (to be closed once internal networking is verified).

---

## Environment

- **Host OS:** Debian 13 (Trixie), minimal base image
- **Hypervisor:** Proxmox VE 9.x (installed on top of the Debian base)
- **Boot/storage:** Two NVMe drives in software RAID-1 (mdraid mirror across
  EFI, `/boot`, swap, and root)
- **Access:** WireGuard tunnel from workstation; key-only SSH
- **Firewall:** `ufw` on the host

All public addresses, hostnames, domains, and keys are intentionally omitted
from this log.

---

## Work completed

### 1. Base OS and identity

- Deployed the server with a clean Debian 13 image and confirmed first login
  over SSH using a key (no password auth from the start).
- Verified hardware on arrival: RAID-1 mirror healthy across all arrays
  (`/proc/mdstat` showed all members up; root array completed its initial
  resync in the background), full RAM present, expected core/thread count
  available.
- Set a proper hostname and fixed `/etc/hosts` so the FQDN and short name both
  resolve to the primary interface - a prerequisite for a clean Proxmox install.
  Confirmed with `hostname --ip-address` returning the public interface (not
  loopback) and `hostname --fqdn` resolving without error.

**Note:** The default image mapped the hostname correctly and did *not* include
a `127.0.1.1` line, which is the usual Proxmox-install footgun. Nothing to fix
there.

### 2. SSH hardening

Applied hardening via a drop-in file (`/etc/ssh/sshd_config.d/99-hardening.conf`)
rather than editing the main config, so it survives package updates:

- `PasswordAuthentication no`
- `PermitRootLogin prohibit-password` (key-only root)
- `PubkeyAuthentication yes`
- Disabled challenge-response / keyboard-interactive auth

Reloaded the SSH service and **verified key auth from a second session before
trusting the change** - the cardinal rule of remote hardening: never close your
only working session until a fresh one is proven. Confirmed effective config
with `sshd -T`.

### 3. Host firewall

Installed and configured `ufw` with a deny-by-default incoming posture:

- Default: deny incoming, allow outgoing
- Allowed: SSH, Proxmox web UI port
- Enabled and confirmed the rule set held (the live SSH session surviving
  `ufw enable` is itself the proof the SSH rule is correct)

### 4. Proxmox VE install

Installed Proxmox on the Debian base via the official no-subscription
repository:

- Added the PVE repo for the matching Debian release
- Added the repository signing key (apt correctly rejected the unsigned repo
  until the key was in place - working as designed)
- Installed `proxmox-ve`, plus `postfix` (configured **local-only** for system
  mail) and `open-iscsi`
- Rebooted into the Proxmox kernel and confirmed the running kernel string
  includes the PVE identifier
- Reached the web UI over HTTPS, set the root password for UI login, dismissed
  the expected self-signed-cert warning and the no-subscription notice

Result: Proxmox dashboard live, node healthy, all resources visible and idle.

### 5. Private access via WireGuard

Chose WireGuard over exposing the management plane publicly. The goal is that
the Proxmox UI and host SSH eventually face *only* the tunnel, with nothing but
the WireGuard UDP port reachable from the internet.

- Installed WireGuard on the host; generated the server keypair and a
  per-device preshared key (defense in depth on top of the public-key handshake)
- Brought up the server interface (`wg0`) on a private tunnel subnet, with
  `PostUp`/`PostDown` rules for masquerade + forwarding, and enabled it as a
  boot service so it auto-starts on every reboot
- Enabled IP forwarding (`net.ipv4.ip_forward=1`) persistently
- Opened only the WireGuard UDP port in the firewall
- Generated a **fresh, dedicated keypair on the workstation** for the lab tunnel
  (rather than reusing an existing identity) - one key per purpose
- Configured the client as a **split tunnel**: only the lab subnets route
  through WireGuard, normal internet traffic stays direct

**Verified end to end:** ping across the tunnel (0% loss), and SSH to the host
over its tunnel address succeeded. The host key fingerprint presented on first
connect matched the provider's out-of-band record - a nice confirmation the
endpoint is the real box.

### 6. Networking design (in progress)

Reviewed the host's `/etc/network/interfaces`. The image left the public IP
directly on the physical NIC with no auto-created bridge - a clean slate to add
internal bridges alongside, with zero risk to the public interface.

Planned two internal bridges (config staged, not yet applied):

- **`vmbr1` - NAT bridge (`10.50.10.0/24`):** internet-connected lab VMs. The
  host acts as gateway and masquerades VM traffic out through the public NIC.
  VMs can reach out for updates/packages; the outside cannot reach in unless a
  port is deliberately forwarded.
- **`vmbr2` - isolated bridge (`10.50.20.0/24`):** the pentest range.
  **No NAT, no internet gateway, by design.** Machines here can talk to each
  other and to the workstation over the tunnel, but have *no route out*. This is
  the air-gapped target range - deliberately vulnerable systems can run here
  knowing nothing can phone home, be reached externally, or become a launchpad.

The asymmetry is the whole point: full access *in* over WireGuard (the tunnel
subnet's allowed-IPs cover both bridges), but the isolated range has no access
*out*. Reaching vulnerable targets requires going through the tunnel; the
targets themselves are sealed.

---

## Problems hit & resolved

- **Repo signature rejection on Proxmox install.** The repo line was added
  before its signing key; apt refused the unsigned repository. Correct behavior
  - resolved by installing the release key, then the install proceeded.
- **WireGuard service failed to start initially.** Caused by an unfilled
  interface-name placeholder in the masquerade rule and a key pasted into the
  wrong field. Fixed both and the interface came up cleanly.
- **Subnet collision with an existing tunnel.** Discovered an unrelated,
  already-active WireGuard tunnel on the workstation using the same `/24` I'd
  originally planned for the NAT bridge. Renumbered the lab into a different
  private range (`10.50.x`) so the two tunnels coexist without routing
  ambiguity. Good reminder to enumerate existing routes *before* picking lab
  subnets.
- **Cross-wired peer identity.** The host's initial WireGuard peer entry was
  populated with the workstation's *existing* tunnel identity. Replaced it with
  the freshly generated lab key so the lab tunnel has a clean, separate identity.

---

## Lessons / notes to self

- Doing the unglamorous parts first (host hardening, firewall, access path,
  network design) surfaced more real decisions than the "fun" tooling would
  have. Most of the day's actual thinking was in routing, isolation boundaries,
  and key hygiene - not in any single command.
- The two-session rule for SSH hardening and the layered fallbacks (tunnel +
  public SSH + provider console) meant no step was ever a one-way door.
- Treat any secret that touches a terminal log, screenshot, or notes as burned.
  Keys generated during a guided setup should be **rotated** once the lab is
  stable, with the real material living only on the machines that need it. This
  is queued as a cleanup task.
- The provider's network uses a MAC-locked uplink, so VMs cannot bridge directly
  onto the public interface - NAT bridging is the correct pattern, which also
  happens to give the isolation posture the lab wants.

---

## Next session

1. Apply the staged bridge config (`vmbr1` NAT, `vmbr2` isolated); validate with
   a dry run before committing, then `ifreload`.
2. Set the firewall forward policy so NAT traffic on `vmbr1` passes, while the
   isolated bridge stays sealed (it has no NAT rule, so the policy change does
   not give it a route out).
3. Re-verify tunnel access after the network changes.
4. **Clamp the public surface:** restrict the Proxmox UI and SSH to the
   WireGuard interface, leaving only the WireGuard UDP port internet-facing.
5. Spin up the first VM and a storage/template baseline.
6. **Key rotation** cleanup pass.

---

*Foundation laid. No VMs yet - and that was the point.*
