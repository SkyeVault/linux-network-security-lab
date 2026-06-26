# Linux Security Lab — DDoS Incident Response & Server Hardening
**Date:** June 2026  
**System:** Ubuntu 24.04 LTS on Proxmox KVM  
**Purpose:** Incident response documentation and hardening reference

---

## Table of Contents

1. [Incident Overview](#incident-overview)
2. [Attack Timeline](#attack-timeline)
3. [Live Triage & Diagnosis](#live-triage--diagnosis)
4. [Local Mitigations Applied](#local-mitigations-applied)
5. [DNS Migration to Cloudflare](#dns-migration-to-cloudflare)
6. [Mail Stack Hardening](#mail-stack-hardening)
7. [Fail2ban Configuration](#fail2ban-configuration)
8. [System Cleanup](#system-cleanup)
9. [Monitoring & Logging](#monitoring--logging)
10. [Persistent Issues & TODO](#persistent-issues--todo)
11. [Key Lessons Learned](#key-lessons-learned)

---

## Incident Overview

The server began experiencing a large-scale DDoS attack. Initial reports indicated a carpet bomb UDP flood from a residential botnet at ~67 Mbps. The attack escalated significantly and was later revealed to be part of a 600 Gbps ransom DDoS targeting the upstream provider, causing total IPv4 packet loss for several hours.

The server itself remained stable throughout — CPU load never exceeded 27%, and RAM usage stayed well under capacity. The bottleneck was entirely upstream bandwidth saturation at the provider level.

The attack was actively managed by the threat actor, who switched vectors multiple times in response to mitigations — a sign of a human operator rather than a dumb script.

---

## Attack Timeline

| Time | Event |
|---|---|
| T+0 | Carpet bomb UDP flood reported at ~67 Mbps |
| T+25m | Gateway ping showing 77.7% packet loss, ~391ms RTT |
| T+46m | tcpdump confirms DNS water torture attack on port 53 |
| T+60m | Attack pivots to TCP SYN flood on port 443 |
| T+75m | HTTP flood on port 80 begins from residential botnet IPs |
| T+100m | Attack returns to DNS water torture with randomized subdomain casing |
| T+120m | Provider confirms 600 Gbps ransom DDoS against entire network |
| T+6h | Upstream carriers implement mitigation, connectivity restored |

---

## Live Triage & Diagnosis

### Confirming the box was not CPU-bound

```bash
htop
```

Result: All CPU cores under 27%, load average well below 1.0. Confirmed as upstream bandwidth saturation, not local resource exhaustion.

### Identifying the correct network interface

```bash
ip link
ip -s link show eth0   # replace eth0 with your interface
```

Look for the interface with high RX packet/byte counts — that's your WAN facing the flood.

### Capturing attack traffic

```bash
sudo tcpdump -i eth0 -w ~/attack.pcap -c 10000 udp
sudo tcpdump -r ~/attack.pcap -n | head -50
```

### DNS Water Torture Identification

Captured packets showed randomized subdomain casing being used to bypass DNS caching:

```
IP x.x.x.x > your.server.53: A? Ns1.YoUrDoMaIn.CoM.
IP x.x.x.x > your.server.53: A? NS1.YOURDOMAIN.COM.
```

Each randomized query misses the cache and forces a fresh authoritative lookup, exhausting DNS server resources.

### Verifying DNS server is not an open resolver

```bash
dig @YOUR_SERVER_IP google.com
```

Expected result: `REFUSED` — server should reject recursive queries for external domains.

### Checking firewall state

```bash
sudo iptables -L INPUT -n -v | head -20
sudo ufw status numbered
```

### Monitoring interface traffic in real time

```bash
watch -n 1 'ip -s link show eth0'
```

### Checking kernel packet drops

```bash
cat /proc/net/softnet_stat
```

Second column shows drop count — if climbing fast, the kernel can't keep up.

### Top attacking source IPs

```bash
sudo tcpdump -i eth0 -n udp port 53 -c 500 2>/dev/null | awk '{print $3}' | cut -d. -f1-3 | sort | uniq -c | sort -rn | head -20
```

---

## Local Mitigations Applied

### Blocking attacking subnets via UFW

```bash
sudo ufw deny from ATTACKING_SUBNET/24
```

Key note: Use `ufw deny` not raw `iptables -A` — on Ubuntu with UFW active, raw iptables rules appended with `-A` end up below UFW's jump chains and never fire. Verify with:

```bash
sudo iptables -L INPUT -n -v --line-numbers
```

Rules should appear above the `ufw-before-logging-input` chain, or use `-I INPUT 1` for insertion.

### Blocking via iptables at top of chain

```bash
sudo iptables -I INPUT 1 -s ATTACKING_SUBNET/24 -j DROP
```

**Important:** Raw iptables rules do not survive reboots. To make persistent:

```bash
sudo apt install netfilter-persistent
sudo netfilter-persistent save
```

### Fixing a deadlocked UFW

If UFW freezes (e.g. from Ctrl+C during sudo prompt):

```bash
sudo rm /run/ufw.lock
sudo ufw status
```

### Note on iptables flush

The command `sudo iptables -F INPUT` wipes ALL firewall rules including UFW chains. Avoid during active attacks. If accidentally run, a reboot restores UFW rules (UFW rules are persistent by default).

---

## DNS Migration to Cloudflare

### Motivation

DNS water torture attacks target your authoritative nameserver directly. Moving to Cloudflare nameservers removes the attack vector from your infrastructure — Cloudflare absorbs the flood at their scale.

### Process

1. Create Cloudflare account at dash.cloudflare.com
2. Add your domain — Cloudflare auto-imports existing DNS records
3. Review and correct proxy settings before switching nameservers
4. Change nameservers at your registrar to Cloudflare's assigned nameservers

### Proxy settings — what to orange cloud vs grey cloud

**Orange cloud (proxied — hides origin IP):**
- Main website, www, and pure web subdomains

**Grey cloud (DNS only — exposes real IP, required for):**
- IRC server — Cloudflare cannot proxy IRC ports
- Mail server — Cloudflare cannot proxy SMTP/IMAP
- Authoritative nameserver records (ns1, etc.)
- Any non-HTTP service (VPN, game servers, etc.)

### Key DNS records to verify after migration

```
SPF:   v=spf1 mx ip4:YOUR_SERVER_IP -all
DKIM:  present and DNS only
DMARC: v=DMARC1; p=reject; rua=mailto:admin@yourdomain.com
MX:    mail.yourdomain.com
```

### Cloudflare security settings to configure

- SSL/TLS → Full (Strict)
- Security → WAF → enabled
- Bot Fight Mode → enabled
- Security Level → High

### Clean up stale DNS records

After migration, audit and remove orphaned records — old cPanel artifacts, test subdomains, and records pointing to defunct services all expand your attack surface unnecessarily.

---

## Mail Stack Hardening

### Problem 1: opendmarc socket not connecting to Postfix

Postfix runs in a chroot jail at `/var/spool/postfix`. Unix socket paths are invisible inside the chroot, causing silent milter connection failures.

**Symptoms in mail.log:**
```
warning: connect to Milter service local:/run/opendmarc/opendmarc.sock: No such file or directory
```

**Root cause:** Even with correct socket permissions, Postfix chroot cannot access host filesystem paths.

**Fix: Switch opendmarc to TCP**

In `/etc/opendmarc.conf`:
```
Socket inet:12345@localhost
```

In `/etc/postfix/main.cf`:
```
smtpd_milters = inet:localhost:12301, inet:localhost:12345
non_smtpd_milters = inet:localhost:12301, inet:localhost:12345
```

```bash
sudo systemctl restart opendmarc postfix
```

**Verify opendmarc is listening:**
```bash
sudo ss -tlnp | grep 12345
```

### Problem 2: opendmarc ignoring connections

```
opendmarc: ignoring connection from localhost
```

This is not an error for locally-originated mail — opendmarc correctly skips DMARC checks for mail generated on the server itself. For external inbound mail, set:

In `/etc/opendmarc.conf`:
```
AuthservID mail.yourdomain.com
TrustedAuthservIDs mail.yourdomain.com, localhost
```

### Problem 3: RejectFailures not enabled

opendmarc detects DMARC failures but does not reject by default.

In `/etc/opendmarc.conf`:
```
RejectFailures true
```

### Problem 4: Legitimate outbound mail rejected by DMARC

Mail clients connecting from home/office IPs may fail DMARC if the connection is not routed through your mail server's submission port with proper authentication, or if the client's IP is not in your SPF record.

**Fix: Add trusted IPs to opendmarc ignore list**

```bash
sudo mkdir -p /etc/opendmarc
sudo nano /etc/opendmarc/ignore.hosts
```

Add IPs that should bypass DMARC checking:
```
127.0.0.1
localhost
YOUR_TRUSTED_CLIENT_IP
```

In `/etc/opendmarc.conf`:
```
IgnoreHosts /etc/opendmarc/ignore.hosts
```

### Problem 5: milter_default_action = accept is a silent hole

When milters fail to connect, Postfix falls back to accepting mail. This means a broken opendmarc silently stops enforcing DMARC.

In `/etc/postfix/main.cf`, consider:
```
milter_default_action = tempfail
```

This causes Postfix to temporarily reject mail when milters are unavailable rather than silently accepting everything.

### Final opendmarc.conf (active settings only)

```
AuthservID mail.yourdomain.com
PidFile /run/opendmarc/opendmarc.pid
PublicSuffixList /usr/share/publicsuffix/public_suffix_list.dat
RejectFailures true
Socket inet:12345@localhost
Syslog true
TrustedAuthservIDs mail.yourdomain.com, localhost
UMask 0002
UserID opendmarc
IgnoreHosts /etc/opendmarc/ignore.hosts
```

### Verification

Test that opendmarc processes inbound mail:
```bash
sudo grep opendmarc /var/log/mail.log | tail -10
```

Expected for legitimate external mail:
```
opendmarc[xxxxx]: MESSAGE_ID: gmail.com pass
```

Expected for local/trusted client mail:
```
opendmarc[xxxxx]: ignoring connection from [TRUSTED_IP]
```

---

## Fail2ban Configuration

### Problem: fail2ban not catching SASL brute force

fail2ban was running but had zero matches despite constant authentication attempts. The postfix-sasl jail had no log file configured — it was configured for the systemd journal but postfix logs to `/var/log/mail.log` via syslog on this system.

**Diagnosis:**
```bash
sudo fail2ban-client get postfix-sasl logpath
# Returns: No file is currently monitored

sudo fail2ban-regex systemd-journal /etc/fail2ban/filter.d/postfix-sasl.conf
# Returns: Lines: 0 lines, 0 ignored, 0 matched, 0 missed

# Confirm logs are in file, not journal:
ls -la /var/log/mail*
```

**Fix 1: Create the missing filter**

```bash
sudo nano /etc/fail2ban/filter.d/postfix-sasl.conf
```

```
[Definition]
failregex = warning: [-._\w]+\[<HOST>\]: SASL (LOGIN|PLAIN|(?:CRAM|DIGEST)-MD5) authentication failed
ignoreregex =
```

**Fix 2: Create jail config pointing to mail.log**

```bash
sudo nano /etc/fail2ban/jail.d/postfix-sasl.local
```

```
[postfix-sasl]
enabled = true
filter = postfix-sasl
backend = auto
logpath = /var/log/mail.log
maxretry = 3
findtime = 300
bantime = 3600
```

```bash
sudo systemctl restart fail2ban
```

**Test the regex against your actual logs:**
```bash
sudo fail2ban-regex /var/log/mail.log /etc/fail2ban/filter.d/postfix-sasl.conf
```

Should show matched lines, not zero.

**Verification:**
```bash
sudo fail2ban-client status postfix-sasl
```

Should show currently banned IPs after a few minutes of operation.

### Note on systemd journal vs syslog

On modern Ubuntu systems, Postfix may log to either the systemd journal or syslog depending on configuration. Check which one is active:

```bash
sudo journalctl -u postfix --since "1 hour ago" | head -5
# If returns "No entries" — postfix is using syslog, not journal
```

If using journal, the systemd unit name may differ from what fail2ban expects:
```bash
sudo journalctl -o json --since "10 minutes ago" | grep -i sasl | python3 -c "import sys,json; [print(json.loads(l).get('_SYSTEMD_UNIT','')) for l in sys.stdin]" | sort -u
```

Use the actual unit name in your jail's `journalmatch` setting.

---

## System Cleanup

### Removing crash-looping services

Services that crash loop waste CPU, pollute logs, and create noise that obscures real security events. Audit regularly:

```bash
sudo systemctl list-units --state=failed
sudo docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Image}}"
```

Docker containers with high restart counts are a red flag:
```bash
sudo docker inspect CONTAINER_NAME | grep -i restart
```

### Stopping and disabling unused services

```bash
sudo systemctl stop SERVICE_NAME
sudo systemctl disable SERVICE_NAME
sudo rm /etc/systemd/system/SERVICE_NAME.service
sudo systemctl daemon-reload
```

### Removing unused Docker containers and images

```bash
sudo docker stop CONTAINER_NAME
sudo docker rm CONTAINER_NAME
sudo docker image prune -a    # removes ALL unused images — confirm before running
sudo docker system df         # check space usage before/after
```

### Dropping orphaned databases

```bash
sudo mysql -e "SHOW DATABASES;"
sudo mysql -e "DROP DATABASE database_name;"
```

### Cleaning up orphaned nginx configs

Stale nginx configs for defunct services expand your attack surface and create confusion. Remove them:

```bash
ls /etc/nginx/sites-enabled/
ls /etc/nginx/sites-available/
sudo rm /etc/nginx/sites-enabled/DEFUNCT_SITE
sudo rm /etc/nginx/sites-available/DEFUNCT_SITE
sudo nginx -t && sudo systemctl reload nginx
```

Always run `nginx -t` before reloading to verify config is valid.

---

## Monitoring & Logging

### Daily security check script

Save as `~/check_logs.sh` and `chmod +x`:

```bash
#!/bin/bash

echo "========================================="
echo "  Daily Security Check - $(date)"
echo "========================================="

echo ""
echo "--- FAIL2BAN STATUS ---"
sudo fail2ban-client status postfix-sasl 2>/dev/null | grep -E "banned|failed"
sudo fail2ban-client status dovecot 2>/dev/null | grep -E "banned|failed"
sudo fail2ban-client status sshd 2>/dev/null | grep -E "banned|failed"

echo ""
echo "--- RECENT MAIL FAILURES (last 24h) ---"
sudo grep "$(date -d '24 hours ago' '+%Y-%m-%d')\|$(date '+%Y-%m-%d')" /var/log/mail.log | grep -E "SASL|reject|warning|error" | tail -20

echo ""
echo "--- NGINX SUSPICIOUS REQUESTS (last 24h) ---"
sudo grep "$(date '+%d/%b/%Y')" /var/log/nginx/access.log | grep -vE "200|301|302|304" | awk '{print $1, $7, $9}' | sort | uniq -c | sort -rn | head -20

echo ""
echo "--- WEBSHELL SCAN ATTEMPTS (last 24h) ---"
sudo grep "$(date '+%d/%b/%Y')" /var/log/nginx/access.log | grep "\.php" | awk '{print $1}' | sort | uniq -c | sort -rn | head -10

echo ""
echo "--- UFW BLOCKS (last 50) ---"
sudo grep 'UFW BLOCK' /var/log/syslog | tail -50 | grep -oP 'SRC=\K[^ ]+' | sort | uniq -c | sort -rn | head -10

echo ""
echo "--- FAILED SSH ATTEMPTS ---"
sudo lastb 2>/dev/null | head -10

echo ""
echo "--- DISK USAGE ---"
df -h | grep -v tmpfs

echo ""
echo "--- MEMORY ---"
free -h

echo ""
echo "--- SERVICES STATUS ---"
for service in postfix dovecot nginx fail2ban opendmarc opendkim; do
    status=$(systemctl is-active $service 2>/dev/null)
    echo "  $service: $status"
done

echo ""
echo "--- DOCKER CONTAINERS ---"
sudo docker ps --format "table {{.Names}}\t{{.Status}}" 2>/dev/null

echo ""
echo "--- TOP 10 IPs HITTING YOUR SERVER TODAY ---"
sudo grep "$(date '+%d/%b/%Y')" /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10

echo ""
echo "========================================="
echo "  Check Complete"
echo "========================================="
```

### Schedule as daily cron with email report

```bash
(crontab -l 2>/dev/null; echo "0 7 * * * /home/YOUR_USER/check_logs.sh | mail -s 'Daily Security Check' admin@yourdomain.com") | crontab -
```

### Useful one-off commands

```bash
# Live mail log
sudo tail -f /var/log/mail.log

# Live auth log
sudo tail -f /var/log/auth.log

# Live nginx access
sudo tail -f /var/log/nginx/access.log

# All logs filtered for failures
sudo tail -f /var/log/auth.log /var/log/mail.log /var/log/nginx/access.log | grep -i "fail\|error\|invalid\|attack\|refused"

# Who is currently connected
ss -tnp | grep ESTABLISHED

# Login history
last | head -20

# Failed login history
sudo lastb | head -20

# Lookup an IP
whois TARGET_IP
dig -x TARGET_IP

# Check what a subnet is hitting
sudo tcpdump -i eth0 -n src net TARGET_SUBNET/24 -c 100

# See top source IPs in nginx log today
sudo grep "$(date '+%d/%b/%Y')" /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -20

# Check fail2ban is reading logs correctly
sudo fail2ban-regex /var/log/mail.log /etc/fail2ban/filter.d/postfix-sasl.conf

# Verify iptables rules and hit counts
sudo iptables -L INPUT -n -v --line-numbers

# Save iptables rules across reboots
sudo netfilter-persistent save
```

---

## Persistent Issues & TODO

| Task | Priority | Notes |
|---|---|---|
| Run pending security updates | High | `sudo apt update && sudo apt upgrade -y` |
| Make iptables blocks persistent | High | `sudo netfilter-persistent save` |
| Add BIND rate limiting | Medium | Add `rate-limit { responses-per-second 5; };` to named.conf.options |
| Set up secondary server | Medium | Redundancy for IRC and mail |
| Move IRC to DDoS-protected host | Medium | Consider Path.net, Frantech, or Voxility-backed provider |
| Set up secondary nameserver | Low | Single point of failure for DNS currently |
| IRC IP cloaking | Low | InspIRCd cloak module protects user IPs from other users |

---

## Key Lessons Learned

**1. Upstream mitigation is the only real fix for volumetric DDoS.**
At 600 Gbps, nothing you do locally matters. The pipe is full before packets reach your box. Get a provider with upstream scrubbing (Voxility, Path.net) or be prepared to wait. Know your provider's DDoS escalation process before an incident.

**2. Carpet bomb UDP floods are designed to evade per-IP thresholds.**
Instead of hammering one IP, the botnet spreads traffic across an entire subnet, keeping per-IP traffic below detection limits while saturating total bandwidth. Traditional rate-limiting per IP is ineffective.

**3. DNS water torture is an intelligent, targeted attack.**
Randomized subdomain casing (`YoUrSuBdOmAiN.example.com`) bypasses DNS caching and forces your authoritative server to respond to every query. Moving to a large provider's nameservers (Cloudflare, etc.) eliminates this vector.

**4. UFW and raw iptables interact in non-obvious ways.**
Raw `iptables -A` appends rules below UFW's jump chains — they never fire. Use `iptables -I INPUT 1` to insert above UFW, or use `ufw deny` commands consistently. Always verify with `-v` to check packet counters.

**5. Postfix chroot breaks Unix socket milter connections silently.**
opendmarc, OpenDKIM, and other milters connected via Unix sockets fail when Postfix runs chrooted. The failure is silent by default (`milter_default_action = accept`). Always use TCP for milter connections on chrooted Postfix.

**6. `milter_default_action = accept` is a silent security hole.**
When milters fail to connect, Postfix accepts mail anyway. A broken opendmarc means no DMARC enforcement. Verify milter connectivity independently after any config change.

**7. Docker containers with restart policies can crash loop indefinitely.**
Containers with `restart: unless-stopped` will spin forever if misconfigured, wasting CPU and polluting logs. Monitor restart counts regularly and disable services you aren't actively using.

**8. Cloudflare free tier is a significant security upgrade.**
Moves authoritative DNS off your infrastructure, hides origin IPs for web services, provides WAF and bot protection, and includes analytics — free. The main limitation is it cannot proxy non-HTTP services like IRC or SMTP.

**9. Test your defenses before an incident.**
UFW locking up, iptables rules in the wrong position, and fail2ban silently ignoring logs are all things that compound stress during a live attack. Run `fail2ban-regex` against your actual log files periodically to confirm jails are working.

**10. IRC networks are persistent DDoS targets.**
Channel conflicts, network rivalries, and personal disputes regularly motivate DDoS attacks against IRC infrastructure. If you run an IRC server, plan for this — it is not a matter of if, but when. Run IRC on DDoS-protected infrastructure if possible.

---

*Incident response documentation — Linux security lab reference.*
