# incidents

Postmortems for things that broke on my machines and in my projects - what
happened, why, what fixed it, and what keeps it from happening again.

The point is the last part. A fix without a written-down cause tends to get
undone a few weeks later by someone (often me, often an AI assistant) who
doesn't know why things are the way they are.

## Index

| Date | Incident | Area | Status |
|---|---|---|---|
| 2026-09-12 | [Claude Code sessions in herdr render as overlapping garbage](2026-09-12-herdr-claude-fullscreen-garbled.md) | herdr, Claude Code, Hyprland | Mitigated, upstream bug open |
| 2026-09-13 | [Claude Code session hung on a 30-minute import run in the foreground](2026-09-13-shop-scraper-import-hung-claude-session.md) | Claude Code, valiora shop-scraper, logging | Resolved |
| 2026-09-25 | [systemd-oomd killed the terminal running Claude Code, three times](2026-09-25-oomd-killed-terminal-parallel-agents.md) | Claude Code subagents, systemd-oomd, Astro builds | Mitigated |
| 2026-10-01 | [Both dock monitors stay black after the dock resets; re-login fixes it](2026-10-01-dock-monitors-dead-after-wake.md) | Hyprland / aquamarine, i915 DP MST, USB-C dock | Open, workaround: re-login |
| 2026-10-06 | [CI red on the first push to main: paths-filter could not fetch the previous commit](2026-10-06-ci-paths-filter-first-push-to-main.md) | GitHub Actions CI, dorny/paths-filter, actions/checkout | Resolved |
| 2026-10-12 | [OAuth login fails with "Login required" although the account is fine](2026-10-12-oauth-login-required-stale-vendor-session.md) | Third-party OAuth, browser session | Resolved |
| 2026-10-12 | [Easy Auth silently rejected a POST with 403 (sub-status 60)](2026-10-12-easy-auth-csrf-403-60-post.md) | Azure Container Apps Easy Auth, CSRF, Referrer-Policy | Mitigated, fix pending deploy |
| 2026-10-13 | [One-time signing tokens reached the central log through the sign-in sidecar](2026-10-13-signing-tokens-in-sidecar-logs.md) | Azure Container Apps Easy Auth sidecar, Log Analytics, one-time links | Mitigated, fix pending deploy |
| 2026-10-14 | [Cleanup job failed every 20 minutes on a record stored before a deploy](2026-10-14-cleanup-job-failed-on-record-stored-before-deploy.md) | Pydantic `extra="forbid"`, stored JSON records, Container Apps job, missing alert | Mitigated, fix pending deploy |

## How these are written

- **Blameless.** The question is what made the failure possible, not who
  pressed the key.
- **Evidence over plausible stories.** Every claim about the cause points at
  something that can be re-checked: a log line, a command and its output, an
  upstream issue. Anything that was not verified is marked as unverified.
- **What didn't work is part of the record.** It saves the next person from
  repeating the same dead ends.
- **Prevention has to live where it acts.** A note here is not enough; the
  config change, the comment next to the setting, or the check in a script
  is what actually prevents a repeat. Each postmortem links to those.
- **Nothing private.** No IP addresses, credentials, customer data or
  conversation contents from other projects.

New incidents start from [TEMPLATE.md](TEMPLATE.md), named
`YYYY-MM-DD-short-slug.md`, and get a row in the index above.
