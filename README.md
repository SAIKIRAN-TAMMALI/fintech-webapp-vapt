# 🛡️ Web Application VAPT — Simulated FinTech Environment

[![Kali Linux](https://img.shields.io/badge/Kali_Linux-557C94?style=flat-square&logo=kalilinux&logoColor=white)](https://www.kali.org)
[![Burp Suite](https://img.shields.io/badge/Burp_Suite-FF6633?style=flat-square&logo=burpsuite&logoColor=white)](https://portswigger.net)
[![Nmap](https://img.shields.io/badge/Nmap-4682B4?style=flat-square&logoColor=white)](https://nmap.org)
[![OWASP](https://img.shields.io/badge/OWASP_Top_10-000000?style=flat-square&logo=owasp&logoColor=white)](https://owasp.org)
[![VirtualBox](https://img.shields.io/badge/VirtualBox-2496ED?style=flat-square&logo=virtualbox&logoColor=white)](https://www.virtualbox.org)

> **Black-box vulnerability assessment & penetration test of a deliberately vulnerable FinTech web application.**
> Completed as a course-end project for the **Advanced Executive Program in Cybersecurity — IIIT Bangalore**.

---

## 📌 Disclaimer

All testing was performed in a **controlled, isolated lab** built specifically for this academic project. No production systems and no real user data were ever involved. The "flags" are **planted markers** standing in for sensitive data — they are not real secrets. This work is for educational and portfolio purposes only.

---

## 🎯 Objective & Scenario

A FinTech company plans to launch a web application for managing personal finances. Before launch, the app is tested in a simulated form to find vulnerabilities that could expose sensitive data. Hidden flags represent critical information, so no real data is at risk during testing.

**My goal:** act as the ethical hacker — enumerate the target, capture the planted flags, classify each weakness using industry standards (CWE / OWASP Top 10 / CVSS), and hand the development team a remediation plan.

---

## 🧭 Scope

| Item | Detail |
|------|--------|
| **Target** | Single host — simulated web app at `10.0.2.8` |
| **Environment** | Isolated VirtualBox lab (NAT network) with Kali Linux as the attacker box |
| **Test type** | Black-box (no credentials or source code provided up front) |
| **Mandate** | Recon → enumeration → exploitation → capture flags → document findings & fixes |
| **Out of scope** | Anything outside the lab subnet; denial-of-service; destructive testing |

---

## 🧰 Tools & Environment

| Tool | Purpose |
|------|---------|
| **VirtualBox** | Isolated, safe environment to host the vulnerable VM |
| **Kali Linux** | Attacker machine with the pentest toolset |
| **Nmap** | Port scanning & service/version reconnaissance |
| **Gobuster / dirb / dirsearch** | Directory & file brute-forcing (content discovery) |
| **wordlists** | Dictionaries used during enumeration |
| **Burp Suite** | Intercepting and inspecting HTTP requests/responses |
| **Browser "view-source"** | Manual inspection of front-end code & comments |
| **FTP client / wget** | Connecting to and pulling files from the FTP service |

---

## 🗺️ Methodology

I worked through the engagement using the **OWASP Web Security Testing Guide (WSTG)** as a checklist, in four phases:

1. **Reconnaissance** — confirm the target is reachable and on the same subnet; scan ports and services.
2. **Enumeration** — brute-force directories and files to map the application's attack surface.
3. **Exploitation / discovery** — interact with exposed services (FTP, web pages, admin panel, debug pages, HTTP traffic) to capture the planted flags.
4. **Analysis & reporting** — classify each finding (CWE / OWASP / CVSS), assess impact, and recommend remediation.

---

## ⚙️ Lab Setup & Commands

> Full command notes live in [`methodology/commands-and-setup.md`](methodology/commands-and-setup.md). Summary below.

**Environment setup**
```bash
# Imported the vulnerable VM into VirtualBox and set the network to NAT,
# placing attacker (Kali) and target on the same subnet.
ip a                       # verify Kali and target are on the same subnet
sudo su                    # elevate to administrative control
apt update                 # refresh package lists
apt install nmap gobuster dirb wordlists -y   # install enumeration toolset
```

**Reconnaissance & enumeration**
```bash
nmap -A 10.0.2.8                                              # ports, services, versions
gobuster dir -u http://10.0.2.8 -w /usr/share/wordlists/dirb/common.txt
dirsearch -u http://10.0.2.8 -e php,html,js,txt,zip -t 50     # second pass for files
```

Gobuster mapped the application's directories and files — surfacing `robots.txt`, `pages/`, `images/`, `css/`, and `js/`, which guided the rest of the assessment:

![Gobuster directory enumeration output](gobuster_op_.png)

**Exploitation / flag capture**
```bash
ftp 10.0.2.8                                                 # FTP not blocked — test access
wget -m --no-passive ftp://anonymous:anonymous@10.0.2.8      # anonymous login confirmed; mirror files
cat flag1.txt                                                # read the captured flag
# Flags 3 & 4: browser "view-source:" on the blog page and /4dm1n/ admin page
# Flag 5: browse to the exposed /c0nf1g/ phpinfo() page
# Flag 6: inspect the HTTP request/response in Burp Suite Repeater
```

---

## 🚩 Findings Summary

Six flags captured — **every one a form of information disclosure / sensitive-data exposure**, mostly rooted in **security misconfiguration**.

| # | Where found | Vulnerability | CWE | OWASP Top 10 | CVSS (est.) | Flag captured |
|---|-------------|---------------|-----|--------------|-------------|:---:|
| 1 | FTP service | Anonymous FTP exposing files + a private key | CWE-287 / 319 | A05 / A07 | **High (7.5)** | ✅ |
| 2 | `robots.txt` | Sensitive data disclosure via robots.txt | CWE-200 | A05 | **Medium (5.3)** | ✅ |
| 3 | Page source | Secret left in an HTML comment | CWE-615 | A05 | **Medium (5.3)** | ✅ |
| 4 | `/4dm1n/` source | Comment leak on an exposed admin login page | CWE-615 / 200 | A05 / A01 | **Medium (6.5)** | ✅ |
| 5 | `/c0nf1g/` | Exposed `phpinfo()` config dump (EOL PHP 7.4.30) | CWE-200 / 489 | A05 / A06 | **Medium (5.3)** | ✅ |
| 6 | HTTP traffic | Sensitive data sent over cleartext HTTP | CWE-319 | A02 | **Medium (6.1)** | ✅ |

*CVSS scores are estimates produced as a learning exercise; real-world scoring depends on production exposure and data sensitivity.*

---

## 🔍 Detailed Findings

> Full write-up with impact analysis and per-finding remediation: [`reports/vulnerability-assessment.pdf`](reports/vulnerability-assessment.pdf). Captured flag values are shown below as proof of completion (they are planted, simulated values).

### 1 — Anonymous FTP exposing files & a private key · *High*

Connected to FTP with no password, listed and downloaded files, and read `flag1.txt`. The directory also exposed `archive-key.asc` (a PGP key file). FTP also transmits everything in cleartext.

`flag 1 → bd923112d59c477a94c9998379152258`

![Anonymous FTP login and flag1.txt retrieved](flag1.png)
![Anonymous FTP login and flag1.txt retrieved cat flag1.txt revealing the flag](screenshots/Anonymous FTP login and flag1.txt retrieved cat flag1.txt revealing the flag.png)

**Fix:** disable anonymous FTP, switch to SFTP/FTPS, remove sensitive files from served folders, rotate the exposed key.

---

### 2 — Secret in `robots.txt` · *Medium*

Gobuster surfaced `robots.txt`; opening it revealed a flag next to `Disallow: /`. `robots.txt` is a crawler hint, not a security control — anyone can read it.

`flag 2 → 930f04eb6ff0eb864b2157dd2aa048c6`

![robots.txt exposing flag 2](flag2.png)

**Fix:** never store secrets in `robots.txt`; protect sensitive paths with real auth.

---

### 3 — Secret in an HTML comment · *Medium*

`view-source:` on a blog page exposed a flag inside an HTML comment. Anything in client-side source is visible to every visitor.

`flag 3 → b35e7489cf89d4188c85d921a2f79821`

![Flag 3 hidden in an HTML comment in the blog page source](Flag3.png)

**Fix:** strip comments at build time; scan for secrets before release.

---

### 4 — Comment leak on an exposed admin login · *Medium*

Found `/4dm1n/`, viewed source, and recovered another flag from a comment. The obfuscated path name was no protection — it was discovered in seconds.

`flag 4 → 337651bfdec655e803343648eae68ac3`

![Exposed admin login page at /4dm1n/](adminpage.png)
![Flag 4 in the admin page source comment](Flag4.png)

**Fix:** remove the comment; restrict the admin panel by IP/VPN/network segmentation; add rate limiting, lockout, and MFA.

---

### 5 — Exposed `phpinfo()` config dump · *Medium*

`/c0nf1g/` served a full `phpinfo()` page leaking PHP version, OS, paths, and modules. It also revealed **PHP 7.4.30 — end-of-life**, so no more security patches.

`flag 5 → 88615e277ae89ad96a4a365a9a854740`

![phpinfo() page at /c0nf1g/ exposing server configuration and flag 5](Flag5.png)

**Fix:** remove debug pages from production, set `expose_php = Off`, upgrade PHP.

---

### 6 — Sensitive data over cleartext HTTP · *Medium*

Burp Suite Repeater showed a `POST` over plain `HTTP/1.1` (no TLS) with the flag visible in the traffic. Anyone on the network path could read it.

`flag 6 → 0b318db8f0d5b38c6d38ede5e2af71c3`

![Burp Suite Repeater showing flag 6 over cleartext HTTP](Flag6.png)

**Fix:** enforce HTTPS/TLS everywhere, enable HSTS, keep sensitive values out of responses/logs.

---

## 🧠 Key Takeaways

1. **One dominant root cause.** Five of the six issues are security misconfiguration / information leaks — the real fix is a cleaner release process that strips debug pages, comments, and diagnostic endpoints before anything ships, not six one-off patches.
2. **Obscurity is not security.** The disguised folder names (`4dm1n`, `c0nf1g`) didn't slow discovery at all. "Hard to guess" ≠ "protected."
3. **Secrets leaked everywhere** — FTP, comments, `robots.txt`, and live traffic — mirroring how real keys and PII escape. A secret-scanning gate in CI would have caught most of them.
4. **Encryption gaps in two places** — cleartext FTP (data at rest in served folders) and cleartext HTTP (data in transit) — so sensitive data was exposed both on the server and on the wire.

---

## ✅ Remediation Roadmap

1. **Immediate:** disable anonymous FTP, rotate the exposed key, delete the `phpinfo()` page, enforce HTTPS.
2. **Short term:** strip comments and debug leftovers from the build; lock down the admin interface.
3. **Ongoing:** add secret-scanning and config checks to the release pipeline, upgrade the EOL PHP, and re-test to confirm the fixes hold.

---

## 📂 Repository Structure

```
fintech-webapp-vapt/
├── README.md                          ← this file
├── reports/
│   ├── vulnerability-assessment.pdf   ← full VA report (goals, scope, findings, CVSS, fixes)
│   └── flags-evidence.pdf             ← the six captured flags with screenshots
├── methodology/
│   └── commands-and-setup.md          ← lab setup + every command used
└── screenshots/                       ← evidence images per flag
    ├── gobuster_op_.png                ← directory enumeration
    ├── flag-1_i_.png                   ← FTP wget download
    ├── Flag1.png                       ← cat flag1.txt
    ├── Flag2.png                       ← robots.txt
    ├── Flag3.png                       ← blog page source comment
    ├── adminpage.png                   ← /4dm1n/ login page
    ├── Flag4.png                       ← admin source comment
    ├── Flag5.png                       ← phpinfo() dump
    └── Flag6.png                       ← Burp Repeater
```

---

## 👤 Author

**Saikiran Tammali** — Cybersecurity Graduate · SOC / Vulnerability Management

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/saikiran-tammali-cybersec32)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/SAIKIRAN-TAMMALI)

*Course-end project — Advanced Executive Program in Cybersecurity, IIIT Bangalore.*
