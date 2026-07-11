# Insufficient Session Expiration — Failure to Invalidate Authentication Sessions After Logout (CWE-613)

**Category:** Session Management / Broken Authentication
**CWE:** CWE-613 (Insufficient Session Expiration)
**Target Type:** Government/Enterprise E-Services Web Portal — *Program name anonymized per disclosure policy*

---

## The vulnerability

Logout is one of those features everyone assumes just works. You click it, the session ends, done. So naturally, it's one of the most under-tested flows in any application — which is exactly why I like to slow down and actually verify it instead of trusting the UI.

I logged into the portal, opened DevTools, and pulled up the authentication cookie — `.AspNet.ApplicationCookie`. Saved its value. Then I did the obvious thing: clicked Logout, watched the cookie disappear from the browser, and moved on... except I didn't move on. I opened an entirely separate incognito window that had never touched this site, manually injected that "expired" cookie value back in, and refreshed.

I was still logged in.

Not a fluke — I repeated the whole cycle. Logged in again, grabbed a fresh cookie, logged out, tried the old one in a clean session. Same result. I let more than 24 hours pass and tried an even older cookie from earlier in the day. Still valid. At that point it was clear: logout on this application does exactly one thing — it deletes a cookie from *your* browser. It does nothing on the server. The session token itself is never touched, never invalidated, never expired. Every cookie you've ever been issued stays alive and authenticated indefinitely, stacking up in the background.

That means anyone who ever captured one of your session cookies — through a shared device, a proxy log, a browser history dump, malware, whatever — could use it days later, from a completely different browser, and just walk right into the account. Logout gives users a false sense of security while doing nothing to actually protect them.

---

## Steps to Reproduce

1. Log in using a valid account.
2. Open DevTools → Application → Cookies, copy the value of
   `.AspNet.ApplicationCookie` → save as **Cookie A**.
3. Log out of the account.
4. Log in again, copy the new `.AspNet.ApplicationCookie` value → save as
   **Cookie B**.
5. Log out again.
6. Open a separate browser or a fresh incognito window that has never been
   used to log in, and navigate to the target application.
7. Manually inject a cookie:
   - Name: `.AspNet.ApplicationCookie`
   - Value: Cookie A
8. Refresh the page (F5).
9. Observe that the account is fully authenticated using the "old" cookie.
10. Repeat with Cookie B (and additional cookies collected over time,
    including after 24+ hours) — all remain valid.

## Proof of Concept

Full video PoC demonstrating reproduction across multiple sessions and
browsers available upon request (withheld from public writeup for
disclosure safety).

---

## Actual Result

All previously issued authentication cookies remained valid and continued
granting authenticated access despite:
- Explicit logout actions performed by the user
- Multiple subsequent logins issuing fresh cookies
- More than 24 hours elapsing between issuance and reuse
- Reuse occurring from a completely separate browser/session context

---

## Impact

- **Persistent Session Hijacking:** Any authentication cookie ever issued to
  a user remains valid indefinitely (or at minimum, well past 24 hours and
  past explicit logout), regardless of how many times the user logs out or
  back in.
- **Session Fixation Exposure:** Since logout does not rotate or invalidate
  the underlying session value, an attacker who fixes or captures a session
  token retains access even after the legitimate user believes they've
  logged out.
- **Violation of User Security Expectations:** Users reasonably assume
  "Logout" terminates their session everywhere it was used — this
  assumption is false here, undermining the basic security guarantee of the
  logout function.
- **Extended Attack Window:** Because old cookies remain valid for 24+
  hours (and were not observed to expire during testing), any credential
  captured via shared devices, logs, browser history, or malware remains
  exploitable long after the fact.

---

## Root Cause

The application performs logout as a **client-side-only operation** —
removing the cookie from the browser — without invalidating the
corresponding session token on the server. New logins generate new tokens,
but old tokens are never revoked, allowing multiple valid sessions to
coexist indefinitely for the same account.

---

## Recommended Remediation

1. **Server-side session invalidation on logout:** Immediately revoke the
   session token server-side when a user logs out, not just client-side
   cookie deletion.
2. **Single active session enforcement (or explicit multi-session
   tracking):** Invalidate previously issued tokens when a new login occurs,
   or maintain a server-side session store that allows selective revocation.
3. **Session expiration policy:** Enforce a reasonable absolute and idle
   timeout on authentication cookies rather than allowing indefinite
   validity.
4. **Session token rotation:** Issue a new session identifier on each login
   and invalidate all prior identifiers tied to that account.
5. **Server-side session store:** Move from purely client-trusted cookies to
   a server-tracked session mechanism, enabling centralized revocation.

---

## Disclosure Status

Reported responsibly through the program's official disclosure channel.
Domain and account-identifying details redacted in accordance with the
program's responsible disclosure policy.
