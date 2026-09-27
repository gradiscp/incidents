# 2026-09-25 - systemd-oomd killed the terminal running Claude Code, three times

**Status:** Mitigated
**Area:** Claude Code (parallel subagents), systemd-oomd, portfolio build (Astro)

## Summary

A Claude Code session split a large portfolio refactor across several
background subagents, each in its own git worktree, each running `npm ci`,
full Astro builds (`astro check` + `astro build`) and headless Firefox for
screenshot comparisons. Everything they spawned lived in the same cgroup as
the terminal. RAM and swap filled up (over 90% of swap), and systemd-oomd killed that whole
scope - the terminal window, Claude Code and every agent in it - three times
within half an hour. Work already committed was safe; the agents' uncommitted
work stayed in their worktrees.

## Impact

- The terminal and the Claude Code session disappeared three times
  (19:18, 19:24, 19:46), each time with all running agents.
- Two agents (font subsetting, comment/README trim) lost their in-flight state
  and had to be restarted; their partial edits survived in their worktrees.
- No committed work lost. The owner had to reopen the session each time.

## Environment

| Component | Detail |
|---|---|
| Machine | 15 GiB RAM, 30 GiB swap (5 GiB already in use) |
| systemd-oomd | `app.slice` monitored; swap-used limit 90% (the trigger here), memory-pressure limit 50% for 20s (from `oomctl`) |
| Terminal | launched via `xdg-terminal-exec`, one scope under `app.slice` |
| Workload | Claude Code + 2-3 background subagents, per agent: own worktree + `node_modules`, `npm run build` (runs `astro check`, a TypeScript language server), headless Firefox via WebDriver BiDi, python http.server |
| Also running | the owner's desktop Firefox and an `astro dev` server |

## Timeline

Times from `journalctl --user` unless marked.

- ~18:00 (est.) - three subagents started in parallel, each in its own worktree
- 19:18:11 - `systemd-oomd killed 1798 process(es)` in the terminal scope
- 19:24:27 - after restarting agents: `killed 1453 process(es)`
- ~19:45 (est.) - two agents resumed with "at most one headless Firefox at a time"
- 19:46:58 - `killed 1463 process(es)` again
- ~20:00 - agents stopped; remaining work run one agent at a time, heavy
  commands in their own scope

## Root cause

systemd-oomd killed the terminal's scope because RAM and swap were both
exhausted: its own log gives the reason as memory used 16.26 of 16.42 GB and
swap used 29.56 of 32.84 GB, "more than 90.00%" (the swap-usage trigger, not
the pressure trigger). It picks the child cgroup of `app.slice` using the most
swap - the terminal scope, with 22-24 GB swapped out at each kill. Every
process a Claude Code agent starts is a descendant of the terminal, so all of
it is accounted to that scope, and the kill takes the terminal and Claude Code
with it.

What inside the scope grew to 20+ GB is not proven (see Unverified). Parallel
agents with TypeScript-server builds, `npm ci` per worktree and headless
Firefox account for several GB; the suspected remainder is leftover headless
Firefox instances and servers that were never cleaned up, because a
`pkill -f` meant for the http server also killed the shell running it.

## Evidence

```
Sep 25 19:18:11 systemd[999]: app-Hyprland-xdg\x2dterminal\x2dexec-977517a3.scope: systemd-oomd killed 1798 process(es) in this unit.
Sep 25 19:24:27 systemd[999]: app-Hyprland-xdg\x2dterminal\x2dexec-e38010c5.scope: systemd-oomd killed 1453 process(es) in this unit.
Sep 25 19:46:58 systemd[999]: app-Hyprland-xdg\x2dterminal\x2dexec-ea541d95.scope: systemd-oomd killed 1463 process(es) in this unit.
```

systemd-oomd's own reason (`journalctl -u systemd-oomd`), same at all three kills:

```
19:46:58 systemd-oomd[726]: Considered 21 cgroups for killing, top candidates were:
    Path: …/app.slice/app-graphical.slice/app-Hyprland-xdg\x2dterminal\x2dexec-ea541d95.scope
        Swap Usage: 22G
19:46:58 systemd-oomd[726]: Marked …ea541d95.scope for killing due to memory used (16260780032) / total (16419115008) and swap used (29560922112) / total (32838500352) being more than 90.00%
```

(19:18: 24.2G swap in the scope; 19:24: 22.2G.) `oomctl` afterwards: swap
used limit 90.00%; 5.1 GiB of swap in use when idle.

`systemd-run --user --scope -- sh -c 'cat /proc/self/cgroup'` →
`.../app.slice/run-p…scope`: a command started this way gets its own scope,
separate from the terminal's.

## What didn't work

- Restarting the agents unchanged: killed again six minutes later.
- "At most one headless Firefox at a time": killed again; parallel builds were
  enough on their own.

## Resolution

- Stopped all parallel agents. Committed work was intact; partial work was
  taken from the agents' worktrees.
- Remaining work runs one agent at a time, with builds and headless browsers
  started through `systemd-run --user --scope`.

## Unverified

- Which processes inside the terminal scope held the 20+ GB. oomd logs only the
  cgroup. Suspected: accumulated headless Firefox instances and servers left
  behind, on top of parallel builds. Check next time with
  `systemd-cgls --user-unit <scope>` and `ps` while agents run.
- That a separate `systemd-run` scope reliably becomes oomd's victim instead of
  the terminal. It is its own child of `app.slice`, so it is a candidate, but
  the choice depends on reclaim counters - not yet observed under pressure.

## Prevention

- **Agent rule of thumb:** on this 15 GiB laptop, never run more than one
  subagent that builds or drives a browser. Split work by time, not by
  parallel agents.
- **Isolate heavy commands:** start builds and headless browsers with
  `systemd-run --user --scope --quiet -- <cmd>` so an oomd kill hits that
  scope instead of the terminal running Claude Code (oomd picks the cgroup
  with the most swap, so the heavy scope should be the one chosen).
- **Clean up:** stop servers by PID; `pkill -f <pattern>` also matches the
  shell running the command and killed it (exit 143/144).
- Where it acts: this note; the agent briefs in the session that follow it.
  Not yet encoded in a config or script - candidate for a Claude Code rule in
  the dotfiles (`config/claude/rules/`) if it recurs.
