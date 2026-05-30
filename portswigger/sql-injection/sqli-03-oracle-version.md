# SQLi Lab 03 — Oracle Database Version via UNION Attack

**Date:** 2026-05-28
**Difficulty:** Practitioner
**Platform:** PortSwigger Web Security Academy
**Category:** SQL Injection — UNION Attack
**Tools used:** Browser, Burp Suite

---

## Objective

Extract the Oracle database version string using a UNION-based injection in the product category filter.

---

## Vulnerability

The category filter is injectable. Oracle requires `FROM` in every `SELECT` statement — the `dual` dummy table satisfies this requirement.

---

## Payloads Used

Confirm two text columns:

```
'+UNION+SELECT+'abc','def'+FROM+dual--
```

Extract version:

```
'+UNION+SELECT+BANNER,NULL+FROM+v$version--
```

---

## What Happened

Oracle version strings appeared as product rows on the page. The lab marked solved when the `BANNER` column contents were returned visibly in the response.

---

## Defense / Detection Angle

- `UNION` keyword in query parameters should trigger WAF alerts
- Database error messages should never reach the client
- Least privilege — app DB user should not have access to `v$version`

---

## What I Learned

Every database has quirks. Oracle's mandatory `FROM` clause and `dual` table are Oracle-specific knowledge that real attackers carry in their toolkit.
