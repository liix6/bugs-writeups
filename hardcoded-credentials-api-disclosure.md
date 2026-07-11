# Sensitive Data Exposure via Public Source Code Repository — Hardcoded Admin Credentials & Internal API Disclosure

**Category:** Information Disclosure / Hardcoded Credentials
**Severity:** Critical
**Target Type:** Enterprise Backend System (IVR / Financial Services Integration) — *Program name anonymized per disclosure policy*

---

## The vulnerability

Sometimes the most damaging vulnerabilities aren't in the application itself — they're sitting in plain sight on GitHub, waiting for anyone curious enough to search for them.

I was doing recon on a company's digital footprint, and instead of poking at the live application, I decided to dork around GitHub for anything related to their internal projects. It's a habit — before touching endpoints, check what's already leaked. And this time, it paid off fast: a public repository containing the backend source code for an IVR (Interactive Voice Response) system.

I started going through the files methodically, not expecting much — most of these repos are just boilerplate or abandoned test projects. But then I hit a seed-data file used to initialize the database with default accounts. Inside it were several administrator accounts, all with emails following the same pattern (`admin*****@*****.com`). That alone wasn't alarming. What was alarming is what came next: every single one of these admin accounts shared **the exact same password hash**.

That's the kind of mistake that turns a single leaked or cracked credential into a master key. One hash, one crack attempt, and every admin account in the system falls at once — not just one.

I kept digging through the codebase and found something worse: the internal API layer was fully mapped out in the source — endpoint names tied directly to cardholder data retrieval, transaction history, and customer personal records. Not obfuscated, not abstracted — just sitting there in the code, staging and production references included.

At that point it wasn't really about "finding a vulnerability" anymore. It was realizing that anyone with basic GitHub search skills — no exploitation required — could walk away with a complete architectural blueprint of a system handling real financial data, plus a shared credential that could unlock administrative access to all of it.

---

## Proof of Concept

- **Hardcoded credentials:** Multiple admin accounts (`admin*****@*****.com`)
  discovered in a public seed-data file, all secured by a **single shared
  password hash**.
- **Internal API exposure:** Endpoint names tied to card data retrieval,
  transaction history, and customer PII were present in source, across both
  staging and production references.

*(Exact credentials, hashes, endpoint names, and screenshots withheld from
this public writeup — shared only with the affected program through the
official disclosure channel.)*

---

## Impact

- **Full Administrative Account Takeover:** A single shared password hash
  across multiple privileged accounts means cracking or leaking one
  credential compromises every admin account simultaneously — full control
  over configuration, user management, and backend services.
- **Large-Scale Financial & Personal Data Exposure Risk:** Disclosed endpoint
  names give attackers a direct roadmap to the systems most likely to hold
  sensitive financial and personally identifiable data.
- **Security Control Bypass:** Full knowledge of internal architecture lets
  an attacker skip past perimeter defenses like a WAF and go straight for
  logic-level flaws (e.g., IDOR) against already-mapped endpoints.
- **Elevated Regulatory & Reputational Risk:** Given the likely presence of
  financial data in scope, this exposure carries significant compliance,
  financial, and reputational consequences for the affected organization.

---

## Root Cause

Sensitive configuration and seed data — including credentials and internal
API definitions — were committed directly into source code and pushed to a
publicly accessible repository, instead of being managed through a proper
secrets manager or excluded from version control entirely.

---

## Recommended Remediation

1. **Rotate all exposed credentials immediately** and invalidate the shared
   hash across every affected account.
2. **Adopt a secrets management solution** (e.g., Azure Key Vault, AWS
   Secrets Manager, HashiCorp Vault) instead of hardcoding credentials or
   seed data in source.
3. **Enforce unique, strong credentials per account** — never reuse password
   hashes across privileged accounts.
4. **Enable repository secret scanning** (e.g., GitHub Secret Scanning /
   push protection) to catch accidental credential commits before they go
   public.
5. **Audit repository visibility and git history** for any previously
   exposed secrets, rotating them even if the repo is later made private.
6. **Apply strong authentication (OAuth2 + MFA)** to sensitive backend API
   routes regardless of expected discoverability.

---

## Disclosure Status

Reported responsibly through the program's official disclosure channel. All
sensitive identifiers — credentials, hashes, endpoint names, and screenshots
— withheld from this public writeup in accordance with responsible
disclosure practices.
