# Sensitive Data Exposure via Public Source Code Repository — Hardcoded Admin Credentials & Internal API Disclosure

**Category:** Information Disclosure / Hardcoded Credentials
**Severity:** Critical
**Target Type:** Enterprise Backend System (IVR / Financial Services Integration) — *Program name anonymized per disclosure policy*

---

## Summary

A publicly accessible source code repository was discovered containing the
backend source of an IVR (Interactive Voice Response) system. The exposed
code included hardcoded administrator credentials — multiple privileged
accounts sharing a single password hash — along with disclosure of internal
API endpoint names responsible for processing financial and customer data.

Key issues:
- Multiple administrator accounts (`admin*****@*****.com`) sharing a single
  hardcoded password hash within a public seed-data file
- Internal API endpoint names related to card data, transaction history, and
  customer records exposed in source
- Both staging and production endpoint references present in the codebase

---

## Proof of Concept

- **Hardcoded credentials:** A seed-data file contained several administrator
  accounts, all secured by the **same hardcoded password hash** — meaning a
  single successful crack or leak compromises every admin account at once.
- **Internal API exposure:** The repository revealed the existence and naming
  of internal endpoints tied to cardholder data retrieval, transaction
  history, and customer PII, along with company-internal data seed files.

*(Screenshots and full technical detail — including exact credentials,
hashes, and endpoint names — withheld from this public writeup and shared
only with the affected program through the official disclosure channel.)*


---

## Impact

- **Full Administrative Account Takeover:** A shared password hash across
  multiple privileged accounts drastically increases the odds that cracking
  or leaking one credential compromises all admin access — including system
  configuration, user management, and backend control.
- **Large-Scale Financial & Personal Data Exposure Risk:** Disclosure of
  endpoint names tied to card data and transaction history creates a direct
  roadmap for attackers to target the systems most likely to hold sensitive
  financial and personally identifiable information.
- **Security Control Bypass:** Knowledge of internal architecture allows
  attackers to bypass perimeter protections (e.g., WAF) and go straight for
  logic-level flaws such as IDOR against the now-mapped endpoints.
- **Elevated Regulatory & Reputational Risk:** Given the sensitivity of
  financial data potentially in scope, this class of exposure carries
  significant compliance, financial, and reputational consequences for the
  affected organization.

---

## Root Cause

Sensitive configuration and seed data — including credentials and internal
API definitions — were committed directly into source code and pushed to a
publicly accessible repository, rather than being managed through a secrets
manager or excluded via `.gitignore`.

---

## Recommended Remediation

1. **Rotate all exposed credentials immediately** and invalidate the shared
   hash across all affected accounts.
2. **Adopt a secrets management solution** (e.g., Azure Key Vault, AWS
   Secrets Manager, HashiCorp Vault) instead of hardcoding credentials or
   seed data in source.
3. **Enforce unique, strong credentials per account** — never share password
   hashes across privileged accounts.
4. **Enable repository secret scanning** (e.g., GitHub Secret Scanning /
   push protection) to catch accidental credential commits before they go
   public.
5. **Audit repository visibility settings** and review git history for any
   previously exposed secrets, rotating them even if the repository is later
   made private.
6. **Apply strong authentication (OAuth2 + MFA)** to sensitive backend API
   routes regardless of whether they're expected to remain undiscovered.

---

## Disclosure Status

Reported responsibly through the program's official disclosure channel.
All sensitive identifiers — credentials, hashes, endpoint names, and
screenshots — withheld from this public writeup in accordance with
responsible disclosure practices.
