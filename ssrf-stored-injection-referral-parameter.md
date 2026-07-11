# Blind SSRF via Stored Injection in a Chat Widget's Metadata Field

**Category:** SSRF (Server-Side Request Forgery) — Blind, Stored/Asynchronous
**Severity:** High
**CVSS:** 3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N
**Target Type:** Third-party live chat integration (SaaS platform) — *Program name anonymized per disclosure policy*

---

## The Story

I was testing a chat widget embedded on a web platform — the kind of "talk to us" bubble you see in the corner of almost every site nowadays. Like any tester, my first instinct was to poke at the obvious input: the message box itself. No surprise there — the `query` field was well sanitized. Anything resembling a payload got stripped or blocked before it even left the browser.

But chat widgets don't just send your message. They send *metadata* along with it — session IDs, page IDs, device info, referral source. That metadata usually gets ignored by testers because it "isn't user input" in the traditional sense. That assumption is exactly what I wanted to test.

I intercepted a normal chat request in Burp Suite and looked at the JSON body. Buried inside was a `referral` field — a stringified JSON blob containing things like `source`, `pageId`, `DeviceType`. Unlike `query`, nothing here seemed to be validated server-side.

So I tried injecting a payload into `source` and `DeviceType` — HTML-breakout style strings pointing to a Burp Collaborator domain. I sent it, got a 200 OK, and moved on to the next test... until a few minutes later, my Collaborator client lit up. HTTP and DNS hits, straight from the target's infrastructure, hitting the exact paths I'd injected (`/source`, `/device`).

That was the moment it clicked: the `referral` field wasn't just being stored — it was being *parsed and processed asynchronously* by an internal service, most likely a link-preview or metadata-enrichment bot running somewhere behind the scenes. And it was fetching whatever URL I told it to.

To confirm this wasn't a one-off fluke, I pushed further — injecting a fake internal-looking hostname as a subdomain of my Collaborator domain. The backend dutifully tried to resolve and connect to it. Blind SSRF, confirmed.

As a bonus, the outbound requests carried a `User-Agent: TelegramBot` header — an accidental fingerprint revealing that the backend's link-processing logic is built on a Telegram-integrated bot framework. Not something an attacker should need to know, but now they do.

---

## Steps to Reproduce

1. Open the chat widget and send a normal message to trigger a baseline request.
2. Intercept the request in Burp Suite. The body resembles:

```json
   {
     "referral": "[{\"source\":\"web\"},{\"pageId\":\"<id>\"},{\"senderId\":\"<id>\"},{\"DeviceType\":\"Chrome,Computer\"}]",
     "sessionId": "<id>",
     "query": "hello",
     "storyId": "<id>"
   }
```

3. Modify the `source` and `DeviceType` values inside the `referral` string to break out of context and inject a remote URL pointing to a Collaborator (or similar OOB) domain.
4. Send the modified request.
5. Wait 2–5 minutes. Out-of-band HTTP/DNS interactions arrive at the attacker-controlled listener, confirming server-side fetch behavior.
6. To confirm internal network reach, inject an internal-sounding hostname as a subdomain of the listener domain and observe the resolution/connection attempt.

## Proof of Concept

Full video PoC and request/response captures available upon request (withheld from public writeup for disclosure safety).

---

## Impact

**1. Security control bypass**
The primary `query` field is sanitized, but the `referral` metadata is not — giving an attacker a side door around front-end and WAF filtering straight into backend processing logic.

**2. Internal network reconnaissance**
Confirmed out-of-band DNS/HTTP interactions show the backend will resolve and connect to arbitrary attacker-supplied hosts, which could be abused to scan internal network ranges and probe non-public services.

**3. Technology fingerprinting**
The disclosed `User-Agent` reveals the backend relies on a Telegram-integrated bot for link/metadata processing — useful reconnaissance for an attacker researching known issues in that stack.

**4. Sensitive header/cookie leakage**
Outbound OOB requests were observed carrying internal session-related headers and cookies, which could aid session hijacking or further environment mapping if intercepted.

**5. Asynchronous, latency-based execution**
Because processing happens 2–5 minutes after injection via a backend worker, this class of SSRF is easy to miss with real-time monitoring and confirms the flaw sits deep in backend processing logic, not the request/response path.

---

## Root Cause

Inconsistent input validation: strict sanitization is applied to the primary chat input (`query`), but structurally similar metadata fields (`referral` → `source`, `DeviceType`) are trusted and passed downstream to an internal fetch/processing service without equivalent checks.

---

## Recommended Fix

1. **Input validation everywhere, not just the obvious fields** — apply the same whitelist-based sanitization to all fields inside `referral`, rejecting special characters (`'`, `"`, `<`, `>`, `/`) outside expected alphanumeric values.
2. **Disable unnecessary outbound fetching** — if the backend bot doesn't need to fetch external URLs, remove that capability entirely.
3. **Egress filtering via internal proxy** — if URL fetching is required, route it through a proxy that blocks internal/reserved IP ranges (`127.0.0.1`, `169.254.169.254`, `10.0.0.0/8`) and restricts protocols to `http`/`https`.
4. **Strip sensitive headers from outbound requests** — backend workers should never forward session cookies or internal auth headers when fetching external/untrusted URLs.

---

## Disclosure Status

Reported responsibly through the program's official bug bounty channel. Technical identifiers (domains, session values, screenshots) redacted in accordance with the program's responsible disclosure policy.
