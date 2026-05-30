# Linux Network Security Lab

An active learning security lab with three tracks: systematic platform-based study, real enterprise infrastructure on Proxmox with live public IPs, and hands-on exploitation of deployed vulnerable machines. Everything generates writeups.

---

## Track 1 — Learning Path (PortSwigger Web Security Academy)

Working through the full PortSwigger curriculum in order. Each topic has a tracker and individual writeups per lab.

| Topic | Progress |
|---|---|
| [SQL Injection](portswigger/sql-injection/) | 6 / 18 |
| Authentication | Planned |
| Path Traversal | Planned |
| OS Command Injection | Planned |
| XSS | Planned |
| ... | |

[Full topic list and progress →](portswigger/)

---

## Track 2 — Real Network Infrastructure

A production Proxmox node with two public IPv4 addresses and dual /64 IPv6 subnets. Internet-facing — honeypots, sensors, and SIEM are live. Configuration and runbooks documented here; IPs are kept local.

| Component | Description |
|---|---|
| [Architecture](real-network/architecture/) | VLAN topology, bridge design, IPv6 addressing |
| [Gateway / Firewall](real-network/gateway/) | OPNsense per WAN, firewall rule philosophy |
| [Services](real-network/services/) | Honeypot, SIEM, network monitoring |
| [Runbooks](real-network/runbooks/) | New VM, firewall changes, incident response |

[Real network documentation →](real-network/)

---

## Track 3 — Deployed Machine Labs

Hands-on exploitation of deliberately vulnerable machines running in the lab. Writeups cover full compromise chains — recon through post-exploitation — with a blue team detection angle on each.

| Lab | Description | Status |
|---|---|---|
| [Vulnerable Network](labs/vulnerable-network/) | DVWA, Metasploitable 2, VulnHub machines | Planned |
| [Active Directory](labs/active-directory/) | Windows AD built to attack — Kerberoasting, DCSync, BloodHound | Planned |
| [Honeypot](labs/honeypot/) | OpenCanary — deployed, collecting real traffic | Planned |
| [Malware Analysis](labs/malware-analysis/) | FlareVM + REMnux sandbox | Planned |
| [SIEM](labs/siem/) | Wazuh — detecting attacks generated from Track 3 machines | Planned |
| [Network Monitoring](labs/network-monitoring/) | Zeek + Suricata on span port | Planned |

---

## Tools

**Offensive:** Metasploit, Burp Suite, nmap, gobuster, hydra, Impacket, BloodHound  
**Defensive:** Wazuh, Zeek, Suricata, OpenCanary  
**Infrastructure:** Proxmox, OPNsense, VLANs

## Templates

- [Platform lab writeup](templates/writeup-platform-lab.md) — PortSwigger, HTB Academy, THM
- [Machine writeup](templates/writeup-template.md) — DVWA, Metasploitable, VulnHub, HTB boxes
