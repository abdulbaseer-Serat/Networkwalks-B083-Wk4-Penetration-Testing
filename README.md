<div align="center">

# 🏥 Mediroza General Hospital
### Web Application Penetration Test

**Week 4 Penetration Testing Internship · Batch B082**

*Confidential Security Assessment — Authorized Educational Engagement*

[![Status](https://img.shields.io/badge/Assessment-Complete-brightgreen?style=for-the-badge)](#-milestone-results)
[![Risk](https://img.shields.io/badge/Overall%20Risk-Critical-red?style=for-the-badge)](#-overall-risk-assessment)
[![Type](https://img.shields.io/badge/Type-Black--Box-blue?style=for-the-badge)](#-scope--methodology)
[![Scope](https://img.shields.io/badge/Scope-Authorized-success?style=for-the-badge)](#-assessment-disclaimer)

</div>

---

## 📋 Table of Contents

- [Executive Summary](#-executive-summary)
- [Scope & Methodology](#-scope--methodology)
- [Tools Used](#-tools-used)
- [Attack Surface Discovery](#-attack-surface-discovery)
- [Findings](#-findings)
- [Milestone Results](#-milestone-results)
- [Risk Rating Summary](#-risk-rating-summary)
- [Overall Risk Assessment](#-overall-risk-assessment)
- [Recommendations & Remediation](#-recommendations--remediation)
- [Evidence Handling & Privacy](#-evidence-handling--privacy)
- [Lessons Learned](#-lessons-learned)
- [Final Deliverables](#-final-deliverables)
- [Disclaimer](#-assessment-disclaimer)

---

## 1. 🎯 Executive Summary

An authorized **black-box penetration test** was conducted on Mediroza General Hospital's web infrastructure. The assessment identified seven security vulnerabilities ranging from Medium to Critical severity.

The most critical issue was a SQL injection vulnerability in the patient portal login page, allowing authentication bypass and unauthorized access to confidential patient records. Further exploitation exposed sensitive data, including patient lab reports, employee salary information, and shareholder details through a publicly accessible database backup.

Overall, the hospital's security posture is poor, with multiple vulnerabilities that could enable attackers to access sensitive medical, financial, and corporate information without authentication. Immediate remediation is recommended to reduce the risk of data breaches and unauthorized access.A controlled **black-box penetration test** was conducted against Mediroza General Hospital's web infrastructure as part of a Week 4 penetration testing internship assignment.

| | |
|---|---|
| 🌐 **Target** | `https://medirozahospital.com` |
| 🏢 **Client** | Mediroza General Hospital |
| 🧪 **Assessment Type** | Black-box Web Application Penetration Test |
| ✅ **Authorization** | Explicitly authorized, educational scope only |
| 🚫 **Excluded** | Social engineering, denial-of-service |

### Objectives

1. Gain access to the restricted patient portal.
2. Retrieve three confidential patient laboratory reports.
3. Crack the encryption protecting all three reports.
4. Identify exposed employee salary information.
5. Identify exposed shareholder information.
6. Document vulnerabilities, evidence, impact, and remediation.

### Key Findings

| ID | Finding | Severity |
|---|---|:---:|
| MED-01 | SQL Injection → Patient Portal Authentication Bypass | 🔴 **Critical** |
| MED-02 | Publicly Accessible Database Backup (HR & Shareholder Data) | 🔴 **Critical** |
| MED-03 | Weak PDF Passwords / Recoverable Encryption | 🟠 **High** |
| MED-04 | SQL Error Disclosure / Unsafe SQL Construction | 🟡 **Medium** |
| MED-05 | Username Enumeration via Auth Responses | 🟢 **Low** |
| MED-06 | Directory Listing / Information Disclosure | 🟢 **Low** |

> **Bottom line:** A username-field SQL injection bypassed patient login entirely, exposing three encrypted lab reports. All three were cracked offline with Hashcat. Separately, a forgotten backup at `/old/` leaked internal staff salary and shareholder records to the open internet — no authentication required.

---


## 2. 🔍 Scope & Methodology

### 2.1 Scope

The penetration test was conducted against the following target:
- **Target:** https://medirozahospital.com


**Out of scope:** - Social engineering attacks - Denial-of-Service (DoS) attacks - Any systems or domains outside the agreed scope

### 2.2 Methodology

The assessment followed a structured black-box penetration testing approach consisting of four phases:

1. **Reconnaissance** – Gathered publicly available information about the target.
2. **Vulnerability Identification** – Analyzed the application to identify security weaknesses.
3. **Exploitation** – Performed controlled exploitation to verify the impact of discovered vulnerabilities.
4. **Documentation** – Recorded findings, evidence, and remediation recommendations.

### Testing Phases

```
Phase 1  Reconnaissance            → headers, sitemap, robots.txt, directory discovery
Phase 2  Authentication Analysis   → input validation, SQLi indicators, enumeration
Phase 3  Restricted Area Access    → SQL injection auth bypass → patient portal
Phase 4  Data Extraction           → 3 encrypted lab report PDFs retrieved
Phase 5  Offline Password Recovery → pdf2john + hashcat (mode 10500) → all 3 cracked
Phase 6  Further Exposure Analysis → /old/ backup discovery → HR & shareholder leak
```

---

## 🛠 Tools Used

| Tool | Purpose |
|---|---|
| `curl` | HTTP requests, header & endpoint analysis |
| Browser developer tools | for inspecting page source and login form behaviour - Authentication testing |
| Networkwalks Hash Calculator | for extracting password hashes from PDF files|
| Networkwalks Password Cracker | for cracking PDF password hashes using wordlists|
| `qpdf` | for decrypting password-protected PDF files after cracking |
| `exiftool` |  for reading hidden metadata from PDF files |
| `pdf2john` | PDF password-hash extraction |
| `wget` | Offline password recovery |
| `pdftotext` | for downloading files from the web server |
| Linux CLI utilities | Evidence collection & file analysis |
| ChatGPT | for converting raw SQL data into readable tables|

---

## 3. Findings and Proof of Exploitation
### 3.1 Summary Table 

| # | Vulnerability | Location | Risk |
|---|---------------|----------|------|
| 1 | Username enumeration on login page | `patient/login.php` | Medium |
| 2 | SQL injection login bypass | `patient/login.php` | Critical |
| 3 | Encrypted PDFs accessible after login bypass | `patient/reports/` | High |
| 4 | Weak PDF passwords crackable with a wordlist | `patient_report_*.pdf` | High |
| 5 | Sensitive metadata left in patient PDF files | `patient_report_3.pdf` | Medium |
| 6 | Forgotten backup folder with directory listing enabled | `old/` | Critical |
| 7 | Confidential staff salaries and shareholder data in plain text | `old/mediroza_db_backup_2019.sql` | Critical |



## 🕵️ 4 Findings and Proof of Exploitation

<details open>

   <summary><strong>Finding-01 · Username Enumeration on Login Page</strong> — 🟡 Medium</summary>

**Location:** `patient/login.php`

#### Description
The login page returned different error messages for invalid usernames and incorrect passwords, allowing attackers to determine whether a username exists on the system.
 
#### Evidence
Username not found
```text
Username: bob
Password: test123
```
<img width="948" height="458" alt="image" src="https://github.com/user-attachments/assets/29e7002f-e00e-4d3f-9f2b-0a134084bf78" />

</details>


<details open>
<summary><strong>Finding-02 · SQL Injection Login Bypass </strong> — 🔴 Critical</summary>

- **Location:** `patient/login.php`
 
#### Description
The login form was vulnerable to SQL Injection due to unsanitized user input. This allowed authentication controls to be bypassed and enabled unauthorized access to the patient portal.
 
#### Evidence
 
A SQL error was returned when a single quote (`'`) was entered, indicating that user input was being processed directly by a database query.
 
Example test input:
 
```text
Username: admin'
Password: test123
```
<img width="959" height="500" alt="image" src="https://github.com/user-attachments/assets/940b2174-38a9-4b34-8995-df72e0ce8545" />

The error confirmed that the parameter was injectable. By using a SQL Injection payload, authentication was bypassed and access to the portal was obtained without valid credentials. The application was building its SQL query like this behind the scenes. ```text SELECT * FROM users WHERE username='admin'' AND password='test123' ```

After an unescaped extra quote broke the query and triggered a database error, I crafted the classic SQL injection bypass payload.

```text
Username: admin'--
Password: anything
```
The `--` comments out everything after it in SQL, so the query becomes. ```text SELECT * FROM users WHERE username='admin' ```

The password check is ignored completely. The query returns the admin row and I am logged in. 

**Impact** An attacker could gain unauthorized access to sensitive patient information and other protected resources.

</details>



<details open>
<summary><strong>Finding-03 · Confidential PDFs Accessible After Login Bypass</strong> — 🟠 High</summary>

**Location:** `patient/reports/`

#### Description
After gaining unauthorized access to the portal through SQL injection, there are three patient lab report PDFs  available for download. These are confidential medical documents that should only be accessible to the named patients and their doctors.

#### Evidence
The portal exposed multiple downloadable PDF files:

```text
patient_report_1.pdf
patient_report_2.pdf
patient_report_3.pdf
```
<img width="1910" height="998" alt="Screenshot 2026-09-29 081823" src="https://github.com/user-attachments/assets/7c96a57f-540c-49e5-bce9-de3c7131d432" />

</details>


<details open >
<summary><strong>Finding-04 · Weak PDF Passwords Crackable with a Wordlist </strong> — 🟠 High</summary>
#### Description
   All three PDFs were password protected. However the passwords were weak and appeared in commonly available password wordlists, making them trivial to crack using automated tools. Using simple passwords on sensitive medical documents does not provide meaningful protection.

**Steps Taken :** 
I used the Networkwalks Hash Calculator to extract a crackable hash from each PDF, then ran each hash through the Networkwalks Password Cracker. Reports 1, 2, and 2 cracked immediately using the built-in 100 word default wordlist. 

<img width="1916" height="1030" alt="Screenshot 2026-09-29 092035" src="https://github.com/user-attachments/assets/03ff83b0-21f5-4151-89e2-a4787f978e82" />

<img width="1915" height="1033" alt="Screenshot 2026-09-29 092223" src="https://github.com/user-attachments/assets/209c8bc9-735c-4d55-b0cd-5f6b1a976d0d" />

Report 3 did not crack with the built-in list. I switched to a larger wordlist (JTR default password list) and ran the attack again.

<img width="1914" height="1034" alt="Screenshot 2026-09-29 093253" src="https://github.com/user-attachments/assets/920cd20a-49c2-49d5-b6bd-579de271ad64" />

**OUTPUT**
 <img width="1903" height="699" alt="image" src="https://github.com/user-attachments/assets/f72147cc-e78f-48d3-885f-8e852ce1cb57" />

</details>



<details Open>
<summary><strong>Finding-05 · 5. Sensitive Metadata in Patient PDF Files</strong> — 🟡 Medium</summary>

**Location:** `patient_report_3.pdf`
 
#### Description
A patient PDF file contained sensitive metadata that was not visible during normal viewing but could be extracted using metadata analysis tools. The metadata revealed internal information about the server environment and disclosed the location of a database backup file.
 
#### Evidence
Metadata analysis of a patient report uncovered internal comments referencing a backup file stored on the server.
</details>



<details Open>
<summary><strong>Finding-06 · Forgotten Backup Folder with Directory Listing Enabled</strong> — 🔴 Critical</summary>

**Location:** `old/`
 
#### Description
A publicly accessible backup directory had directory listing enabled, exposing sensitive files to anyone who visited the URL. The directory contained a database backup file that should not have been accessible from the internet
 
#### Evidence
During reconnaissance, a backup directory was identified and found to allow directory listing. The directory exposed a database backup file:
 
```text
mediroza_db_backup_2019.sql
```
</details>



<details Open>
<summary><strong>Finding-07 · Confidential Staff and Shareholder Data Exposed</strong> — 🔴 Critical</summary>

   **Location:** `mediroza_db_backup_2019.sql`
 
#### Description
A publicly accessible database backup contained highly sensitive information, including employee records, salary data, and shareholder information. The data was stored in plain text and could be accessed without authentication.
 
#### Evidence
Analysis of the exposed database backup revealed:
 
- Employee names, job titles, contact details, and salary information.
- Shareholder names and ownership details.
- Internal organizational information that should not be publicly accessible.
 
#### Impact
An attacker could access confidential financial and personal data, leading to privacy violations, financial fraud risks, reputational damage, and regulatory compliance issues.
 
#### Recommendation
- Remove sensitive backup files from publicly accessible locations.
- Encrypt backups containing confidential information.
- Restrict access using proper authentication and authorization controls.
- Implement data classification and retention policies.
- Perform regular audits to identify exposed sensitive data.
</details>
---

## ✅ Milestone Results

| Milestone | Achievement | Status |
|-----------|-------------|----------|
| **M1** | Authentication bypass achieved and patient reports accessed | ✅ |
| **M2** | Protected PDFs accessed and analyzed | ✅ |
| **M3** | Sensitive employee and shareholder data exposure confirmed | ✅ |
| **M4** | Penetration testing report completed and documented | ✅ |

### Overall Outcome
✅ Successfully demonstrated a complete attack chain from initial access to sensitive data exposure, highlighting multiple critical security weaknesses within the target environment.

```
M1 ████████████████████ 100% ✅
M2 ████████████████████ 100% ✅
M3 ████████████████████ 100% ✅
M4 ████████████████████ 100% ✅
```

---

## 📊 Risk Rating Summary

| Finding | Severity | Primary Impact |
|----------|:---------:|----------------|
| SQL Injection Authentication Bypass | 🔴 Critical | Unauthorized access to the patient portal and sensitive records |
| Exposed Database Backup | 🔴 Critical | Disclosure of employee and shareholder information |
| Confidential PDFs Accessible After Login Bypass | 🟠 High | Unauthorized access to patient medical reports |
| Weak PDF Password Protection | 🟠 High | Exposure of protected medical documents |
| Sensitive Metadata in PDF Files | 🟡 Medium | Information leakage aiding further attacks |
| Username Enumeration | 🟡 Medium | Enables discovery of valid user accounts |
| Directory Listing Enabled | 🔴 Critical | Public access to sensitive backup files and resources |

### Overall Risk Rating: 🔴 Critical
Multiple critical vulnerabilities enabled a complete attack path from initial access to the disclosure of sensitive patient, employee, and shareholder data.

---

## 🔗 Full Attack Chain Summary

1. **Reconnaissance** revealed hidden directories through `robots.txt`.
2. **Username enumeration** identified a valid account on the patient portal.
3. **SQL Injection** in the login page enabled authentication bypass.
4. Unauthorized access exposed **three confidential patient PDF reports**.
5. Weak PDF passwords allowed access to protected report contents.
6. PDF metadata disclosed information about an internal backup location.
7. A publicly accessible `/old/` directory exposed a database backup file.
8. The backup contained sensitive **employee salary**, **personal**, and **shareholder** information.


---

## 🔧 Recommendations & Remediation

### Priority 1 — Immediate
- Fix SQL injection with parameterized queries throughout
- Remove `/old/mediroza_db_backup_2019.sql` from the public web root
- Disable directory indexing site-wide
- Investigate whether exposed data was accessed by unauthorized parties

### Priority 2 — High
- Strengthen patient authentication (secure hashing, MFA, rate limiting, lockout, CSRF protection)
- Replace static PDF passwords with strong secrets + application-level authorization
- Remove verbose database error disclosure from production

### Priority 3 — Medium / Low
- Generic authentication error messages (prevent enumeration)
- Reduce technology/version disclosure (`X-Powered-By`, `Server`, CMS version)
- Implement security monitoring (failed logins, SQLi patterns, backup access)
- Store backups outside the web root, encrypted, access-restricted, periodically audited

---



## 💡 Lessons Learned

| # | Takeaway |
|---|---|
| 1 | Authentication must never trust client-supplied input |
| 2 | Verbose error messages are attacker intelligence, not just noise |
| 3 | `robots.txt` is not an access-control mechanism |
| 4 | Backups and legacy files are part of the attack surface |
| 5 | Encryption doesn't compensate for a weak password |
| 6 | Low-severity findings can chain into critical impact |

---

## 📦 Final Deliverables

| Deliverable | Status |
|---|:---:|
| Target reconnaissance | ✅ |
| Authentication analysis & bypass | ✅ |
| Patient portal access | ✅ |
| Three patient PDFs retrieved | ✅ |
| PDF hashes extracted & passwords recovered | ✅ |
| Three PDFs decrypted & verified | ✅ |
| Staff salary exposure identified | ✅ |
| Shareholder exposure identified | ✅ |
| Evidence collected (private) | ✅ |
| Risk ratings assigned | ✅ |
| Remediation recommendations | ✅ |
| Professional report | ✅ |

---

## 📄 Assessment Disclaimer

This penetration test was conducted as part of an **authorized educational security assessment**. The target was explicitly authorized for testing, and all activity was restricted to the agreed scope. No social engineering or denial-of-service testing was performed. All sensitive information has been redacted from this public report. This methodology and evidence are provided for authorized security assessment and educational purposes only.

---

<div align="center">

**Tags:** `penetration-testing` `web-security` `cybersecurity` `ethical-hacking` `black-box-pentest` `sql-injection` `authentication-bypass` `information-disclosure` `password-cracking` `hashcat` `pdf-security` `security-assessment` `vulnerability-assessment`

*Week 4 Penetration Testing Internship — Batch B083 - Abdulbasir Serat*

</div>
