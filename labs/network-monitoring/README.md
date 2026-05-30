# Network Monitoring — Zeek / Suricata

## Overview

Zeek and Suricata run on the monitoring VLAN and receive a mirror of all victim-segment traffic. This lab covers writing detection signatures, tuning rules, and correlating network alerts with host-based Wazuh alerts.

## Setup

Both tools run on the same Ubuntu host (`10.0.30.20`) on VLAN 30, with a span/mirror port from the VLAN 20 switch.

| Tool | Purpose |
|---|---|
| Zeek | Protocol analysis, connection logs, file extraction |
| Suricata | Signature-based IDS, rule tuning, alert generation |

## Detection Goals

- [ ] Detect nmap SYN scans
- [ ] Detect Metasploit payloads in network traffic
- [ ] Detect DNS anomalies (tunneling, beaconing)
- [ ] Detect SMB enumeration / lateral movement patterns
- [ ] Detect C2 callback patterns (periodic beaconing)
- [ ] Write custom Suricata rules for lab-specific traffic

## Setup

See [`setup/`](setup/) for Zeek and Suricata installation, interface mirroring, and Elastic/Wazuh log shipping configuration.

## Writeups

See [`writeups/`](writeups/) for documented detection exercises with real packet captures and rule explanations.
