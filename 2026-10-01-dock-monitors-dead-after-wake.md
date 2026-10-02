# 2026-10-01 - Both dock monitors stay black after the dock resets; re-login fixes it

**Status:** Open (workaround: log out and back in)
**Area:** Hyprland / aquamarine, i915 DP MST, USB-C dock

## Summary

When the USB-C dock resets - reproduced on 2026-10-02 by pulling the dock's
power cable - its two monitors come back under new connector names and stay
black: Hyprland lists them, but every modeset it tries is rejected with
`EINVAL`, down to 720x400. Replugging the dock and `hyprctl reload` do not
help. Logging out and back in (new Hyprland, same boot) brings both back on
the very same connectors, so the stuck state lives in the running Hyprland /
aquamarine instance. It has happened before (per the owner, not recorded).

## Impact

- 2026-10-01: both external monitors unusable until a reboot, roughly
  13:00-13:21.
- 2026-10-02: unusable for about 5 minutes (06:38-06:43), until a re-login.
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

### 2026-10-01

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

### 2026-10-02

- 06:35:51 - boot; DP-6 / DP-7 modeset normally (Hyprland log l. 410, 425)
- 06:38:47 - owner pulls the power cable that feeds the dock (and the laptop
  through it). Every USB device on the dock disconnects at once and is back
  at 06:38:48-53 - the whole dock reset. No display blanking before it.
- Directly after - DP-6 / DP-7 removed (l. 502), DP-8 / DP-9 added (l. 662,
  912), every modeset fails again (604 failed commits by 06:40)
- 06:41 - logs saved to `~/.cache/hyprland/dock-dropout-2026-10-02-*.log`
- ~06:43 (est.) - owner logs out and back in; new Hyprland started 06:43:45
  (`uwsm_hyprland.desktop: Starting: /usr/bin/start-hyprland`)
- 06:44 - both monitors running on DP-8 1080p@60 and DP-9 1080p@144, scale
  1.25, same boot (`uptime -s` still 06:35:51)

## Root cause

Partly established.

- **Trigger: the dock resets.** On 2026-10-02 pulling its power cable reset
  the whole dock (all its USB devices dropped at 06:38:47). The kernel then
  tears down the MST connectors and creates new ones (DP-6 / DP-7 ->
  DP-8 / DP-9). On 2026-10-01 the same re-numbering happened; what reset the
  dock that time is not known (see Unverified).
- **Failure: the running compositor cannot use the new connectors.** Every
  atomic commit aquamarine builds for DP-8 / DP-9 is rejected with `EINVAL`,
  for all 31-36 modes including 640x480 and 720x400, so it is not a bandwidth
  or mode problem.
- **The stuck state is in the running Hyprland / aquamarine instance.** A new
  Hyprland in the same boot drove the very same DP-8 / DP-9 connectors at
  once. What exactly is stale (CRTC assignment, a cached property ID, ...) is
  not known; the kernel's own state is reset when a new DRM master takes over,
  so a kernel-side part is not fully excluded.

## Evidence

Saved before recovering (`/run/user` is wiped on reboot and the log is
replaced on re-login):

- `~/.cache/hyprland/dock-dropout-2026-10-01-hyprland.log` and
  `...-2026-10-01-kernel.log` - first occurrence
- `~/.cache/hyprland/dock-dropout-2026-10-02-hyprland.log` and
  `...-2026-10-02-kernel.log` - power-cable occurrence

Dock reset on 2026-10-02, kernel log:

```
06:38:47 kernel: usb 3-3: USB disconnect, device number 2
06:38:47 kernel: usb 4-2: USB disconnect, device number 2
06:38:47 kernel: r8152-cfgselector 4-2.3: USB disconnect, device number 3
06:38:48 kernel: usb 3-3: New USB device found, idVendor=2109, idProduct=2817
06:38:48 kernel: usb 4-2: New USB device found, idVendor=2109, idProduct=0817
```

Hyprland log of the first occurrence (2026-10-01):

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

The kernel log at default level has no i915 / DRM error for this, on either
day; only the USB disconnect / reconnect of the dock.

After the re-login on 2026-10-02, same boot:

```
$ uptime -s
2026-10-02 06:35:51
$ hyprctl monitors -j | jq ...
DP-8 AW2521HFA 1920x1080@60.00000 scale=1.25
DP-9 LS24AG30x 1920x1080@143.99800 scale=1.25
```

## What didn't work

- **Unplug and replug the dock** (twice). Connectors come back as the same
  DP-8 / DP-9 and every modeset still fails.
- **`hyprctl reload`.** Re-applies monitor rules; one more failed commit.
- **Not tried on purpose:** `hyprctl dispatch dpms off/on`. It blanks the
  laptop panel too, and Hyprland is known to segfault on hotplug events while
  the panel is blanked (dotfiles `CLAUDE.md`, "Lock and idle").

## Resolution

Log out and back in. That restarts Hyprland and is enough (2026-10-02). A
reboot also works (2026-10-01) but is not needed.

## Unverified

- What reset the dock on 2026-10-01. That day all displays were blanked and
  woken just before the drop and the owner saw "dark on mouse move", so idle
  blanking was the first suspect. The 2026-10-02 case had no blanking at all,
  so blanking is at most one of several ways to reset the dock, not the cause
  of the stuck state.
- That the dock resets *because* it loses its power input. It fits (it
  carries the laptop's charging and has no other supply), but only the
  timing is shown.
- Whether this is a known Hyprland / aquamarine bug. Upstream issues were
  not searched yet.
- The dock model; `lsusb` is not installed. USB IDs from the kernel log:
  hub `2109:2817` / `2109:0817`, Ethernet `0bda:8153`.

## Prevention

No fix yet; a workaround and a habit.

- Pull or plug the dock's power cable only while the laptop is off or
  suspended.
- If the monitors stay black after a dock reset: log out and back in
  (`omarchy logout`, which runs `uwsm stop`). Do not bother replugging the
  dock, `hyprctl reload` or `omarchy restart hyprctl` (the same reload).
- If it happens again, save `$XDG_RUNTIME_DIR/hypr/*/hyprland.log` before
  logging out (the re-login replaces it) and note what happened right before.

Open: search Hyprland / aquamarine upstream issues for MST connectors that
cannot be modeset after a hotplug, and report it with the saved logs if it is
not known. That needs the owner's go-ahead.
