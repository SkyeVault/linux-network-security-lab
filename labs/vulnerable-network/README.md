# Vulnerable Machine Lab

Writeups for hands-on exploitation of deliberately vulnerable machines deployed in the Proxmox lab. This is Track 3 — real attack chains against real deployed targets, not platform-guided labs.

For PortSwigger Web Security Academy labs, see [`../../portswigger/`](../../portswigger/).

## Planned Targets

| Machine | VLAN | IP | Focus |
|---|---|---|---|
| DVWA | 20 | 10.0.20.10 | Web app vulnerabilities — SQLi, XSS, CSRF, file upload, command injection |
| Metasploitable 2 | 20 | 10.0.20.11 | Network services, misconfigured daemons, Samba, vsftpd |
| VulnHub (rotating) | 20 | 10.0.20.20+ | Full machine compromise — boot to root |

## Setup

See [`setup/`](setup/) for Proxmox provisioning steps. These machines are air-gapped on VLAN 20 — no internet access.

## Writeups

See [`writeups/`](writeups/) for completed attack writeups. Each uses [`../../templates/writeup-template.md`](../../templates/writeup-template.md), covering recon through post-exploitation with a blue team detection section.

## When this comes online

This section fills in once the Proxmox real-network infrastructure (Track 2) is deployed. The victim VLAN will be isolated from the monitoring VLAN, and Wazuh agents on these machines will generate real alerts from attacks run against them.
