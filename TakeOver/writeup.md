# 🌐 TakeOver — TryHackMe Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green)
![Category](https://img.shields.io/badge/Category-Web%20|%20Subdomain%20Enumeration%20|%20SSL%20Certificate%20Analysis-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Author:** Miraç Akkuş (LatenT)  
> **Date:** May 2026  
> **Room:** [TakeOver](https://tryhackme.com/room/takeover)

---

## 📋 Executive Summary

This writeup documents the exploitation of the **TakeOver** machine, demonstrating **subdomain enumeration** and **SSL certificate analysis** techniques to discover hidden infrastructure. The attack chain leverages virtual host fuzzing to identify subdomains, certificate Subject Alternative Name (SAN) inspection to uncover secret subdomains, and HTTP header analysis to extract the flag from a misconfigured AWS S3 redirect.

---

## 🛠️ Tools & Technologies

| Tool | Purpose |
|------|---------|
| `nmap` | Network reconnaissance and service enumeration |
| `ffuf` | Fast web fuzzer for virtual host/subdomain discovery |
| `curl` | HTTP request testing and header analysis |
| `openssl` | SSL/TLS certificate inspection |
| `gobuster` | Directory enumeration |

---

## 🔍 Phase 1: Reconnaissance & Enumeration

### Network Scanning

```bash
nmap -sV -sC -T4 <TARGET_IP>
```

### Findings

| Port | State | Service | Version | Notes |
|------|-------|---------|---------|-------|
| 22/tcp | open | SSH | OpenSSH 8.2p1 Ubuntu | Standard access |
| 80/tcp | open | HTTP | Apache httpd 2.4.41 | Redirects to HTTPS |
| 443/tcp | open | SSL/HTTP | Apache httpd 2.4.41 | SSL certificate present |

**Key Observations:**
- Port 80 redirects to `https://futurevera.thm`
- SSL certificate expired (Not valid after: 2023-03-13)
- Custom domain `futurevera.thm` requires `/etc/hosts` configuration

### Host File Configuration

```bash
echo "<TARGET_IP> futurevera.thm" | sudo tee -a /etc/hosts
```

---

## 🎯 Phase 2: Subdomain Enumeration

### Baseline Response Analysis

Before fuzzing, establish the baseline response size for non-existent subdomains:

```bash
curl -I -k -H "Host: randomname.futurevera.thm" https://<TARGET_IP>
```

**Result:** Response size = **4605 bytes** (standard error page for invalid subdomains)

### Virtual Host Fuzzing with ffuf

```bash
ffuf -u https://<TARGET_IP> \
     -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
     -H "Host: FUZZ.futurevera.thm" \
     -fs 4605
```

**Discovered Subdomains:**

| Subdomain | Status | Size | Notes |
|-----------|--------|------|-------|
| `support.futurevera.thm` | 200 | 1522 | Support page |
| `blog.futurevera.thm` | 200 | 3838 | Blog page |

### Host File Update

```bash
echo "<TARGET_IP> support.futurevera.thm blog.futurevera.thm" | sudo tee -a /etc/hosts
```

---

## 🔐 Phase 3: SSL Certificate Analysis

### Certificate Inspection (support.futurevera.thm)

```bash
nmap -p 443 --script ssl-cert support.futurevera.thm
```

**Certificate Details:**

| Field | Value |
|-------|-------|
| Subject | `CN=support.futurevera.thm` |
| Organization | Futurevera |
| State | Oregon |
| Country | US |
| **Subject Alternative Name** | `DNS:secrethelpdesk934752.support.futurevera.thm` |

**Critical Finding:** Hidden subdomain discovered in SAN field — `secrethelpdesk934752.support.futurevera.thm`

### Certificate Inspection (blog.futurevera.thm)

```bash
nmap -p 443 --script ssl-cert blog.futurevera.thm
```

**Result:** No additional SAN entries — standard certificate only.

---

## 🌐 Phase 4: Secret Subdomain Access

### Host File Configuration

```bash
echo "<TARGET_IP> secrethelpdesk934752.support.futurevera.thm" | sudo tee -a /etc/hosts
```

### HTTP vs HTTPS Behavior Analysis

| Protocol | Response | Notes |
|----------|----------|-------|
| HTTPS | Certificate error / timeout | SSL certificate mismatch |
| HTTP | Immediate 302 redirect | Clean response |

### curl Header Analysis

```bash
curl -I http://secrethelpdesk934752.support.futurevera.thm
```

**Response:**
```http
HTTP/1.1 302 Found
Date: Sat, 15 Apr 2023 18:03:13 GMT
Server: Apache/2.4.41 (Ubuntu)
Location: http://flag{beea0d6edfcee06a59b83fb50ae81b2f}.s3-website-us-west-3.amazonaws.com/
Content-Type: text/html; charset=UTF-8
```

---

## 🗺️ Attack Path Visualization

```
[Attacker]
    │
    ├───nmap───► [Target: 22, 80, 443]
    │                │
    │                ▼
    │        [futurevera.thm configured in /etc/hosts]
    │                │
    │                ▼
    │        [Baseline: 4605 bytes error page]
    │                │
    │                ▼
    │        [ffuf vhost fuzzing]
    │                │
    │                ▼
    │        [support.futurevera.thm discovered]
    │                │
    │                ▼
    │        [SSL certificate SAN analysis]
    │                │
    │                ▼
    │        [secrethelpdesk934752.support.futurevera.thm found]
    │                │
    │                ▼
    │        [HTTP request to secret subdomain]
    │                │
    │                ▼
    │        [302 Redirect to AWS S3]
    │                │
    │                ▼
    └────────► [Flag exposed in Location header]
```

---

## 🛡️ Vulnerability Assessment & Remediation

| # | Vulnerability | Severity | CVSS | Remediation |
|---|--------------|----------|------|-------------|
| 1 | **Subdomain Enumeration Exposure** | 🟠 High | 7.5 | Implement wildcard DNS responses; use DNSSEC; restrict zone transfers |
| 2 | **SSL Certificate SAN Leakage** | 🟠 High | 7.0 | Minimize SAN entries; use wildcard certificates sparingly; audit certificate metadata |
| 3 | **Expired SSL Certificate** | 🟡 Medium | 5.3 | Implement automated certificate renewal (Let's Encrypt, cert-manager) |
| 4 | **AWS S3 Subdomain Takeover** | 🔴 Critical | 9.0 | Remove dangling DNS records; verify S3 bucket existence before CNAME creation; use AWS Route 53 health checks |
| 5 | **Sensitive Data in Redirect URLs** | 🔴 Critical | 9.1 | Never embed secrets in URLs; use POST bodies or secure headers; implement proper session management |

---

## 🎓 Key Takeaways

1. **Certificate SAN Analysis:** SSL certificates often leak hidden infrastructure through Subject Alternative Names — always inspect
2. **Baseline Filtering:** Establish response baselines before fuzzing to eliminate false positives (`-fs` flag in ffuf)
3. **HTTP vs HTTPS:** Protocol differences can reveal distinct behavior — always test both
4. **Header Inspection:** `curl -I` reveals redirect chains and hidden data in HTTP headers
5. **Subdomain Takeover:** Dangling DNS records pointing to deleted cloud resources are a critical attack vector

---

---

## 🚩 Flag

| Flag | Value |
|------|-------|
| **Flag** | `flag{beea0d6edfcee06a59b83fb50ae81b2f}` |

---

> **⚠️ Legal Disclaimer:** This writeup is for educational and research purposes only. Always obtain explicit written authorization before testing systems you do not own.

---

**Author:** Miraç Akkuş (LatenT)  
**Date:** May 2026
```
