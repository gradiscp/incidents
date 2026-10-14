# 2026-10-12 - OAuth login fails with "Login required" although the account is fine

**Status:** Resolved
**Area:** OAuth 2.0 authorization code flow against a third-party CRM, browser session, web app login

## Summary

In a company project, signing in through a third-party CRM's OAuth server failed for one developer
with "login failed". The token exchange succeeded, but the CRM's API rejected the fresh token on the
very first call with `401 {"error":"Login required"}`. The old app showed the same, so it looked like a
missing account permission. A manual OAuth run in a private window worked with the same account: the
cause was a stale or unusual CRM session in the developer's normal browser. Logging in from a private
window (or logging out of the CRM) fixed it.

## Impact

- About an hour spent on the wrong hypotheses (code bug, account permissions) on the day the new
  app's staging environment first ran against the live CRM.
- Nothing lost; no customer affected. Only the developer's own login was affected; a colleague could
  sign in normally.

## Environment

| Component | Detail |
|---|---|
| App | company project web app + API; the API runs the code exchange (confidential client, HTTP Basic client auth) and then calls the CRM's `account/read` with `Authorization: Bearer` and an API key header |
| CRM | third-party, own OAuth server (not Microsoft), authorization code flow without PKCE |
| Browser | Firefox on Linux, normal profile with an existing CRM web session vs. a private window |

## Timeline

Local time (CEST). Times marked *log* come from the API's audit log (UTC + 2 h); the rest is estimated.

- 14:07 *log* - first attempt on staging: `auth.login.<crm>.failed step=account status=401`.
- 14:08 *log* - second attempt, same result.
- ~14:15 - the old app (same CRM client) fails the same way → assumed account permission problem.
- ~14:20 - API key checked directly against the CRM (200 with key, 403 without) → key fine.
- ~14:30 - code compared line by line with the old app (scopes, headers, token exchange) → identical.
- ~14:40 - manual OAuth run in a private window: token exchange 200, all scopes granted,
  `account/read` 200 for the same account.
- 14:55 *log* - login in the app from a private window succeeds (`auth.login`).

## Root cause

The authorization code was issued from the developer's existing CRM session in the normal browser.
The token the API got for that code passed the token endpoint but was refused by the CRM's API as
"Login required". In a fresh session (private window) the same account, client, scopes and code path
work. What exactly is wrong with that browser session is not known (see *Unverified*).

## Evidence

- API audit log, after a successful token exchange:
  `{"action": "auth.login.<crm>.failed", "actorId": "unknown", "details": {"step": "account", "status": 401}}`
- CRM's answer to `account/read` with an invalid token (curl): `{"error":"Login required","errorDetail":null} [401]` —
  the same status the app got, so the app's token was treated as invalid.
- Manual run in a private window (script, tokens never printed): token `200`, `token_type: Bearer`,
  `expires_in: 3600`, refresh token present, all ten requested scopes granted; `account/read` `200`
  with the account's data; revoke `200`.
- App login from a private window right after: audit line `auth.login` (success).

## What didn't work

- **Assuming a code bug:** the request is byte-for-byte what the old app sends; the old app failed
  the same way for this account.
- **Assuming a missing account permission:** the manual run with the same account succeeded.
- **The registered localhost redirect URI for the manual run:** `redirect_uri_mismatch` — the old
  localhost path noted in the project's credentials note is not (or no longer) registered.
- **What did work for debugging:** using another registered redirect URI whose host answers that path
  with `404` (so nothing consumes the code), copying the code from the address bar and exchanging it
  with a small script that prints only scopes, token type, expiry and the API's answer.

## Resolution

Sign in from a private window, or log out of the CRM in the normal browser (or clear its cookies),
then start the app's login again.

## Unverified

- Why the normal browser's CRM session produces tokens the API rejects (possible: an admin
  "log in as another user" session, a half-expired session, a session flag such as a forced
  password change). The vendor was not asked.
- Whether the CRM's API ever returns more detail than "Login required" for this case.

## Prevention

- The project's CRM integration reference now has a troubleshooting section: "Login required" right
  after a successful token exchange → try a private window / log out of the CRM first.
- Backlog item in the project: log the CRM's error text and the granted scopes on a failed login
  (never tokens), so the audit line says why instead of only the status.
- General habit: test a third-party OAuth login in a private window before suspecting code or
  permissions.
