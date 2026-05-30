# Honeypot — OpenCanary

## Overview

OpenCanary deployed as a lightweight honeypot on the victim VLAN. It emulates common services (SSH, HTTP, SMB, FTP) and logs all interaction. This lab covers deployment, alert tuning, and analysis of any traffic that hits it.

## Services Emulated

| Service | Port | Purpose |
|---|---|---|
| SSH | 22 | Credential harvesting attempts |
| HTTP | 80/8080 | Web scanner / path traversal attempts |
| FTP | 21 | Anonymous login attempts |
| SMB | 445 | Lateral movement probing |
| MySQL | 3306 | DB enumeration attempts |

## Setup

See [`setup/`](setup/) for OpenCanary installation and configuration on Ubuntu.

## Analysis

See [`analysis/`](analysis/) for documented attacker interactions, traffic patterns, and behavioral notes.

## Alert Routing

OpenCanary alerts are forwarded to Wazuh via syslog for correlation with other lab activity.
