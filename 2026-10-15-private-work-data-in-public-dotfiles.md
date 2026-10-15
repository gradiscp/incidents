# 2026-10-15 - Employer, work account and a private e-mail address were in the public dotfiles repo

**Status:** Mitigated - working tree and history clean, GitHub keeps two PR refs
**Area:** public dotfiles repo, Claude Code rules, git config, git history

## Summary

The dotfiles repo is public and its own rule says "nothing private". An audit
on 2026-10-15 found a Claude rule file naming the employer's GitHub org, two
private repos, the work account and a token's scope and expiry (pushed
2026-10-14), a git `includeIf` naming the work directory (pushed 2026-10-05),
and, in older history, a private e-mail address in `CLAUDE.md` and in one
commit message (2026-08-24 to 2026-09-13). The files were moved to untracked
per-machine files and the history was rewritten with `git filter-repo` and
force-pushed. GitHub still serves the old commits through `refs/pull/1` and
`refs/pull/2`.

## Impact

- Public exposure: e-mail address for 20 days, work names for 1 to 10 days
  (see timeline), until the force-push on 2026-10-15.
- No credentials or keys were ever in the repo (checked: no tokens, no IPs,
  no SSH material in any blob).
- Whether anyone cloned or archived the repo in that time is unknown.

## Environment

| Component | Detail |
|---|---|
| Repo | github.com/gradiscp/dotfiles, public, MIT |
| Rule | `~/.claude/rules` is a symlink to the whole `config/claude/rules/` dir, so a rule can only be private if git ignores it |
| Tool | git-filter-repo (Arch package), fresh `--no-local` clone, `.git` swapped under the live working tree |

## Timeline

Dates from the rewritten and the old git log.

- 2026-08-24 - private e-mail address enters `CLAUDE.md`; also in one commit message.
- 2026-09-13 - removed from `CLAUDE.md` during a rewrite (history keeps it).
- 2026-10-05 - `[includeIf "gitdir:~/Projects/<work>/"]` with the work directory name committed.
- 2026-10-14 - rule file with org, repos, account and token expiry committed and pushed.
- 2026-10-15 ~16:50 - audit agent finds all of it (`grep` over tree and `git log -p`).
- 2026-10-15 ~17:50 - files split into tracked public part and gitignored local part; git include made unconditional with the `includeIf` moved to an untracked file.
- 2026-10-15 ~18:10 - history rewritten in a scratch clone (37 pattern hits before, 0 after), verified, `.git` swapped; owner force-pushes.

## Root cause

Two mechanisms:

1. The rule directory is linked as a whole, so a work-specific rule had no
   place to live except inside the public repo. The rule was written where
   Claude would read it, without the "public" constraint in mind.
2. No check between "commit" and "push" looked for private patterns, and the
   repo's own rule listed what is private but had no enforcement.

## Evidence

- `grep -rniE 'Vermogensberatung|<account>|<work-dir>' --exclude-dir=.git .` before the fix: 4 hits in 3 files.
- `git log --all -p | grep -ciE '<patterns>'` before the rewrite: 37; after: 0.
- `git count-objects -vH`: `.git` 27 MB before, 15 MB after (the remaining size is theme wallpapers, kept on purpose).

## What didn't work

- `git filter-repo` on the live clone is refused without `--force` and would
  remove the remote; a fresh `--no-local` clone, rewrite, verify, then swap
  the `.git` directory worked and left the working tree untouched.
- A force-push cannot remove `refs/pull/N` on GitHub.

## Resolution

- Public part of the rule stays in `config/claude/rules/git-remote.md`; the
  work part lives in `config/claude/rules/gh-work.local.md`, ignored via
  `config/claude/rules/*.local.md`, still loaded through the link.
- `config/git/config` has a plain `[include] path = ~/.config/git/config.local`;
  the `includeIf` lives in that untracked file.
- History rewritten and force-pushed (dotfiles commit `5ecc6ad` and the rewrite).

## Unverified

- Clones or archives made while the data was public.
- GitHub's `refs/pull/1` and `/2`: removal needs GitHub Support or deleting and recreating the repository; not done.

## Prevention

- `.gitignore` pattern and the CLAUDE.md "Repo rules" section name the two
  untracked places for anything machine- or work-specific.
- The Claude deny list now covers `~/.config/git/config.local` and the work
  identity file, so a session cannot read them back into a tracked file.
- Before a push: `git grep -niE '<private patterns>' HEAD` (CHEATSHEET candidate).
