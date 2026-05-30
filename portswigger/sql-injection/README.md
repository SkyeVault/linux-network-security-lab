# SQL Injection

**Progress: 6 / 18 labs complete**

SQL injection is the starting point — it covers the widest range of technique patterns (error-based, UNION-based, blind, out-of-band) and every other injection class shares the same underlying logic of breaking out of a data context into a control context.

Writeups: each file maps to one lab. Naming follows `sqli-NN-short-description.md`.

---

## Lab Tracker

### Apprentice

| # | Lab | Writeup | Done |
|---|---|---|---|
| 01 | SQL injection vulnerability in WHERE clause allowing retrieval of hidden data | [sqli-01-where-clause.md](sqli-01-where-clause.md) | ✅ |
| 02 | SQL injection vulnerability allowing login bypass | [sqli-02-login-bypass.md](sqli-02-login-bypass.md) | ✅ |

### Practitioner

| # | Lab | Writeup | Done |
|---|---|---|---|
| 03 | SQL injection UNION attack, determining the number of columns | [sqli-04-union-columns.md](sqli-04-union-columns.md) | ✅ |
| 04 | SQL injection UNION attack, finding a column containing text | [sqli-05-union-text-column.md](sqli-05-union-text-column.md) | ✅ |
| 05 | SQL injection UNION attack, retrieving data from other tables | [sqli-06-union-retrieve-data.md](sqli-06-union-retrieve-data.md) | ✅ |
| 06 | SQL injection UNION attack, retrieving multiple values in a single column | — | ⬜ |
| 07 | SQL injection attack, querying the database type and version on Oracle | [sqli-03-oracle-version.md](sqli-03-oracle-version.md) | ✅ |
| 08 | SQL injection attack, querying the database type and version on MySQL and Microsoft | — | ⬜ |
| 09 | SQL injection attack, listing the database contents on non-Oracle databases | — | ⬜ |
| 10 | SQL injection attack, listing the database contents on Oracle | — | ⬜ |
| 11 | Blind SQL injection with conditional responses | — | ⬜ |
| 12 | Blind SQL injection with conditional errors | — | ⬜ |
| 13 | Visible error-based SQL injection | — | ⬜ |
| 14 | Blind SQL injection with time delays | — | ⬜ |
| 15 | Blind SQL injection with time delays and information retrieval | — | ⬜ |
| 16 | Blind SQL injection with out-of-band interaction | — | ⬜ |
| 17 | Blind SQL injection with out-of-band data exfiltration | — | ⬜ |
| 18 | SQL injection with filter bypass via XML encoding | — | ⬜ |

---

## Technique Map

The 18 labs are not independent exercises — they build a complete picture of one attack class:

```
Classic (visible output)
├── WHERE clause bypass          → labs 01-02
├── UNION attacks                → labs 03-07
│   ├── column count             → lab 03
│   ├── column type              → lab 04
│   ├── cross-table retrieval    → labs 05-06
│   └── DB fingerprinting        → labs 07-10
└── Error-based (visible errors) → lab 13

Blind (no visible output)
├── Boolean-based                → labs 11-12
├── Time-based                   → labs 14-15
└── Out-of-band (DNS/HTTP)       → labs 16-17

Filter bypass                    → lab 18
```

Completing the blind labs (11-17) is where real depth comes from — those techniques apply whenever output is suppressed, which is most real-world applications.
