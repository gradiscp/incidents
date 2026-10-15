# 2026-10-15 - Tools wrote through dotfiles symlinks straight into the public repo

**Status:** Mitigated
**Area:** dotfiles symlinks, mise, herdr, Omarchy `omarchy.agents` widget, Claude Code `settings.json`

## Summary

Most of `~/.config` is a symlink into the public dotfiles repo. Twice on one
day a tool wrote a config file through such a link: a lazy `codex` wrapper
re-added `codex` to the mise config after it had been removed, and `herdr
integration install claude` added a SessionStart hook with an absolute home
path to the Claude settings. Both landed in commits before they were noticed.
One commit was amended before the push; the other was found by diffing the
history afterwards. The repo now guards both writers and documents the trap.

## Impact

- A removed tool (`codex`) came back into `config/mise/config.toml` and was
  committed; found about an hour later, not pushed.
- `config/claude/settings.json` gained a duplicate hook containing
  `/home/<user>/...`, which the repo rules forbid; committed with `git add -A`,
  caught by reading the commit, amended before the push.
- An earlier case of the same mechanism (absolute home paths in
  `settings.json`) had reached GitHub and was cleaned on 2026-09-22
  (dotfiles commit `9f9e6ff`).

## Environment

| Component | Detail |
|---|---|
| Omarchy | 4.0.4-1; `omarchy-mise-install` writes a wrapper per tool into `~/.local/bin` that runs `mise use -g <tool>` on first call |
| mise | `~/.config/mise/config.toml` -> repo file |
| herdr | `integration install claude` edits `~/.claude/settings.json` -> repo file |
| Omarchy shell | `omarchy.agents` bar widget polls `claude`, `codex` and `fireworks` for usage meters |

## Timeline

Local time, estimated from the shell session.

- 17:30 - `codex`, `pi`, `node` removed from the mise config; the `codex` wrapper in `~/.local/bin` still exists.
- 17:3x - `omarchy restart shell`; the agents widget runs `codex`, the wrapper runs `mise use -g codex`, mise writes `codex = "latest"` through the link.
- 17:45 - the change is swept into the "Shell" commit by `git add -A`.
- 18:30 - `mise ls` still lists codex; the link is checked, `git log -p` shows the write; fixed, widget providers disabled.
- 21:22 - `install.sh` test run calls `herdr integration install claude`; herdr appends a second SessionStart hook with an absolute path through the link.
- 21:23 - `git add -A` commits it with an unrelated change; `git show` reveals it; commit amended, install.sh guarded.

## Root cause

A symlink makes the repo file the live file. Any program that rewrites "its"
config rewrites the repo. Two programs did so legitimately from their point of
view: mise persists `mise use -g`, herdr installs its hook. Neither knows the
file is version-controlled and public. `git add -A` then turns the surprise
into a commit.

## Evidence

- `git log -p -- config/mise/config.toml` showed `+codex = "latest"` in the
  Shell commit, right after the Look commit had removed it.
- `git show HEAD -- config/claude/settings.json` showed the added block with
  `"command": "bash '/home/<user>/.claude/hooks/herdr-agent-state.sh' session"`.
- `cat ~/.local/bin/codex` (before removal): `mise use -g "codex" || exit 1`.

## What didn't work

- Removing the tool from the mise config without removing the wrapper that
  reinstalls it.
- Relying on `git status` being clean before a commit: the write happened
  between two commits in the same session.

## Resolution

- All lazy agent wrappers removed from `~/.local/bin`; `install.sh` removes
  them on a fresh machine; the agents widget polls only Claude
  (dotfiles commit `1ce94b4`).
- `herdr integration install claude` runs only before `settings.json` is
  linked and only if its hook script is missing (dotfiles commit `abacb60`).

## Unverified

- Whether other Omarchy or mise commands write through links as well. The
  drift check catches a link replaced by a file, but not a write through an
  intact link.

## Prevention

- Guards above, in `install.sh`.
- CLAUDE.md rule: `git diff config/claude/settings.json` before every commit;
  `*.local.md` and `~/.config/git/config.local` for anything machine- or
  work-specific.
- Habit: `git add <paths>` for linked files, not `git add -A`.
