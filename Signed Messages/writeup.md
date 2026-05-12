# 🔐 Signed Messages — TryHackMe Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Category](https://img.shields.io/badge/Category-Cryptography%20|%20RSA%20|%20Signature%20Forgery-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Author:** Miraç Akkuş (LatenT)  
> **Date:** May 2026  
> **Room:** [Signed Messages](https://tryhackme.com/room/lafb2026e8)

---

## 📋 Executive Summary

This writeup documents the exploitation of the **Signed Messages** machine, demonstrating a cryptographic attack against a flawed RSA implementation. The vulnerability stems from **predictable seed-based key generation**, allowing private key reconstruction from public key parameters. The attack chain progresses through signature forgery as the `admin` user to retrieve the flag from the verification endpoint.

---

## 🛠️ Tools & Technologies

| Tool | Purpose |
|------|---------|
| `curl` | HTTP interaction and endpoint testing |
| `openssl` | RSA key analysis |
| `python3` | Cryptographic computations and signature generation |
| `cryptography` library | RSA key construction and PSS padding |

---

## 🔍 Phase 1: Reconnaissance & Enumeration

### Application Endpoints

```bash
curl -s http://<TARGET_IP>:5000/
```

**Discovered Endpoints:**

| Endpoint | Function | Access |
|----------|----------|--------|
| `/register` | User registration & RSA key generation | Public |
| `/login` | Username-only authentication | Public |
| `/messages` | Public message board | Public |
| `/compose` | Message composition | Authenticated |
| `/verify` | Digital signature verification | Public |
| `/about` | Platform information | Public |

### Public Message Board (`/messages`)

- Messages displayed without raw signatures or key data
- **Welcome announcement from administrator** identified — target message for forgery
- Cryptographic operations handled server-side

### Signature Verification (`/verify`)

**Form Fields:**
- `username` — User identity
- `message` — Plaintext content
- `signature` — Hex-encoded digital signature

**Key Observations:**
- Status banner shows valid/invalid signature status
- Hidden `flag-display` div populated only upon successful admin verification

---

## 🎯 Phase 2: Authentication Analysis

### Weak Login Mechanism

```bash
curl -s -X POST \
     -d 'username=admin' \
     http://<TARGET_IP>:5000/login
```

**Critical Finding:** No password field exists. Submitting `admin` logs in as administrator immediately.

**Vulnerability:** Complete absence of password-based authentication.

---

## 🔐 Phase 3: Cryptographic Analysis

### RSA Key Generation Weakness

Upon registration, the system generates RSA key pairs. Analysis reveals:

**Vulnerability:** Keys generated from **predictable seed values** using deterministic random number generation.

**Technical Details:**
- Seed derived from user-controlled input or timestamp
- `nextprime()` applied directly to hash output
- Resulting primes approximately **256-bit** (not true 2048-bit as advertised)
- Creates approximately **512-bit RSA modulus** — trivially factorable

### Key Reconstruction

**Step 1:** Extract public key parameters from registration or key export.

**Step 2:** Factor small modulus `n` using online tools or Python:

```python
from sympy import factorint

n = <extracted_modulus>
factors = factorint(n)
p, q = list(factors.keys())
```

**Step 3:** Reconstruct private key:

```python
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.primitives import serialization

# Calculate private exponent
phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)

# Reconstruct private key
private_key = rsa.RSAPrivateNumbers(
    d=d,
    p=p,
    q=q,
    dmp1=d % (p - 1),
    dmq1=d % (q - 1),
    iqmp=pow(q, -1, p),
    public_numbers=rsa.RSAPublicNumbers(e=e, n=n)
).private_key()
```

---

## ✍️ Phase 4: Signature Forgery

### Admin Message Selection

Target message: Administrator's welcome announcement from `/messages`

### Signature Generation

```python
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import padding
import binascii

message = b"<ADMIN_WELCOME_MESSAGE>"

signature = private_key.sign(
    message,
    padding.PSS(
        mgf=padding.MGF1(hashes.SHA256()),
        salt_length=padding.PSS.MAX_LENGTH
    ),
    hashes.SHA256()
)

signature_hex = binascii.hexlify(signature).decode()
```

**Critical Note:** `PSS.MAX_LENGTH` must be used — fixed salt lengths fail due to small modulus size.

---

## 🏁 Phase 5: Flag Extraction

### Verification Request

```bash
curl -X POST http://<TARGET_IP>:5000/verify \
  -d "username=admin" \
  -d "message=<ADMIN_WELCOME_MESSAGE>" \
  -d "signature=<SIGNATURE_HEX>"
```

**Result:** Valid signature confirmed — flag displayed in response.

---

## 🗺️ Attack Path Visualization

```
[Attacker]
    │
    ├───curl───► [Target: Port 5000]
    │                │
    │                ▼
    │        [Endpoint Enumeration]
    │                │
    │                ▼
    │        [/register - Key Generation]
    │                │
    │                ▼
    │        [Extract Public Key]
    │                │
    │                ▼
    │        [Factor Small Modulus (512-bit)]
    │                │
    │                ▼
    │        [Reconstruct Private Key]
    │                │
    │                ▼
    │        [Forge Admin Signature]
    │                │
    │                ▼
    │        [POST /verify with forged sig]
    │                │
    │                ▼
    └────────► [Flag Retrieved]
```

---

## 🛡️ Vulnerability Assessment & Remediation

| # | Vulnerability | Severity | CVSS | Remediation |
|---|--------------|----------|------|-------------|
| 1 | **Predictable RSA Key Generation** | 🔴 Critical | 9.8 | Use cryptographically secure random number generator (CSPRNG); seed from OS entropy (`/dev/urandom`, `getrandom()`) |
| 2 | **Insufficient Key Size** | 🔴 Critical | 9.1 | Enforce minimum 2048-bit RSA; validate prime bit length |
| 3 | **Passwordless Authentication** | 🔴 Critical | 9.0 | Implement password-based auth; add MFA; session management |
| 4 | **RSA-PSS Salt Length Mismatch** | 🟠 High | 7.5 | Use `PSS.MAX_LENGTH` or explicitly validate salt length against key size |
| 5 | **Client-Side Key Exposure** | 🟠 High | 7.0 | Never expose private key parameters; server-side only operations |

---

## 🎓 Key Takeaways

1. **Deterministic RNG = Broken Crypto:** Predictable seeds completely undermine RSA security
2. **Small Modulus = Trivial Factorization:** 512-bit RSA is factorable in seconds on modern hardware
3. **UI ≠ Reality:** "2048-bit" marketing text contradicted actual 512-bit implementation
4. **PSS.MAX_LENGTH Critical:** Salt length must adapt to modulus size — fixed values fail with small keys
5. **Passwordless Auth is Fatal:** Single-factor username-only authentication is not authentication

---

---

## 🚩 Flag

| Flag | Value |
|------|-------|
| **Flag** | `THM{PR3D1CT4BL3_S33D5_BR34K_H34RT5}` |

---

> **⚠️ Legal Disclaimer:** This writeup is for educational and research purposes only. Always obtain explicit written authorization before testing systems you do not own.

---

**Author:** Miraç Akkuş (LatenT)  
**Date:** May 2026
```
