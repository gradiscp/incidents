# 2026-10-01 - Both dock monitors stay black after the dock drops them; only a reboot helps

**Status:** Open (workaround: reboot)
**Area:** Hyprland / aquamarine, i915 DP MST, USB-C dock, idle blanking

## Summary

Both external monitors on the USB-C dock went dark whenever the mouse moved,
then stayed black. Hyprland still listed them, but the kernel rejected every
mode it tried for them, down to 720x400. Unplugging and replugging the dock
and `hyprctl reload` changed nothing; a reboot brought both back. It has
happened before (per the owner, not recorded); this is the first write-up.

## Impact

- Both external monitors unusable until a reboot, roughly 13:00-13:21.
- Laptop panel (eDP-1) kept working throughout. Nothing lost.
- Dock USB devices (keyboard, mouse, Ethernet) came back after each replug;
  only the displays did not.

## Environment

| Component | Detail |
|---|---|
| Machine | Samsung Galaxy Book Pro 360 (Tiger Lake, i915), kernel 7.2.5-3-omarchy |
| Compositor | Hyprland 0.56.2, aquamarine 0.15.0 |
| Dock | USB-C dock with DP MST; monitors appear as DP-6 / DP-7 (later DP-8 / DP-9) |
| Monitors | Dell AW2521HFA (1080p@60), Samsung LS24AG30x (1080p@144) |
| Monitor config | single catch-all rule: `mode = "preferred"`, `position = "auto"`, scale 1.25 |
| Idle | `gradiscp.idle`: screensaver after 300 s, lock (`omarchy-system-lock`, blanks the displays) after 420 s; values changed earlier the same day from 120 / 180 |

## Timeline

The Hyprland log has no timestamps; its line numbers give the order. Clock
times are from `journalctl` unless marked.

- Boot - DP-6 modeset 1920x1080@60, DP-7 1920x1080@144 (log l. 397, 513)
- Morning - normal use with both monitors
- Unknown time - eDP-1, DP-6 and DP-7 all disabled together, then re-enabled
  with the same modes (l. 12125-12265)
- Directly after - udev removes card1-DP-6 and DP-7 (l. 12548), adds DP-8 and
  DP-9 (l. 12722, 12972). Every modeset on them fails from here (first at
  l. 12821)
- ~13:00 (est.) - owner sees: screens go dark when the mouse moves, are visible
  when nothing moves
- 13:03:17 - `gradiscp.idle` last event `idle-monitor: active`
  (`omarchy-shell idle status`)
- 13:03:42 - owner unplugs the dock; USB disconnects in the kernel log, devices
  back at 13:03:46-47. DP-8 / DP-9 re-added, still failing
- ~13:05 - `hyprctl reload`: one more failed commit, no change
- ~13:13 - second unplug/replug: DP-8 / DP-9 re-added, still failing
- 13:15 - logs saved to `~/.cache/hyprland/`
- 13:21:07 - reboot (`uptime -s`); monitors back as DP-6 1080p@60 and
  DP-7 1080p@144

## Root cause

Not established. What is shown: after the dock's MST connectors were torn
down and re-created as DP-8 / DP-9, the kernel rejected every atomic modeset
for them with `EINVAL`, for all 31-36 modes including 640x480 and 720x400.
That rules out a bandwidth or mode problem on the Hyprland side and points at
stale state in the display stack (i915 MST or aquamarine) that only a reboot
cleared.

The trigger is suspected, not proven: the drop happened right after all three
displays were blanked and woken together, and the owner's symptom (dark on
mouse move) matches a wake. See Unverified.

## Evidence

Saved before the reboot (`/run/user` is wiped on reboot):

- `~/.cache/hyprland/dock-dropout-2026-10-01-hyprland.log` - Hyprland log of
  the failing session
- `~/.cache/hyprland/dock-dropout-2026-10-01-kernel.log` - `journalctl -k -b`
  of the same boot

In the Hyprland log:

```
drm: eDP-1 is disabled, releasing crtc 171          (l. 12125)
drm: DP-6 is disabled, releasing crtc 309
drm: DP-7 is disabled, releasing crtc 447
...
drm: Modesetting DP-6 with 1920x1080@60.00Hz        (l. 12263)
drm: Modesetting DP-7 with 1920x1080@144.00Hz       (l. 12265)
udev: new udev remove event for card1-DP-6          (l. 12548)
udev: new udev remove event for card1-DP-7
udev: new udev add event for card1-DP-8             (l. 12722)
atomic drm request: failed to commit: Invalid argument, flags: ATOMIC_ALLOW_MODESET ATOMIC_TEST_ONLY
drm: atomic commit failed with max_bpc set, retrying without max_bpc
... (same for every mode down to 720x400)
drm: Modesetting DP-8 with 720x400@70.08Hz
ERR: atomic drm request: failed to commit: Invalid argument, flags: ATOMIC_ALLOW_MODESET PAGE_FLIP_EVENT
```

1812 `failed to commit: Invalid argument` lines in total, 12 of them real
(non-test) commits.

State while broken:

```
$ hyprctl monitors all -j | jq ...
DP-8 AW2521HFA 0x0@60 disabled=false
DP-9 LS24AG30x 0x0@60 disabled=false
$ cat /sys/class/drm/card1-DP-8/{status,enabled}
connected
disabled
```

The kernel log at default level has no i915 / DRM error for this; only the
USB disconnect / reconnect of the dock at 13:03.

## What didn't work

- **Unplug and replug the dock** (twice). Connectors come back as the same
  DP-8 / DP-9 and every modeset still fails.
- **`hyprctl reload`.** Re-applies monitor rules; one more failed commit.
- **Not tried on purpose:** `hyprctl dispatch dpms off/on`. It blanks the
  laptop panel too, and Hyprland is known to segfault on hotplug events while
  the panel is blanked (dotfiles `CLAUDE.md`, "Lock and idle").

## Resolution

Reboot. Logging out and back in (restarts Hyprland, not the kernel) was not
tried, so it is unknown whether that would be enough.

## Unverified

- That the blank-and-wake was the screensaver or the lock (`omarchy-system-lock`
  blanks the displays). The log shows all three displays disabled and
  re-enabled together, which fits, but has no timestamps to tie it to the idle
  events.
- Whether the stale state is in the kernel (i915 MST) or in aquamarine. A
  re-login would tell: if it helps, the kernel side is fine.
- Whether the idle values changed earlier that day (120/180 s -> 300/420 s)
  matter. Probably not; the earlier occurrences predate the change.
- The dock model; `lsusb` is not installed. USB IDs from the kernel log:
  hub `2109:2817` / `2109:0817`, Ethernet `0bda:8153`.

## Prevention

None yet. Next time it happens:

1. Before rebooting, note whether it followed the screensaver or lock.
2. Save `$XDG_RUNTIME_DIR/hypr/*/hyprland.log` and `journalctl -k -b`
   before rebooting.
3. Try logging out and back in before a full reboot.

If blank-and-wake is confirmed as the trigger, candidates are leaving the
external monitors out of the idle blanking or reporting it upstream
(Hyprland / aquamarine or i915). Both need the owner's decision.
