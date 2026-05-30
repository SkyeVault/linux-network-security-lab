# Active Directory Lab

## Overview

A Windows Server Active Directory environment built from scratch, then attacked using common techniques. The goal is to document both the offensive techniques and the detection opportunities they create.

## Environment

| Host | OS | Role |
|---|---|---|
| dc01 | Windows Server 2022 | Domain Controller |
| ws01 | Windows 10 | Workstation (domain-joined) |
| ws02 | Windows 10 | Workstation (domain-joined) |

**Domain:** lab.local

## Attack Techniques Covered

- Password spraying
- Kerberoasting
- AS-REP Roasting
- Pass-the-Hash / Pass-the-Ticket
- BloodHound enumeration
- DCSync
- GPO abuse

## Setup

See [`setup/`](setup/) for domain build steps, user/group configuration, and intentional misconfigurations.

## Writeups

See [`writeups/`](writeups/) for per-technique attack walkthroughs with detection notes.
