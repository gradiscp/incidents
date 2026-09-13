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
