# Linux Network Security Lab

A hands-on lab built on Proxmox for practicing and documenting offensive and defensive security techniques. Every project here includes a writeup explaining the goal, setup, findings, and takeaways.

## Lab Architecture

The lab runs on a Proxmox hypervisor with isolated network segments — an attacker VM, victim VMs, and a dedicated monitoring VM on separate VLANs so traffic flows through inspection points.

See [`architecture/`](architecture/) for network diagrams and the full topology.

## Projects

| Project | Description | Status |
|---|---|---|
| [Vulnerable Network](labs/vulnerable-network/) | DVWA, Metasploitable, VulnHub — active exploitation and writeups | In progress |
| [SIEM](labs/siem/) | Wazuh deployed on Proxmox; detecting attack traffic I generate myself | Planned |
| [Active Directory Lab](labs/active-directory/) | Build a Windows AD environment then attack it with common techniques | Planned |
| [Honeypot](labs/honeypot/) | OpenCanary deployment, traffic analysis, and attacker behavior patterns | Planned |
| [Malware Analysis](labs/malware-analysis/) | FlareVM + REMnux sandbox for static and dynamic analysis | Planned |
| [Network Monitoring](labs/network-monitoring/) | Zeek / Suricata for traffic capture, alerting, and anomaly detection | Planned |

## Tools Used

**Offensive:** Metasploit, Burp Suite, nmap, gobuster, hydra, Impacket  
**Defensive:** Wazuh, Zeek, Suricata, OpenCanary  
**Infrastructure:** Proxmox, pfSense/OPNsense, VLANs

## Writeup Format

Each lab follows the same structure so findings are comparable and reproducible. See [`templates/writeup-template.md`](templates/writeup-template.md).
