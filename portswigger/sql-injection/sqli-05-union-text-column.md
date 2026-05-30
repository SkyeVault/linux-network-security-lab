# SQLi Lab 05 — UNION Attack, Finding a Column Containing Text

**Date:** 2026-05-28
**Difficulty:** Practitioner
**Platform:** PortSwigger Web Security Academy
**Category:** SQL Injection — UNION Attack
**Tools used:** Browser

---

## Objective

Identify which column in the query returns text data so string-based payloads can be injected into that position.

---

## Vulnerability

Not all columns accept text — some may be integer typed. Placing a string value in each position reveals which columns can carry text output.

---

## Payload Used

```
'+UNION+SELECT+'abc',NULL,NULL--
'+UNION+SELECT+NULL,'abc',NULL--
'+UNION+SELECT+NULL,NULL,'abc'--
```

Rotated the string through each position until it appeared in the response.

---

## What Happened

The string `'abc'` appeared on the page when placed in the correct column position, confirming that column accepts and returns text data.

---

## Defense / Detection Angle

- String literals appearing in unexpected response locations indicate active probing
- Logging and alerting on modified category parameters
- Input validation rejecting non-alphanumeric characters

---

## What I Learned

Column type matters for data extraction. Identifying text-compatible columns is a required step before retrieving meaningful string data from the database.
