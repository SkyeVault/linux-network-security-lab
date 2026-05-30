# SQLi Lab 01 — WHERE Clause Hidden Data Retrieval

**Date:** 2026-05-28
**Difficulty:** Apprentice
**Platform:** PortSwigger Web Security Academy
**Category:** SQL Injection
**Tools used:** Browser

---

## Objective

Retrieve hidden products from a shopping application by injecting into the category filter parameter.

---

## Vulnerability

The app builds a query that includes `AND released=1` to hide unreleased products. Injecting into the category parameter allows bypassing this condition entirely.

---

## Payload Used

```
'+OR+1=1--
```

## What Happened

Modified the category parameter in the URL. The page returned all products including hidden ones, confirming the injection worked.

---

## Defense / Detection Angle

- WAF rules detecting `OR 1=1` or comment sequences in parameters
- Parameterized queries eliminate this entirely
- Anomalous response size compared to baseline

---

## What I Learned

Any user-controlled input passed directly into a SQL query is a potential injection point. The `released=1` filter is invisible to the user but controllable through injection.
