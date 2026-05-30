# Runbook: Firewall Rule Changes

Changes to live firewall rules affect internet-facing infrastructure. Follow this procedure every time.

## Before the change

- [ ] Take an OPNsense config backup: System → Configuration → Backups → Download
- [ ] Document what you're changing and why (one sentence is fine)
- [ ] If opening a new port: confirm the service is actually listening and ready
- [ ] Test from outside the lab that the port is currently closed (use `nmap $WAN_IP -p <port>` from a machine off the lab network)

## Making the change (OPNsense UI)

1. Firewall → Rules → [WAN interface]
2. Add/edit rule with:
   - Protocol: TCP/UDP/both
   - Source: `any` (or specific IP range if restricted)
   - Destination: `$WAN_IP` (or alias)
   - Port: specific port(s) — never a range unless required
   - Description: what this rule is for
3. Apply changes

## After the change

- [ ] Test from outside: confirm the intended port is now open (`nmap` or `nc`)
- [ ] Confirm unrelated ports are still closed
- [ ] Check OPNsense firewall log for unexpected traffic hitting the new rule
- [ ] If closing a port: confirm nothing broke (service that was using it)

## Rolling back

Restore the downloaded config backup via System → Configuration → Backups → Restore, then apply.
