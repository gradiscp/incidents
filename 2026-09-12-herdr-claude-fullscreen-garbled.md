# 2026-09-12 - Claude Code sessions in herdr render as overlapping garbage

**Status:** Mitigated - the upstream herdr bug is still open
**Area:** herdr (terminal workspace manager), Claude Code, Hyprland

## Summary

Every Claude Code session running inside herdr became unreadable: new lines
were drawn over leftovers of old ones, with fragments of different messages
fused into the same row. Claude Code ran in its fullscreen renderer, which
uses the terminal's alternate screen, and herdr has an open bug where
resizing an alternate-screen application corrupts herdr's own copy of the
screen. The herdr window was resized again and again because its Hyprland
workspace tiles with `dwindle`, which splits the screen whenever another
window opens beside it. The fullscreen renderer was switched off and the
sessions were restarted.

## Impact

- Four Claude Code sessions in herdr were unreadable. The processes kept
  running and kept accepting input; only the display was broken.
- Nothing was lost. Claude Code stores conversations on disk, and each
  session could be continued with `claude --resume`.
- A Claude Code session in a plain foot window (outside herdr) also turned
  garbled later, right after the settings change - see *Unverified*.

## Environment

| Component | Version / setting |
|---|---|
| OS | Arch Linux with Omarchy, kernel 7.2.3 |
| Compositor | Hyprland; global layout `scrolling`, but workspaces 1-5 override it with `dwindle` |
| Terminal | foot |
| herdr | 0.8.2 (Omarchy package repo) |
| Claude Code | 2.1.267, installed via mise |
| Claude Code setting | `"tui": "fullscreen"` in `~/.claude/settings.json` |

## Timeline

Local time (CEST). Times marked *log* come from
`~/.config/herdr/herdr-server.log` (UTC there, converted); *mtime* and
*ps* are file modification and process start times.

- 2026-09-10 18:51 - Claude Code 2.1.267 installed (mise install directory
  timestamp). Not shown to be related, listed for completeness.
- 12:19-12:23 and 14:44-14:45 *log* - herdr client briefly switched from 233
  to 113 columns and back, several times.
- 14:50 *mtime* - Hyprland layout override for workspace 2 last written
  (`workspace-layouts/2.lua`, content: `dwindle`).
- 15:00:51-15:01:05 *log* - herdr client resized four times between 233
  and 113 columns.
- 15:09:32 *log*, *ps* - a foot window was opened on the same workspace;
  herdr's client resized to 113 columns in the same second.
- Afterwards - garbled Claude Code panes noticed and reported.
- Investigation (see *Evidence*): corruption confirmed inside herdr, Ctrl+L
  tried, matching upstream issues found.
- 15:16-15:17 *log* - Claude Code exited in the herdr panes.
- 15:18:40 *mtime* - `"tui": "fullscreen"` removed from the Claude Code
  settings.
- Shortly after - the Claude Code session in the plain foot window rendered
  garbled as well; it was restarted with `claude --resume` at 15:20:40
  (*ps*).
- Later - `hyprctl` showed workspace 2 tiled with `dwindle`, not the global
  `scrolling` layout.

## Root cause

1. **Claude Code's fullscreen renderer draws on the alternate screen.** With
   `"tui": "fullscreen"` every session was a full-screen terminal
   application, the same class of program as vim or less.
2. **herdr corrupts its screen grid when an alternate-screen application is
   resized.** herdr keeps its own copy of each pane's screen and draws the
   outer terminal from that copy. On a resize, cells of that copy end up
   stale or shifted. This is
   [herdr#3329](https://github.com/herdrdev/herdr/issues/3329), open as of
   2026-09-12. [herdr#3686](https://github.com/herdrdev/herdr/issues/3686)
   reports exactly this with Claude Code and was closed as its duplicate.
3. **Hyprland halved the herdr window whenever a second window joined its
   workspace.** The global layout is `scrolling` with `column_width = 1.0`,
   where each window gets its own full-width column. But Omarchy's
   `SUPER+H` (toggle layout) writes a per-workspace override to
   `~/.local/state/omarchy/workspace-layouts/<N>.lua` that beats the global
   setting, and workspaces 1-5 all carry one set to `dwindle`. `dwindle`
   splits the screen between windows: herdr and a foot window sat at 755 px
   each (`hyprctl clients`), which gave herdr 113 columns instead of 235.
   Every window opened, closed or moved on that workspace resized every pane
   in herdr.

The corruption is persistent: Claude Code does not repaint over the broken
grid, so the pane stays garbled until the program is restarted.

## Evidence

- `herdr pane read <pane>` returned the same overlapping text as the screen.
  That output comes from the herdr server's grid, which rules out the outer
  terminal (foot) as the cause.
- Pane metadata showed `max_offset_from_bottom: 0` for every Claude pane:
  no scrollback, consistent with an alternate-screen application.
- herdr server log, client resizes (UTC):
  ```
  13:00:51 client resize cols=233 rows=55
  13:00:52 client resize cols=113 rows=55
  13:01:03 client resize cols=233 rows=55
  13:01:05 client resize cols=113 rows=55
  13:09:32 client resize cols=113 rows=55
  ```
- The foot window next to herdr was started at 15:09:32 local
  (`ps -o lstart`), the same second as the last resize above.
- Layout of the workspace herdr was on:
  ```
  $ hyprctl activeworkspace -j   # "tiledLayout": "dwindle"
  $ hyprctl getoption general:layout   # str: scrolling
  $ cat ~/.local/state/omarchy/workspace-layouts/2.lua
  hl.workspace_rule({ workspace = "2", layout = "dwindle" })
  ```
  `1.lua`, `3.lua`, `4.lua` and `5.lua` contain the same rule for their
  workspaces.
- herdr#3686, same symptom with Claude Code: "No keystroke, `ctrl+l`, ...
  the visible grid never changes". The maintainer's closing comment names
  the trigger: "an alternate-screen application is resized, then stale and
  miswrapped cells appear in `pane read`".

## What didn't work

- **Ctrl+L** (sent with `herdr pane send-keys <pane> ctrl+l`). Claude Code
  redrew, and the pane stayed garbled, as herdr#3686 describes.
- **Updating herdr** is not a fix. herdr 0.9.0 was available upstream, but
  its changelog lists no fix for this, herdr#3329 is still open, and the
  Omarchy package repo only had 0.8.2.
- **`ui.pane_scrollbars = false`**, the workaround suggested in herdr#3329
  for a one-column resize on every alternate-screen switch, was already set
  here. It removes one resize trigger, not window resizes.
- **Updating Claude Code** (2.1.268 was available) - nothing in its changelog
  relates to this.

## Resolution

- Removed `"tui": "fullscreen"` from the Claude Code settings, so new
  sessions use the classic renderer, which does not use the alternate
  screen.
- Restarted each affected session with `/exit` and `claude --resume`.

## Unverified

- **That the classic renderer survives herdr resizes cleanly.** herdr#3329
  is about alternate-screen applications, so it should not apply, but no
  resize was tested after the change.
- **Why the session in the plain foot window broke.** That window was not
  inside herdr. The only thing that happened around that time was the
  settings change at 15:18:40, so the likely cause is Claude Code picking up
  the removed `tui` setting while running and switching renderers
  mid-session. No log was available to confirm it.
- **Which workspace herdr was on during the earlier resizes, and what
  triggered them.** The switches to 113 columns at 12:19 and 14:44 predate
  the 14:50 write of `2.lua`; a file's modification time only shows its last
  write, so whether workspace 2 was already `dwindle` then is unknown.
- **Whether the `dwindle` overrides are wanted.** They may have been set on
  purpose with `SUPER+H`, or by accident.
- **How the herdr window ended up in a Hyprland window group** (a group with
  only itself in it, which draws a title bar above the window). `SUPER+G`
  toggles grouping in Omarchy; whether that was pressed is not known. A
  group of one does not change the window's width and is not part of the
  root cause.

## Prevention

- **Keep `tui` unset while herdr#3329 is open.** Documented next to the
  setting's home in the dotfiles repo (`CLAUDE.md`, section on Claude
  settings), so the fullscreen renderer is not switched back on without
  knowing why it is off.
- **Check workspace layout overrides when windows resize unexpectedly.**
  `ls ~/.local/state/omarchy/workspace-layouts/` - any file there beats the
  global `scrolling` layout for that workspace. The dotfiles `CLAUDE.md`
  (Workspaces section) already documents this gotcha; it had bitten twice
  before and is a contributing cause here.
- **Change Claude Code renderer settings between sessions, not under running
  ones.** Until the live-switch behaviour above is understood, edit the
  setting, then restart the sessions deliberately.
- **Recovery is cheap:** `/exit`, then `claude --resume`. Ctrl+L does not
  help.
- **Re-evaluate** when herdr#3329 is closed: the fullscreen renderer has real
  advantages (no flicker, mouse support), and this is the only reason it is
  off.
