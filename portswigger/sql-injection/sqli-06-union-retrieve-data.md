# SQLi Lab 06 — UNION Attack, Retrieving Data from Other Tables

**Date:** 2026-05-28
**Difficulty:** Practitioner
**Platform:** PortSwigger Web Security Academy
**Category:** SQL Injection — UNION Attack
**Tools used:** Browser

---

## Objective

Extract usernames and passwords from a separate users table using a UNION-based injection.

---

## Vulnerability

Once column count and text-compatible columns are known, the UNION can query any accessible table — including credential stores.

---

## Payload Used

```
'+UNION+SELECT+username,password+FROM+users--
```

---

## What Happened

Usernames and passwords from the `users` table appeared as rows on the product page. Administrator credentials were visible in plaintext, used to log in and solve the lab.

---

## Defense / Detection Angle

- Passwords should never be stored in plaintext
- App DB user should only have SELECT on necessary tables
- `UNION` keyword detection in WAF rules
- Anomalous response content — credential-shaped strings appearing in product listings

---

## What I Learned

SQL injection impact goes far beyond the immediate query. Access to one injectable parameter can expose every table the database user has permission to read — including credential tables.
