# Lab Report — Infrastructure as Code: First Declarative Resource

**Date:** 2026-06-12
**Phase:** IaC foundation
**Status:** Complete — end-to-end provisioning pipeline verified

---

## Objective

Move the lab from hand-configured to *declared in code*. Establish an
Infrastructure-as-Code pipeline that can provision containers and VMs on the
Proxmox host reproducibly, version-controlled, with no manual clicking through
the web UI. Prove the pipeline end to end by standing up a real container.

---

## Tooling

| Layer        | Choice                | Rationale                                              |
|--------------|-----------------------|--------------------------------------------------------|
| Provisioning | OpenTofu              | Open-source, Terraform-compatible, no licensing concerns |
| Provider     | `bpg/proxmox`         | Actively maintained (the older Telmate provider is not) |
| Auth         | Proxmox API token     | Scoped credential, never the root web password         |
| Access       | over WireGuard tunnel | API is not public; control plane stays private          |

Config management (Ansible) is deferred to a later phase — this report covers
the provisioning half only.

---

## What was built

A single Debian 13 LXC container, declared entirely in OpenTofu:

- Unprivileged container (root mapped to an unprivileged host UID for isolation)
- 1 core / 512 MB / 8 GB disk on directory storage
- Attached to the NAT bridge (`vmbr1`, `10.50.10.0/24`), static IP
- SSH public key injected at creation
- `nesting` feature enabled (required for the template's systemd version)

The relevant value of this is not the container — it is that the container is
**described in version-controlled code** and can be destroyed and recreated on
demand. The same pattern now scales to every future VM and target.

---

## Access architecture (relevant context)

The Proxmox API is reachable only over the WireGuard tunnel. Therefore the IaC
runs from the operator workstation with the tunnel up, pointing at the tunnel
address. If the tunnel is down, provisioning cannot reach the API — by design.
The control plane is as private as the management plane.

Created resources land on the appropriate bridge: internet-connected workloads
on the NAT segment, isolated targets (future) on the air-gapped segment.

---

## Problems encountered & resolved

This is the part worth keeping. The pipeline did not work first try; each
failure was instructive.

### 1. `file()` not allowed in tfvars

`.tfvars` files hold literal values only — function calls (`file()`,
`pathexpand()`) are not permitted there; they belong in `.tf` files. Moved the
key-reading logic into the provider/resource config and kept tfvars to plain
values.

### 2. `~` not expanded by `file()`

`file("~/.ssh/...")` looks for a literal `~` directory. Wrapped paths in
`pathexpand()` so the home directory resolves correctly.

### 3. Referenced SSH keys did not exist

The config pointed at key filenames that were never created on the workstation
(leftover names from an earlier plan). Surfaced only when `file()` tried to read
them. Lesson: validate that referenced files exist before relying on them.

### 4. `storage 'local-lvm' does not exist`

The default Proxmox tutorial assumes an LVM-thin pool (`local-lvm`). This host
uses software RAID with a directory-backed filesystem, so the only storage is
`local` (directory type). Confirmed with `pvesm status` and pointed the config
at the real storage. Promoted the value to a variable so it is defined once
rather than hardcoded per resource.

### 5. systemd-in-container warning

Debian 13's systemd version needs the LXC `nesting` feature to run cleanly in an
unprivileged container. Added a `features { nesting = true }` block and
re-applied.

### 6. SSH key confusion (the big one)

The container built successfully, but SSH fell back to a password prompt. Root
cause: the workstation held **eight** SSH keys, **two of which shared the same
comment** (`lorelei@...`). Three different keys were involved across the
config — the one the server actually trusted, the one the config *claimed* was
the server key (untrusted), and a third injected into the container. The reused
comment made the keys visually indistinguishable.

**Resolution:**
- Identified the genuinely-trusted key empirically (tested each with
  `IdentitiesOnly=yes` rather than trusting the comment).
- Generated/standardized on a single, clearly-labeled lab key.
- Added that key to the host's authorized_keys (keeping the prior key as a
  fallback until the new one was confirmed).
- Fixed the IaC config to reference the correct key consistently.
- Wrote `~/.ssh/config` with `IdentitiesOnly yes` so SSH offers only the
  intended key instead of cycling through all of them and falling back to a
  password.

The `IdentitiesOnly yes` directive was the specific fix for the password-prompt
symptom: with eight keys present, the client was exhausting auth attempts before
reaching the right one.

---

## Verification

- `tofu apply` → `Apply complete! Resources: 1 added` (container id 100).
- Container visible and running in the Proxmox UI under the node.
- `ssh <container>` from the workstation lands in a root shell over the tunnel,
  via the lab key, with no password and no manual flags — confirming the full
  chain: SSH config → key → tunnel → NAT bridge → IaC-declared container.

---

## Lessons / notes to self

- **Trust nothing by its label.** Two keys with the same comment caused most of
  the day's friction. Verify identity empirically (fingerprints, actual login
  tests), not by the name a human typed into a comment field.
- **`IdentitiesOnly yes` is not optional with many keys.** Without it, a
  workstation with several keys will appear to "randomly" fail auth.
- **Defaults assume a layout you may not have.** `local-lvm` is a tutorial
  default, not a guarantee. Check `pvesm status` before assuming storage names.
- **Promote recurring values to variables immediately.** The storage ID will
  appear in every resource; defining it once is cheaper than fixing it eight
  times later.
- **The errors were the lesson.** A pipeline that worked first try would have
  taught far less than six sequential, well-scoped failures did.

---

## Next phase

1. Refactor the single resource into a reusable module so each target is a few
   lines, not a full block.
2. Build cloud-init VM templates (Debian/Ubuntu) for fast cloning.
3. Define the isolated target set on the air-gapped bridge declaratively.
4. Add Ansible for in-guest configuration.
5. Reconcile remaining unaccounted-for SSH keys (audit where each is trusted).

---

*The container is trivial. The pipeline that produced it is the deliverable.*
