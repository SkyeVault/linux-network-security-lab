# SIEM Lab — Wazuh

## Overview

Wazuh deployed on Proxmox on the monitoring VLAN (VLAN 30). Wazuh agents are installed on victim machines and the firewall. The attacker VM generates real attack traffic; this lab documents what Wazuh detects, what it misses, and how to tune rules.

## Architecture

- Wazuh Manager: `10.0.30.10`
- Agents: DVWA (10.0.20.10), Metasploitable (10.0.20.11)
- Log sources: syslog, auditd, apache2 access/error logs

## Setup

See [`setup/`](setup/) for Wazuh installation, agent enrollment, and dashboard configuration.

## Detections

See [`detections/`](detections/) for documented detection rules and real alerts triggered during attack exercises.

## Goals

- [ ] Detect port scanning (nmap)
- [ ] Detect brute force (hydra against SSH/HTTP)
- [ ] Detect web exploitation (SQLi, LFI in DVWA)
- [ ] Detect reverse shell callbacks
- [ ] Detect privilege escalation on victim hosts
