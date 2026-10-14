# 2026-10-13 - One-time signing tokens reached the central log through the sign-in sidecar

**Status:** Mitigated - fix written, not yet deployed
**Area:** Azure Container Apps built-in authentication (Easy Auth) sidecar, Log Analytics, one-time links

## Summary

In a company project's staging environment, the one-time token of the customer's signing link ended up
in Log Analytics. The token was part of the URL path, and Container Apps' built-in sign-in runs as a
sidecar in front of the web app that logs every request with its full path. The app itself scrubbed the
token from its own access log, but the sidecar logs before the app sees the request. Fix: the token moved
out of the path, into the URL fragment for the page and into a request header for the API calls.

## Impact

- 79 log lines with a token over 14 days, staging only, kept for the workspace's 30-day retention.
- Each token was valid for 5 minutes and only until the signature succeeded, so every logged token was
  already dead when found. Nothing was misused as far as the logs show.
- Production was not affected: it has no sign-in sidecar in front of the web app.

## Environment

| Component | Detail |
|---|---|
| Hosting | Azure Container Apps, web app (Next.js, proxies `/api/*` to an internal API) + API app |
| Sign-in | Container Apps authentication (`authConfigs`), Microsoft Entra provider, on staging only |
| Logs | `appLogsConfiguration.destination: azure-monitor`, table `ContainerAppConsoleLogs`, 30 days |
| Link | mailed to the customer as `https://<staging-host>/<locale>/sign/<token>`; the page called `/api/v1/remote-sign/<token>` and `/api/v1/remote-sign/<token>/documents/<key>` |
| App logging | the API's access log replaced the token in remote-sign paths with `[redacted]` |

## Timeline

Local time, estimated (no log line marks the finding itself).

- 2026-10-13 morning - a full code review flags the risk: the sidecar's console output is collected,
  and its request lines may carry the path.
- 2026-10-13 afternoon - read-only Log Analytics query over 14 days confirms it (see Evidence).
- 2026-10-13 afternoon - fix decided and written: token in the fragment and a header.

## Root cause

Easy Auth runs as an extra container (`http-auth`) in the web app's replica and handles every request
before the app. It writes ASP.NET-style request logs to stdout (`Request starting … GET https://…/path`,
`RequestCompleted … "Path":"/…"`), and Container Apps collects every container's console output into
`ContainerAppConsoleLogs`. A secret in the URL path therefore reaches the central log no matter what the
app does with its own logs.

## Evidence

Query (read-only, over the staging workspace):

```kusto
ContainerAppConsoleLogs
| where Log has 'sign/'
| extend route = case(Log has '/remote-sign/', 'api', Log has '/sign/', 'page', 'other')
| summarize lines=count() by ContainerAppName, ContainerName, route
```

Result: container `http-auth` of the web app 22 lines (page) + 57 lines (API calls); the API app 12
lines, all of them with `[redacted]` in place of the token.

Line shapes from the sidecar (token replaced here):
`Request starting HTTP/1.1 GET https://<staging-host>/de/sign/<token>` and
`RequestCompleted … "Path":"/de/sign/<token>" …`.

## What didn't work

- **The app's own log scrubbing:** it worked for the app's lines and was irrelevant here, because the
  sidecar logs the request before it reaches the app.
- **`Referrer-Policy: no-referrer` on the page:** keeps the token out of other sites' logs, not out of
  the platform's own request log.
- Considered and not used:
  - **Redacting at ingestion** with a workspace transformation (the table supports it): more
    infrastructure, staging only, and it treats the symptom.
  - **Excluding the path from authentication:** the sidecar would still see and log the request.
  - **Dropping the sign-in for an IP allowlist:** staging needs the sign-in.

## Resolution

- The mailed link is now `https://<host>/<locale>/remote-sign#<token>`. A browser never sends the
  fragment to a server, so no proxy or sidecar sees it.
- The page reads the token from `location.hash`. It sends it to the API only in a custom request header
  (`X-Sign-Token`), never in a path or query string.
  - `Authorization` was avoided on purpose: Easy Auth tries to validate bearer tokens itself.
- The old path routes answer 404. Tests assert that the header is required and that no token appears
  in a path.
- The already logged lines age out with the 30-day retention; all their tokens were dead.

## Unverified

- That the sidecar never logs request headers. No header values appeared in the sampled lines, but
  that is not a guarantee.
- The fix on staging itself: pending the next deploy. Check it with the query above (expect only lines
  without a token).
- The customer page's POSTs from a `no-referrer` page also depend on the Easy Auth CSRF setting from
  the 2026-10-12 incident, which is still unverified after a deploy.

## Prevention

- **Rule:** never put a secret (token, code, key) in a URL path or query string. A link that must carry
  one uses the fragment, and the page hands it to the API in a header. This is written next to the
  link's route handlers and in the customer page's code, and the project's review checklist asks for it.
- **Tests** fail if a token route takes the token from the path again.
- **Habit:** with a platform component in front of the app (sign-in sidecar, gateway, WAF), check its
  logs too before assuming that the app's scrubbing is enough.
