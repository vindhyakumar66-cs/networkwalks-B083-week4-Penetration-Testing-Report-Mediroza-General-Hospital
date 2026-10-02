# Mediroza Hospital Web Application Penetration Test Report

[![Severity: Critical](https://img.shields.io/badge/Overall%20Risk-Critical-red.svg)](#risk-ratings)
[![Type: Black-Box Assessment](https://img.shields.io/badge/Assessment-Black--Box%20Pentest-blue.svg)](#scope-and-methodology)
[![Target: Authorized Lab](https://img.shields.io/badge/Scope-Authorized%20Testing%20Environment-green.svg)](#scope-and-methodology)

> **Disclaimer:** This report document was generated as part of a controlled educational assessment. Testing was conducted exclusively within authorized parameters.

---

## 1. Executive Summary

A black-box penetration test was performed against the web infrastructure of **Mediroza General Hospital** (`https://medirozahospital.com`). The objective was to identify security weaknesses, assess exposure risks, and provide practical remediation steps.

An authentication bypass flaw in the patient portal granted unauthorized access to restricted medical documents. Three password-protected patient lab PDFs were retrieved; subsequent hash cracking recovered weak passwords protecting the files. Metadata extracted from one of the documents revealed an unindexed internal path (`/old/`), leading to the discovery of an unprotected full SQL database backup. This database exposed internal hospital employee salaries and shareholder distributions in plain text.

**Overall Risk Level:** **CRITICAL**

---

## 2. Scope & Methodology

### In-Scope
* `https://medirozahospital.com`

### Out of Scope
* Denial of Service (DoS/DDoS)
* Social Engineering / Phishing
* Any host or infrastructure outside the explicit domain

### Tools & Techniques
* **Reconnaissance & HTTP Inspection:** `curl`, Web Browser Developer Tools, `robots.txt` analysis
* **Authentication Testing:** Input handling and SQL syntax error analysis
* **Credential Analysis & Hash Cracking:** Browser-based hash extraction tools, dictionary analysis via `rockyou.txt`
* **Metadata Analysis:** `exiftool`
* **Artifact Extraction & Validation:** `wget`, `file`, `sha256sum`, `grep`, `less`

---

## 3. Summary of Findings

| ID | Finding | Location | Risk Rating | Status |
|---|---|---|---|---|
| SEC-01 | Authentication Bypass on Patient Login | `/patient/login.php` | **Critical** | Confirmed |
| SEC-02 | Unauthenticated Access to Patient Documents | Patient Portal Dashboard | **High** | Confirmed |
| SEC-03 | Weak Document Password Encryption | `patient_report_*.pdf` | **High** | Confirmed |
| SEC-04 | Information Leakage in File Metadata | `patient_report_3.pdf` | **Medium** | Confirmed |
| SEC-05 | Publicly Accessible Directory Listing | `/old/` | **Critical** | Confirmed |
| SEC-06 | Sensitive Internal Data Exposed in Plaintext Backup | `/old/mediroza_db_backup_2019.sql` | **Critical** | Confirmed |

---

## 4. Detailed Vulnerability Breakdown

### SEC-01: Authentication Bypass on Patient Login
* **Severity:** Critical
* **Affected Endpoint:** `/patient/login.php`
* **Description:** Input handling flaws in the login form allow input characters (such as unescaped single quotes) to alter backend SQL queries, resulting in database syntax errors and credential bypass.
* **Impact:** Arbitrary users can bypass portal authentication without valid credentials.

### SEC-02: Direct Access to Medical Records
* **Severity:** High
* **Affected Area:** Patient Portal Dashboard
* **Description:** After authentication bypass, the interface exposes links to download patient lab reports without secondary authorization checks.
* **Impact:** Confidential patient health information (PHI) is exposed to unauthorized users.

### SEC-03: Weak Document Passwords
* **Severity:** High
* **Affected Files:** `patient_report_1.pdf`, `patient_report_2.pdf`, `patient_report_3.pdf`
* **Description:** Passwords securing the PDF files relied on default sequences (`123456`, `password`) and common wordlist strings easily cracked using standard dictionary attacks.
* **Impact:** Document-level encryption provides no defense against unauthorized decryption.

### SEC-04: Metadata Information Leakage
* **Severity:** Medium
* **Affected File:** `patient_report_3.pdf`
* **Description:** Internal administrative notes were left in the PDF comments metadata field:
  > `"DB backup moved to /old before site migration, do not delete"`
* **Impact:** Exposes internal infrastructure layout and forgotten storage paths directly to attackers.

### SEC-05 & SEC-06: Directory Listing & Plaintext Database Backup Exposure
* **Severity:** Critical
* **Affected Endpoint:** `/old/` and `/old/mediroza_db_backup_2019.sql`
* **Description:** The `/old/` directory—referenced in `robots.txt` and file metadata—had directory indexing enabled (HTTP 200). A publicly downloadable database dump (`6,346 bytes`) contained unencrypted database schemas and tables.
* **Impact:** Full compromise of organizational records, exposing employee salaries (`monthly_salary_zar`) and equity distribution records (`shareholders`).

---

## 5. Remediation Roadmap

1. **Implement Prepared Statements:** Use parameterized queries across all database access code to eliminate SQL injection and authentication bypass weaknesses.
2. **Enforce Strict Access Controls:** Verify that requests for patient reports authenticate the caller and ensure authorization is checked against user sessions.
3. **Automate Metadata Stripping:** Sanitize document properties and remove internal operational comments before publishing files externally.
4. **Disable Web Directory Indexing:** Ensure web server configuration blocks directory browsing (e.g., `Options -Indexes` in Apache or appropriate Nginx directives).
5. **Secure Storage for Backups:** Never store database backups inside the public web document root; encrypt backups at rest with role-based access restrictions.
