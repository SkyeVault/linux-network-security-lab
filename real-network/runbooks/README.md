# Runbooks

Operational procedures for common tasks. This is live internet-facing infrastructure — follow these before making changes.

| Runbook | When to use |
|---|---|
| [new-vm.md](new-vm.md) | Provisioning any new VM on either segment |
| [firewall-change.md](firewall-change.md) | Adding, removing, or modifying firewall rules |
| [incident-response.md](incident-response.md) | Suspected compromise of any VM |

## General principles

- **Snapshot before you change.** Every VM gets a Proxmox snapshot before significant changes.
- **One change at a time.** Don't change firewall rules and reconfigure a service simultaneously.
- **Log the change.** A brief note in the relevant runbook or git commit message is enough.
- **Test from outside.** After any firewall change, verify intended behavior from a machine outside the lab (phone on mobile data works).
