# Security Incident Report
**Date:** June 29, 2026
**Time:** 01:41 – 02:10 UTC
**Server:** [CLIENT NAME] (`[ATTACKER IP]`)
**Reported By:** [CLIENT NAME]
**Severity:** Medium
**Status:** Resolved

---

## Summary

A coordinated automated scan originating from four Google Cloud Platform (GCP) virtual machines targeted the server's web-accessible `.git/config` file. The scanner successfully retrieved the file, which contained a reference to a private GitHub repository. No credentials, source code, or sensitive data were exposed. The vulnerability has been remediated and all attacking IPs have been blocked.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 01:41:16 | First scanning IP begins probing `.git/config` paths |
| 01:41:16 | 30 requests return HTTP 200, exposing `/var/www/.git/config` |
| 01:59:33 | Second wave begins from three additional GCP IPs |
| 01:59:33 | First pass of requests returns HTTP 444 (blocked by default server config) |
| 01:59:34 | Second pass returns HTTP 200 across all probed paths |
| 02:10 | Incident discovered during routine traffic review via tcpdump |
| 02:10 | All four IPs blocked via iptables |
| 02:10 | `.git` directory removed from webroot |
| 02:10 | nginx block rule added to deny all `/.git/` access |
| 02:10 | nginx reloaded and fix verified |

---

## Attacking Infrastructure

All four source IPs belong to Google Cloud Platform customer instances, suggesting a single actor operating multiple VMs to distribute the scan.

| IP Address | Requests | Result |
|---|---|---|
| `[ATTACKER IP 1]` | 60 | 30 × HTTP 200 |
| `[ATTACKER IP 2]` | 60 | 30 × HTTP 200 |
| `[ATTACKER IP 3]` | 60 | 30 × HTTP 200 |
| `[ATTACKER IP 4]` | 30 | HTTP 200 |

**Tactics observed:**
- Automated scanning across common webroot subdirectory paths (`/backend/`, `/api/`, `/src/`, `/admin/`, `/wp-content/`, etc.)
- User-Agent rotation across each request to evade detection (cycling through dozens of spoofed mobile, desktop, and legacy browser strings)
- Parallel scanning from multiple IPs to increase speed and evade single-IP rate limiting

---

## Vulnerability

A `.git` directory was present at `/var/www/.git/`, placing it inside the nginx webroot. This occurred as a result of a `git clone` or `git init` operation being run directly inside `/var/www/` rather than outside the webroot. nginx served the contents of this directory as static files with no access restriction in place.

**File exposed:** `/var/www/.git/config`
**File size:** 21 bytes
**Contents:**
```
[remote "origin"]
    url = [GIT-REPO]
```

---

## Impact Assessment

**Confirmed exposed:**
- GitHub repository URL: `[GIT-REPO]`
- Repository name and GitHub username

**Not exposed:**
- Source code (repository is private; URL alone does not grant access)
- Credentials or API keys (none present in the config file)
- Database information
- SSH keys or server configuration
- Any other files (all 200 responses returned the same 21-byte file)

**Log review scope:**
- Current access log reviewed in full
- Rotated logs reviewed — no prior activity from these IPs found
- Attack appears isolated to this single date

---

## Remediation Steps Taken

1. **Blocked all four attacker IPs** via iptables

2. **Removed the exposed `.git` directory:**
   ```
   rm -rf /var/www/.git
   ```

3. **Created nginx snippet** at `/etc/nginx/snippets/block-git.conf`:
   ```nginx
   location ~ /\.git {
       deny all;
       return 404;
   }
   ```

4. **Applied snippet** to the primary server block in nginx site configuration

5. **Tested and reloaded nginx:**
   ```
   nginx -t && systemctl reload nginx
   ```

---

## Additional Findings During Investigation

During the investigation the following additional issues were identified and addressed separately:

- **xrdp (port 3389)** was found listening publicly — service disabled as it was not in use
- **iptables rules were not persistent** across reboots — `iptables-persistent` flagged for installation

---

## Recommendations

1. **Persist firewall rules** — install `iptables-persistent` and run `netfilter-persistent save` to ensure blocks survive reboots
2. **Never clone or init git repos inside the webroot** — keep all repositories in home directories and deploy only built output to `/var/www/`
3. **Audit all sites** — apply the `.git` nginx block to all virtual hosts, not just the primary domain
4. **Consider fail2ban rule** for `.git` probing — auto-block IPs that request `/.git/` paths
5. **Review deploy user access** — confirm all CI/CD source IPs are expected and authorized
6. **Disable unused services** — audit all listening ports periodically and disable anything not actively needed

---

## Files Modified

| File | Change |
|---|---|
| `/etc/nginx/snippets/block-git.conf` | Created — blocks `.git` access |
| `/etc/nginx/sites-enabled/[site config]` | Added `include` for block-git snippet |
| `/var/www/.git/` | Deleted entirely |
| `iptables INPUT chain` | Four DROP rules added |

---

*Report prepared June 29, 2026*
