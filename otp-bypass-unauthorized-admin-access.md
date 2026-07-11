# Critical 2FA/OTP Bypass Leading to Unauthorized Administrative Account Creation

**Category:** Broken Authentication / Business Logic Flaw / Broken Access Control
**Target Type:** Healthcare Platform — Government Digital Identity (National ID Verification) Integration — *Program name anonymized per disclosure policy*

---

## The Story

Some of the scariest bugs aren't complex exploits — they're a single line of trust the backend never should have given the frontend. This was one of those.

The target was a healthcare platform that used a national digital-identity verification gateway as its first authentication factor, followed by a mobile OTP as a second layer before letting anyone register as a "Facility Manager" — a role with full administrative control over medical facility data, practitioner records, and sensitive organizational information. Two layers of verification, on paper. In practice, only one of them actually mattered.

I went through the registration flow normally: verified my identity through the government gateway, got redirected to enter a mobile number and OTP. Instead of typing in a real code, I threw in an arbitrary 4-digit number and caught the request in Burp Suite before it left. The server did exactly what it should — responded with `400 Bad Request`. Invalid OTP, rejected, as expected.

But instead of stopping there, I flipped on response interception and asked myself the obvious question: what does the frontend actually check to decide "verified" or "not verified"? So when the server sent back its `400`, I intercepted the *response* this time — not the request — and rewrote it to `200 OK` with a simple `{"isVerified":true}` body, then forwarded it to the browser.

The frontend didn't blink. It treated my forged response as gospel and walked straight into the next registration step — no re-check, no server-side confirmation that an OTP had ever actually succeeded. Registration completed. And then came the part that turned this from "annoying UI trust issue" into a full-blown critical: the server issued a valid, properly signed JWT token — carrying a user ID, role ID, full permission list, and organization ID — granting genuine "Facility Manager" administrative access.

To make sure this wasn't some fragile client-side illusion that would evaporate on refresh, I logged out and logged back in from a completely different device, with no interception running at all. The account was real. The token was real. The access was real. A permanent, fully privileged administrative account had been created off the back of an OTP that was never actually valid — the server had simply never bothered to check.

What made it worse was a small detail I noticed while testing other flows: the platform's password-reset feature correctly validated usernames server-side and rejected invalid ones properly. So the backend clearly *knew how* to do server-side validation — it just never applied that same discipline to the one flow that mattered most: proving a human actually owns the phone number they claim to control before handing them administrative keys to real medical facility data.

---

## Steps to Reproduce

1. Begin the "Create New Account" flow, integrated with a government digital-identity verification gateway.
2. Complete identity verification through the gateway.
3. On the following "Registration Details" step, enter a mobile number and
   submit an arbitrary/incorrect OTP code, intercepting the request in a
   proxy tool.
4. Observe the server correctly returns `400 Bad Request` for the invalid
   OTP.
5. Enable response interception, and modify the intercepted response to:
6. ```
   HTTP/2 200 OKContent-Type: application/json{"data":{"isVerified":true},"statusCode":200}
   ```
6. Forward the modified response to the browser.
7. Observe the frontend proceeds to complete registration successfully.
8. Inspect issued session cookies/tokens — a valid signed JWT is granted,
   containing user ID, role ID, permission list, and organization ID.
9. Log out, and log back in from a separate device/session with no
   interception active — confirm the account and elevated access persist.

## Proof of Concept

Video PoC demonstrating full registration bypass and resulting dashboard
access, along with original proxy logs showing the legitimate `400` response,
available upon request (withheld from public writeup for disclosure safety).

---

## Impact

- **Secondary Verification Bypass:** Any user who completes the primary
  identity verification step can forge their way past the mandatory OTP
  factor entirely, undermining the platform's two-factor security model.
- **Full Administrative Takeover:** Successful bypass leads directly to a
  persistent, fully privileged "Facility Manager" account — capable of
  viewing, modifying, and deleting sensitive facility, practitioner, and
  organizational data.
- **Unauthorized Privileged JWT Issuance:** A legitimately signed JWT with
  elevated permissions is issued off the back of a forged client response,
  enabling unrestricted authorized-looking API access.
- **Sensitive Data Exposure Risk:** Elevated access exposes organizational
  and potentially personal data spanning government, private, and business
  sector facilities.
- **Mass Abuse Potential:** Because the flaw doesn't depend on any
  legitimate phone number or OTP, it could be scripted to generate unlimited
  fraudulent privileged accounts.

---

## Root Cause

The backend never persists or independently re-verifies OTP validation
state. Instead, the frontend infers "OTP verified" purely from the HTTP
status code of a single response, which is fully attacker-controllable via
response interception. No server-side check occurs before issuing a
privileged session token.

---

## Recommended Remediation

1. **Server-side OTP state enforcement:** The server must track and enforce
   OTP verification state authoritatively — the client must never be the
   source of truth for "verified."
2. **Re-validate before privilege assignment:** Before issuing any
   privileged JWT or completing registration, the server must independently
   confirm that a genuine OTP challenge was passed.
3. **Authorization checks beyond identity verification:** After identity
   verification, the platform must confirm the individual is actually
   authorized to manage the specific facility they're claiming, rather than
   granting management rights by default.
4. **JWT issuance gating:** Tokens carrying elevated roles/permissions
   should only be issued after a fully server-verified authentication
   handshake, never based on frontend-reported state.

---

## Disclosure Status

Reported responsibly through the program's official disclosure channel
(staging environment). Domain, tokens, and identity-verification provider
details redacted in accordance with the program's responsible disclosure
policy.
