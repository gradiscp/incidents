# 2026-10-15 - A package the dotfiles remove on purpose was back for three weeks, unnoticed

**Status:** Resolved
**Area:** dotfiles `remove-unwanted-apps.sh`, `omarchy-drift-check`, pacman

## Summary

`remove-unwanted-apps.sh` removed LibreOffice on 2026-09-13. On 2026-09-24 it
was installed again with a plain `pacman -S libreoffice-fresh` (by hand or
through Omarchy's Install menu, which runs the same command). Nothing checked
for that, so the 423 MB package sat there until an audit on 2026-10-15 ran
`pacman -Q` against the removal list. The drift check now reports every
package from that list that is installed, after each `omarchy update` and on
demand.

## Impact

- 423 MB disk and one unwanted app in the menus for three weeks. Nothing lost.
- The assumption "install.sh reproduces this machine" was false for that
  period: a fresh install would have lacked LibreOffice, this machine had it.

## Environment

| Component | Detail |
|---|---|
| Omarchy | 4.0.4-1, LibreOffice is in `omarchy-base.packages` (line 73) |
| dotfiles | `remove-unwanted-apps.sh` with a fixed `unwanted` list; drift check only compared symlinks and the theme NoExtract rule |

## Timeline

Local time, from `/var/log/pacman.log` unless marked.

- 2026-09-13 20:33 - `removed libreoffice-fresh (26.8.0-2)` by the removal script.
- 2026-09-24 17:22 - `Running 'pacman -S libreoffice-fresh'`, then `installed libreoffice-fresh (26.8.0-2)` with its dependencies.
- 2026-10-15 ~17:00 (estimated) - audit lists explicitly installed packages against the removal list; LibreOffice is there.
- 2026-10-15 18:00 - removed again by the updated script.

## Root cause

The removal was a one-shot action with no check that its result holds. Any
later `pacman -S` of a listed package, whether typed or clicked in Omarchy's
Install menu, silently undid it. During the audit the reinstall was first
blamed on `omarchy update`; the log shows an explicit install command, so the
cause was a manual or menu action, not an update.

## Evidence

```
$ grep -E '(removed|installed) libreoffice-fresh|Running .pacman -S libreoffice' /var/log/pacman.log
[2026-09-13T20:33:10+0200] [ALPM] removed libreoffice-fresh (26.8.0-2)
[2026-09-24T17:22:34+0200] [PACMAN] Running 'pacman -S libreoffice-fresh'
[2026-09-24T17:22:39+0200] [ALPM] installed libreoffice-fresh (26.8.0-2)
```

The only pacman invocation that day was that one command; no `omarchy update`
ran on 2026-09-24.

## What didn't work

- Reading `remove-unwanted-apps.sh` and assuming the machine matched it.
- The first explanation ("an update reinstalled it"), written into the
  dotfiles before the log was read; corrected on the same day.

## Resolution

`remove-unwanted-apps.sh` removed the package again (dotfiles commit `9d7dec1`).

## Unverified

- Whether the 2026-09-24 install came from a terminal or from Omarchy's
  Install menu. Both run `pacman -S`; the log does not distinguish them.

## Prevention

- `remove-unwanted-apps.sh --check` prints every package from the removal
  list that is installed and exits 1.
- `omarchy-drift-check` calls it and raises a red toast; the check runs as an
  `omarchy update` post-update hook and by hand (CHEATSHEET).
- Both in the dotfiles repo, commit `9d7dec1` (2026-10-15).
