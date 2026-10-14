# 2026-10-12 - Easy Auth silently rejected a POST with 403 (sub-status 60)

**Status:** Mitigated - fix written, not yet verified after deploy
**Area:** Azure Container Apps built-in authentication (Easy Auth), CSRF mitigation, Next.js proxy, Referrer-Policy

## Summary

In a company project's staging environment, which sits behind Container Apps' built-in Microsoft
sign-in (Easy Auth), one specific POST - the customer's signature on a signing page - failed every
time with the app's generic "request failed" text. The request never reached the app: Easy Auth's CSRF
mitigation answered `403` with sub-status `60` in under a millisecond. Fix: list the app's own origins
in the auth config's `login.allowedExternalRedirectUrls`.

## Impact

- The remote-signing flow could not be completed on staging for several hours of testing (nine
  rejected attempts in the logs).
- Production is not affected: it has no Easy Auth in front.
- Nothing lost.

## Environment

| Component | Detail |
|---|---|
| Hosting | Azure Container Apps, web app (Next.js, proxies `/api/*` to an internal API) + API app |
| Auth | Container Apps authentication (`authConfigs`), Microsoft Entra provider, "Require authentication", unauthenticated → redirect to login; `login.allowedExternalRedirectUrls` not set |
| Page | the signing page sets `Referrer-Policy: no-referrer` (the URL carries a one-time token) |
| Request | `fetch` POST with a JSON body (6-10 KB signature image), same origin, Easy Auth session cookie |

## Timeline

Local time (CEST). *log* = from Log Analytics (UTC + 2 h); the rest estimated.

- 14:47 *log* - first rejected signature POST (`403`, sub-status `60`).
- 14:47-18:28 *log* - eight more rejected attempts; the advisor's own POSTs on other pages pass.
- ~18:30 - API live log stream shows only the page's GET requests, no POST.
- ~18:35 - Log Analytics over all API replicas: no POST ever reached the API.
- ~18:40 - the web app's console logs (where Easy Auth's middleware writes) show
  `RequestCompleted … StatusCode 403, SubStatusCode 60` for every signature POST.
- ~18:50 - matched to Easy Auth's documented CSRF mitigation; fix added to the auth config.

## Root cause

Easy Auth rejects a POST that is authenticated through its session cookie, comes from a known browser,
and whose `Origin`/`Referer` is not in the configured list of allowed external redirect URLs (or CORS
origins) - [Microsoft docs, "Mitigation of cross-site request forgery"](https://learn.microsoft.com/en-us/azure/app-service/overview-authentication-authorization#mitigation-of-cross-site-request-forgery).
The list was empty. The signing page sends no `Referer` (`Referrer-Policy: no-referrer`), so its POST
carried only an `Origin` that was not on the list. POSTs from other pages (which send a `Referer`)
passed - see *Unverified* for why exactly.

## Evidence

- Web app console logs (Easy Auth middleware), one per attempt:
  `Request starting HTTP/1.1 POST http://<staging-host>/api/v1/remote-sign/<token> - application/json 10646`
  `RequestCompleted … "Path":"/api/v1/remote-sign/<token>" … "StatusCode":403,"SubStatusCode":60`
  `Request finished HTTP/1.1 POST https://<staging-host>/api/v1/remote-sign/<token> - 403 - - 0.7ms`
- The advisor's body-less POST on another page in the same logs: `- 200 … 684ms`.
- Log Analytics `ContainerAppConsoleLogs` for the API app across all replicas: the page's GETs, never
  the POST.
- The app's own error shown: a generic "request failed" (no response body it knew).

## What didn't work

- **The live log stream** (`az containerapp logs show` / portal Log stream): it connects to one replica
  only, so a missing line there proves nothing; Log Analytics over all replicas did.
- **Looking in the API's logs at all:** the request never got past Easy Auth in front of the web app.
- **Thinking of the Entra app registration's redirect URI list:** that list is checked by Microsoft's
  login server; this check runs in Easy Auth itself, with its own list in the Container App's auth
  config.
- **First guess, body size:** the rejected bodies were only 6-10 KB, and the rejection took < 1 ms.

## Resolution

`login.allowedExternalRedirectUrls` in the Container App's `authConfigs` now lists the app's public
origins (custom domain and generated FQDN), set in the project's infrastructure code so a deploy never
drops it.

## Unverified

- That the fix clears the 403 - pending the next staging deploy. If not, the other documented
  condition is the ingress CORS allowed origins.
- Why POSTs with a same-origin `Referer` passed while the docs' condition reads as if they should be
  rejected too; the observed difference between passing and failing POSTs is the missing `Referer`.

## Prevention

- The setting lives in the infrastructure code next to the auth config, with a comment naming this
  CSRF mitigation.
- The project's infra README explains it and has a "Logs" how-to: Log Analytics over all replicas,
  and Easy Auth's `RequestCompleted` lines (status + sub-status) are in the **web** app's console logs.
- Habit: with Easy Auth in front, a request that "never arrives" is checked in the web app's console
  logs for Easy Auth's sub-status first.
