# Runbook: Incident Response

Use this when a VM shows signs of compromise or unexpected behavior.

## Indicators to watch for

- Wazuh alert: outbound connection from an isolated VLAN
- Wazuh alert: new cron job, sshd config change, or sudoers modification
- Suricata alert: known C2 traffic pattern from a VM
- Firewall log: unexpected source IP making successful connections
- VM resource spike with no known cause

## Immediate containment

**Step 1: Isolate the VM at the hypervisor level.**

In Proxmox, remove the VM's NIC or move it to a quarantine VLAN with no routing:

```bash
# Proxmox CLI — remove network device from running VM
qm set <VMID> --delete net0

# Or change VLAN tag to quarantine VLAN (e.g. 999)
qm set <VMID> --net0 virtio,bridge=vmbr2,tag=999
```

Do this before anything else. Don't shut the VM down yet — live memory may contain useful evidence.

**Step 2: Take a snapshot for forensic purposes.**

```bash
qm snapshot <VMID> incident-<date>-<time>
```

## Evidence collection (before shutdown)

From another VM on VLAN 310 with a temporary route to the affected VLAN:

```bash
# Memory — if volatility is available
# Network connections
ss -tnp
netstat -tnp

# Running processes
ps auxf

# Recent auth
last
lastlog
journalctl -u sshd --since "1 hour ago"

# Cron / persistence
crontab -l
ls -la /etc/cron*
systemctl list-units --type=service --state=running
```

## Analysis

- Pull Zeek logs from VLAN 310 covering the timeframe: `conn.log`, `dns.log`, `http.log`
- Pull Wazuh alerts for the affected agent
- Look for lateral movement attempts to other VLANs — check firewall deny logs

## Recovery

1. Rebuild from the last known-good snapshot (pre-incident)
2. If no clean snapshot exists, rebuild from template
3. Identify and patch the vulnerability before redeploying
4. Update Suricata/Wazuh rules to detect the IOCs found
5. Document findings as a writeup in the relevant lab section
