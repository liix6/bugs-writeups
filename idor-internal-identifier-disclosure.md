# IDOR — Internal Sensitive Identifier Disclosure via User Info Endpoint

**Category:** IDOR (Insecure Direct Object Reference)
**Severity:** High
**Target Type:** Community Platform (Large-scale consumer tech company) — *Program name anonymized per disclosure policy*

---

## Summary

An IDOR vulnerability was identified in a community platform's user information
endpoint. By modifying a numeric `uid` parameter, an attacker could retrieve
another user's private data — including a sensitive internal identifier used
to correlate accounts across internal company systems — without any
authorization checks.

Key issues:
- No access control enforced on the `uid` parameter
- An internal sensitive identifier is exposed in the API response
- This identifier links the user's account across multiple internal
  platforms, enabling cross-system user correlation

---

## Vulnerability Details

The `othersInfo` endpoint accepts a `uid` parameter to retrieve public profile
data for a given user. However, the endpoint fails to verify that the
requesting user is authorized to view the target user's full data set,
returning internal fields intended for privileged/internal use only.

---

## Steps to Reproduce

1. Log in as Account A on the target community platform.
2. Intercept the following request using an HTTP proxy (e.g., Burp Suite):

,,,
GET /ajax/user/frontend/user/othersInfo?uid=XXXXX
,,,


3. Modify the `uid` parameter to the identifier of another user (Account B).
4. Send the modified request.
5. Observe that the response discloses data belonging to Account B, including:
- An internal sensitive identifier (linking the account to other internal
  systems)
- Precise account creation and last-active timestamps
- Full permissions mapping
- Punishment/penalty history
- Agreement/consent status

## Proof of Concept

A video PoC demonstrating full reproduction is available upon request
(withheld from public writeup for disclosure safety).

---

## Impact

- Unauthorized access to any user's private account data via simple
parameter manipulation
- Disclosure of an internal identifier not intended for public exposure
- Cross-platform account correlation: the same internal identifier was
found to link to a separate internal environment (e.g., a developer
portal), expanding the impact beyond the original platform
- Increased attack surface — the internal identifier could potentially be
leveraged against other internal services relying on the same identity
system

---

## Root Cause

Missing object-level authorization: the endpoint trusts the client-supplied
`uid` value without verifying that the requester has permission to access
the target user's extended profile data.

---

## Remediation

- Enforce strict authorization checks on the `uid` parameter — restrict
access to the requester's own data unless explicit permission exists
- Remove or mask internal sensitive identifiers from API responses intended
for general users
- Use non-sensitive public-facing identifiers instead of internal system
identifiers wherever possible
- Audit related endpoints for the same authorization gap

---

## Disclosure Status

Reported responsibly through the program's official bug bounty channel.
Details anonymized in accordance with the program's responsible disclosure
policy.

