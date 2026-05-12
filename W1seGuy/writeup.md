# W1seGuy — TryHackMe Cryptographic Analysis

> **Classification:** CTF Writeup | **Vector:** Cryptographic Implementation Flaw | **Severity:** Critical

---

## 1. Threat Model

Remote service (`<TARGET_IP>:1337`) encrypts flags using a **deterministic repeating XOR cipher** with a **5-character alphanumeric key**. The predictable key length and known flag format (`THM{...}`) render the scheme vulnerable to **known-plaintext cryptanalysis**.

---

## 2. Cryptographic Weaknesses

| Control | Failure | CWE Reference |
|---------|---------|---------------|
| Key entropy | 5 chars from `[A-Za-z0-9]` → ~29.9 bits | CWE-331 |
| Key reuse | Repeating `key[i % 5]` across plaintext | CWE-323 |
| No IV/nonce | Deterministic output; identical plaintexts produce identical ciphertexts | CWE-329 |

---

## 3. Attack Vector: Known-Plaintext Recovery

**Mathematical foundation:** Given ciphertext `C` and plaintext `P`, `C ⊕ P = K`.

```python
from pwn import remote, xor

# Establish session
r = remote('<TARGET_IP>', 1337)
hex_ct = r.recvline().strip().decode()
ct = bytes.fromhex(hex_ct)

# Phase 1: Recover key bytes 0-3 via known prefix
known_prefix = b"THM{"
key_partial = xor(ct[:4], known_prefix)

# Phase 2: Brute-force key byte 4 using structural validation
for candidate in (string.ascii_letters + string.digits):
    key = key_partial + candidate.encode()
    pt = xor(ct, key)
    if pt.startswith(b"THM{") and pt.endswith(b"}"):
        r.sendline(key.decode())
        flag2 = r.recvline().strip().decode()
        print(f"Recovered Key: {key.decode()}")
        print(f"Flag 1: {pt.decode()}")
        print(f"Flag 2: {flag2}")
        break
```

---

## 4. Results

| Artifact | Value |
|----------|-------|
| **Encryption Key** | `IK0Gz` (example) |
| **Flag 1 (decrypted)** | `THM{p1alntExtAtt4ckcAnr3alLyhUrty0urxOr}` |
| **Flag 2 (server response)** | `THM{BrUt3_ForC1nG_XOR_cAn_B3_FuN_nO?}` |

---

## 5. Remediation

1. **Replace XOR with AES-256-GCM** for authenticated encryption
2. **Key length ≥ 128 bits** with CSPRNG generation
3. **Unique nonce per encryption** — never reuse keystream material

---

> **Analyst:** Miraç Akkuş (LatenT) | **Date:** 2026-05-12 | **Classification:** TLP:CLEAR
