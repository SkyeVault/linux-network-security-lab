# Vulnerable Network Lab

## Overview

A deliberately vulnerable network segment containing DVWA, Metasploitable 2, and VulnHub machines. The goal is to attack these systems using real tools and techniques, then document findings in reproducible writeups.

## Targets

| Machine | IP | Services | Focus |
|---|---|---|---|
| DVWA | 10.0.20.10 | HTTP (80) | Web app vulnerabilities |
| Metasploitable 2 | 10.0.20.11 | Multiple | Network services, misconfigs |
| VulnHub (varies) | 10.0.20.20+ | Varies | Full machine compromise |

## Setup

See [`setup/`](setup/) for provisioning steps and Proxmox VM configuration.

## Writeups

See [`writeups/`](writeups/) for individual attack writeups. Each follows the standard format in [`../../templates/writeup-template.md`](../../templates/writeup-template.md).
