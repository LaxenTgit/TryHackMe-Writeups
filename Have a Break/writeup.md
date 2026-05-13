# 🕵️ Have a Break — Digital Forensics Investigation

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Category](https://img.shields.io/badge/Category-Digital%20Forensics%20|%20OSINT%20|%20Incident%20Response-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Author:** Miraç Akkuş (LatenT)  
> **Date:** May 2026  
> **Room:** [Have a Break](https://tryhackme.com/room/haveabreak)

---

## 📋 Case Overview

A Czech logistics company suspects internal data leakage. Provided evidence includes:
- Anonymous email (`exhibit_a.eml`)
- Dash cam footage (`exhibit_b.png`)
- Route planning system logs (`access_log.csv`)
- Employee database (`employees.csv`)
- Internal memo (`ecta_memo.pdf`)
- Communication export (`comms_export.txt`)

**Objective:** Identify the VPN service, location, timestamps, and the culprit behind the leak.

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| EML Parser | Email header analysis |
| ipinfo.io / bgp.tools | IP lookup and ASN query |
| Google Maps | Petrol station geolocation |
| Epieos | Free OSINT and email pivot |
| CSV Analysis | Log and employee data correlation |

---

## 🔍 Question 1: Which VPN service was used?

### Email Header Analysis

Opened `exhibit_a.eml`. Sender: `notmyname2847@gmail.com`, recipient: `redakce@novinybrno.cz` (Czech news outlet).

**Critical Header:**
```
Received: from [193.32.249.132] ([193.32.249.132])
        by smtp.gmail.com with ESMTPSA id
        4fb4d7f45d1cf-66e02d37620sm407278a12.2.2026.03.27.23.14.55
```

Subsequent `Received:` hops are Google infrastructure (`209.85.220.41`, etc.) — irrelevant for origin identification.

### IP Investigation

| Attribute | Value |
|-----------|-------|
| IP | `193.32.249.132` |
| ASN | AS39351 |
| Organisation | 31173 Services AB |
| Location | Amsterdam, Netherlands |
| Flags | VPN exit node detected |

31173 Services is a Swedish hosting provider with documented partnership with **Mullvad VPN**. The `193.32.249.0/24` subnet is part of Mullvad's Amsterdam server infrastructure.

**Answer:** `Mullvad VPN`

---

## 🗺️ Question 2: Petrol station full address?

### Initial Approach

Used Orlen's official site (`orlen.cz/stanice`) — more reliable than Google Maps alone. Filtered for **LPG-compatible stations** based on dash cam evidence.

### Dead End

Searched D1 highway for LPG Orlen stations, but none matched dash cam footage — specifically missing **Brno and Olomouc** road signs.

### Breakthrough

Analyzed `ecta_memo.pdf`:

> *"March 2026, a dashcam SD card was recovered from a vehicle stopped for an unrelated traffic matter near Hulín."*

### Location Verification

Searched Google Maps for Orlen stations in **Hulín region**. Only one matched all criteria:

| Attribute | Value |
|-----------|-------|
| Brand | Orlen |
| Location | Hulín, Czech Republic |
| Features | LPG available |
| Road Signage | Brno / Olomouc directions confirmed |

**Answer:** `Hulín Orlen station address`

---

## ⏰ Question 3: Suspicious action timestamp?

### Log Analysis

Examined `access_log.csv`, filtered for **March 25, 2026**.

### Anomaly Detection

| Timestamp | User | Action | File |
|-----------|------|--------|------|
| 22:14:09 | BR-0291 | **EXPORT** | Sensitive route PDF |
| [Various] | [Other users] | VIEW / EDIT | [Various files] |

**Red Flags:**
- **EXPORT action** — all others only VIEW/EDIT
- **After-hours** — 22:14 unusual for legitimate business
- **Preceding failed auth** — BR-0291 failed auth on March 24th (reconnaissance)
- **Possible admin intervention** — restrictions applied March 27th

**Answer:** `22:14:09`

---

## 👤 Question 4: Whistleblower employee ID?

### Temporal Correlation

Cross-referenced `access_log.csv` with `employees.csv`:

| Timestamp | User | Role | Activity |
|-----------|------|------|----------|
| 23:41 (Mar 25) | BR-0312 | Dispatch Operator | System access |
| 23:12 (Mar 24) | BR-0312 | Dispatch Operator | Late-night activity |

### Behavioral Profile

- BR-0312 is a **Dispatch Operator** with regular late-shift patterns
- Active immediately after BR-0291's suspicious EXPORT at 22:14
- Anonymous email stated: *"I do not know who to trust inside the company right now"*

This suggests BR-0312 witnessed the anomaly but, as non-IT personnel, chose external whistleblowing over internal reporting.

**Answer:** `BR-0312`

---

## 🎯 Question 5: Culprit employee ID?

Based on Question 3 findings, the EXPORT action at 22:14:09 on March 25th represents the data exfiltration event.

**Answer:** `BR-0291`

---

## 🔎 Question 6: Culprit's full name?

### OSINT Pivot

No full names in provided files. External intelligence required.

### Email Discovery

From `comms_export.txt`, identified suspicious email: `kraliknovak09@gmail.com`

### Tool Evaluation

| Platform | Result |
|----------|--------|
| TruePeopleSearch | Requires payment — rejected |
| BeenVerified | Requires payment — rejected |
| **Epieos** | **Free, comprehensive results** |

### Epieos Analysis

**Input:** `kraliknovak09@gmail.com`

**Results:** Associated Google Maps profile link with location history and user-generated content.

### Google Maps Verification

Followed Epieos link. Profile contained:
- Location check-ins
- Public reviews
- Associated business listings

**Extracted Name:** Radovan Blšťák

**Answer:** `Radovan Blšťák`

---

## 📊 Investigation Summary

| Question | Answer | Technique |
|----------|--------|-----------|
| VPN Service | Mullvad VPN | Email header analysis, ASN lookup |
| Petrol Station | Hulín Orlen | PDF analysis, Google Maps verification |
| Suspicious Timestamp | 22:14:09 | CSV log analysis, anomaly detection |
| Whistleblower ID | BR-0312 | Temporal correlation, behavioral analysis |
| Culprit ID | BR-0291 | Action attribution from logs |
| Culprit Name | Radovan Blšťák | OSINT via Epieos, Google Maps pivot |

---

## 🛡️ Remediation & Recommendations

| # | Vulnerability | Severity | Remediation |
|---|--------------|----------|-------------|
| 1 | Email Header Leakage | 🟠 High | Strip client IP from outbound emails; use corporate VPN |
| 2 | After-Hours EXPORT | 🔴 Critical | Time-based access controls; dual authorization |
| 3 | Failed Auth Monitoring | 🟠 High | Alert on repeated failed authentications; account lockout |
| 4 | OSINT Exposure | 🟠 High | Employee social media training; privacy settings audit |
| 5 | Log Retention | 🟡 Medium | Extend log retention; implement SIEM correlation |

---

## 🎓 Key Takeaways

1. **Email headers don't lie** — The first `Received:` hop after client submission reveals true origin IP
2. **Official sources > third-party** — Orlen's own station finder proved more reliable than Google Maps
3. **Contextual documents matter** — `ecta_memo.pdf` provided the geographic pivot when visual evidence was insufficient
4. **Temporal correlation is powerful** — BR-0312's post-incident access pattern revealed whistleblower identity
5. **Free OSINT tools exist** — Epieos outperformed paid alternatives for this investigation
6. **Payment walls are not dead ends** — Free alternatives often provide equivalent or superior results

---

> *"I do not know who to trust inside the company right now."*  
> — BR-0312, Dispatch Operator, whistleblower

---

**Author:** Miraç Akkuş (LatenT)  
**Date:** Mayıs 2026
