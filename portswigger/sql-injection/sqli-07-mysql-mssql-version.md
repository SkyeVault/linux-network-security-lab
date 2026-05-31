# SQLi Lab 07 — MySQL/MSSQL Database Version via UNION Attack

**Date:** 2026-05-30
**Difficulty:** Practitioner
**Platform:** PortSwigger Web Security Academy
**Category:** SQL Injection — UNION Attack
**Tools used:** Burp Suite Repeater

---

## Objective
Extract the database version string from a MySQL or Microsoft SQL Server backend using a UNION-based injection in the product category filter.

---

## Vulnerability
The category filter passes user input directly into a SQL query without sanitization. MySQL and MSSQL store version information in a global system variable rather than a table, and use # instead of -- for comment syntax.

---

## Payload Used
Confirm two text columns:
```
'+UNION+SELECT+'abc','def'#
```

Extract version:
```
'+UNION+SELECT+@@version,NULL#
```

---

## What Happened
Intercepted the category filter request in Burp Suite and sent it to Repeater. Modified the category parameter with the column confirmation payload first, then swapped in the version payload. The MySQL version string appeared as a product row in the response, solving the lab.

---

## Key Differences from Oracle
| Feature | Oracle | MySQL/MSSQL |
|---------|--------|-------------|
| Comment syntax | `--` | `#` |
| Version location | `v$version` table | `@@version` variable |
| Requires FROM | Yes, use `dual` | No |

---

## Defense / Detection Angle
- WAF rules detecting UNION and @@ patterns in parameters
- Database error messages should never reach the client
- Least privilege — app DB user should not have access to system variables
- Monitor for anomalous response sizes from category filters

---

## What I Learned
Different databases have different syntax quirks that affect payload construction. MySQL uses # for comments and stores version in @@version rather than a queryable table. Knowing these differences is essential for real engagements where the database type isn't known in advance.
