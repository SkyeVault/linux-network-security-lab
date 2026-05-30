# Runbook: Provisioning a New VM

## Pre-flight

- [ ] Decide: WAN A or WAN B segment? Which VLAN?
- [ ] Assign VMID from the correct range (100-199 / 200-299 / 300-399)
- [ ] Pick hostname following `<role>-<segment>-<sequence>` convention
- [ ] Determine IP assignment (static or DHCP reservation)
- [ ] Identify what ports/services this VM will expose

## Proxmox setup

1. Clone from appropriate template (VMID 900+) or create from ISO
2. Set VMID and name
3. Assign NIC to `vmbr2` with correct VLAN tag
4. Set vCPU and RAM appropriate to workload
5. Take initial snapshot: `pre-install-<date>`

## Network configuration (inside VM)

```bash
# Ubuntu — set static IP via netplan
# /etc/netplan/00-eth0.yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - <assigned-ipv4>/24
        - <assigned-ipv6>/64   # if IPv6 needed
      routes:
        - to: default
          via: <vlan-gateway>
      nameservers:
        addresses: [1.1.1.1, 2606:4700:4700::1111]
```

## Firewall rules

- [ ] Add inbound rules on the relevant OPNsense (fw-wan-a or fw-wan-b) if this VM is DMZ-exposed
- [ ] Add VLAN 310 pass rule if Wazuh agent connectivity needed
- [ ] Verify no unintended outbound from isolated VLANs

## Post-provisioning

- [ ] Install and enroll Wazuh agent (if applicable)
- [ ] Verify Zeek/Suricata sees traffic to/from this VM
- [ ] Add entry to `inventory.md` (local, gitignored)
- [ ] Document in the relevant service directory under `real-network/services/`
- [ ] Take post-install snapshot: `baseline-<date>`
