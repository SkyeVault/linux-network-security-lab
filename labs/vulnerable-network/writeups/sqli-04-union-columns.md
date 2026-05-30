# SQLi Lab 04 — UNION Attack, Determining Number of Columns

**Date:** 2026-05-28
**Difficulty:** Practitioner
**Platform:** PortSwigger Web Security Academy
**Category:** SQL Injection — UNION Attack
**Tools used:** Browser

---

## Objective

Determine how many columns the original query returns by incrementally adding NULL values to a UNION SELECT.

---

## Vulnerability

UNION attacks require matching column counts between the original query and the injected query. Adding NULLs one at a time reveals the count when the error disappears.

---

## Payload Used

```
'+UNION+SELECT+NULL--
'+UNION+SELECT+NULL,NULL--
'+UNION+SELECT+NULL,NULL,NULL--
```

Incremented until no error returned.

---

## What Happened

The first two payloads returned errors. The third returned a valid page, confirming the original query returns three columns.

---

## Defense / Detection Angle

- Repeated requests with incrementing NULL patterns indicate enumeration
- WAF rules on `UNION SELECT` sequences
- Parameterized queries prevent exploitation entirely

---

## What I Learned

Before any UNION attack can extract data, column count must match exactly. This enumeration step is always required and follows a predictable pattern.
