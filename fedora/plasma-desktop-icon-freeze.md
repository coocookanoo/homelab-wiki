# Plasma desktop icons stop responding to clicks

**Status: workaround confirmed (5 occurrences). Root cause NOT identified.**
**Never reboot for this** — the fix takes ~2 seconds.

## Symptom

Desktop icons become completely unresponsive to mouse clicks — no selection, no
open, no visual reaction at all. Everything else keeps working: open application
windows, panels, KRunner, keyboard shortcuts. Only the desktop containment's
input handling is wedged.

plasmashell does **not** crash. It stays alive the whole time.

## Fix

```
fix-desktop-icons
```

Also in KRunner as **"Fix Desktop Icons"**. It wraps:

```
systemctl --user restart plasma-plasmashell.service
```

~2 seconds. Open application windows are untouched — only the desktop and panels
flicker. The script additionally records pre-restart state to
`~/.local/state/desktop-icon-freeze.log`, verifies the shell came back, and
sends a notification.

- Script: `~/.local/bin/fix-desktop-icons`
- KRunner entry: `~/.local/share/applications/fix-desktop-icons.desktop`
- Evidence log: `~/.local/state/desktop-icon-freeze.log`

## Occurrences

| # | Date | plasmashell uptime at freeze | Notes |
|---|------|------------------------------|-------|
| 1 | (first, undated) | — | Fix not yet known; needed a full reboot |
| 2 | 2026-08-08 | — | Restart-fix discovered |
| 3 | 2026-08-13 ~14:40 | 3h42m | Script written this day — but it logged **nothing**, see below |
| 4 | 2026-08-20 ~15:27 | 4h47m | Wayland, `~/GoogleDrive` FUSE mounted. First entry with real data |
| 5 | 2026-08-22 ~16:20 | 5h04m | Wayland, `~/GoogleDrive` mounted. Kernel 7.1.9, kwin **6.7.4-2**. First fully clean log entry |

All three measured cases are multi-hour: **3h42m, 4h46m, 5h04m** — a 3.5–5h
band, and increasing. Not enough points to call it a cause, but it is the only
consistent quantity across occurrences so far, and no short-uptime freeze has
ever been seen. In both #4 and #5 plasmashell's uptime was effectively the
session uptime (shell started at boot), so "plasmashell uptime" and "time since
login" are not yet distinguishable.

Frequency is increasing: #3 → #4 was 7 days, #4 → #5 was 2 days.

## Why there is no root cause yet

journald shows **nothing**: no crash, no error, no stop/restart events, no
coredump, no kwin errors, and no correlation with display changes, suspend, or
DPMS. plasmashell's own process is healthy across the freeze.

This is a silent input-handling wedge in the desktop containment — a known class
of Plasma 6 / Wayland bug that leaves no log trace. Because journald is a dead
end, the freeze log is the *only* remaining diagnostic avenue: it accumulates
comparable state snapshots across occurrences so a pattern can eventually be
spotted.

## The script was silently broken until 2026-08-20

Occurrence #3 recorded no evidence at all — the log contained only an error and
the string `restarted OK`. Two separate bugs:

1. **Counter aborted the whole evidence block.**
   ```bash
   echo "=== $(date -Is) : freeze #$(( $(grep -c '^=== ' "$LOG" 2>/dev/null || echo 0) + 1 )) ==="
   ```
   `grep -c` prints `0` *and* exits 1 when there are no matches, so `|| echo 0`
   appended a second `0`. `$(( ))` received `0\n0 + 1` and threw
   `arithmetic syntax error`. Under `set -euo pipefail` that killed every
   remaining command in the `{ ... }` block — so uptime, start time, session,
   mounts and outputs were all lost. Fixed:
   ```bash
   n=$(grep -c '^=== ' "$LOG" 2>/dev/null) || n=0
   echo "=== $(date -Is) : freeze #$(( n + 4 )) ==="
   ```
   The `+ 4` offset accounts for the 3 occurrences that happened before the log
   held its first real entry, so the numbering matches reality.

2. **Display names were never captured.**
   `kscreen-doctor -o` emits ANSI color codes, so `grep -E '^Output|enabled'`
   never matched the `Output:` lines — only bare `enabled` fragments wrapped in
   escape sequences got through. Fixed by stripping escapes first:
   ```bash
   kscreen-doctor -o 2>/dev/null | sed 's/\x1b\[[0-9;]*m//g' \
       | grep -E 'Output|enabled' | sed 's/^/  /'
   ```
   Now records `Output: 1 HDMI-A-2` / `Output: 2 DP-6` with their states.

Both fixes were verified by running the block against an empty log (the exact
case that failed before) and against a populated one — block exits 0 in both,
and the counter advances correctly.

Pre-patch copy kept at `~/.local/bin/fix-desktop-icons.bak.20260820`.

**Caveat:** entry #4 in the log was written by the pre-patch script seconds
before the patch, so its `outputs:` block is still ANSI-garbled. Its other
fields are fine. Entry #5 (2026-08-22) is the first fully clean one and does
record both display names — the patch works in the field, not just in test.

## Ruled out

- **kwin 6.7.4-2 does not fix it — confirmed empirically 2026-08-22.** On
  08-13 this was only a guess from the changelog ("apps possibly getting
  disconnected on Mesa 26+", unrelated to input handling). That update is now
  installed and occurrence #5 happened anyway. Closed.
- **Not a kernel issue in any obvious way.** #5 occurred on kernel
  7.1.9-200.fc44 after a fresh boot; earlier occurrences were on 7.1.7/7.1.8.
- **Not stale session state carried across a reboot.** #5 came 5h into a clean
  boot, with plasmashell started at login.
- **The rclone `~/GoogleDrive` FUSE mount is not implicated.** The
  `Invalid path data; path truncated` line logged at the exact minute the mount
  was added (2026-08-13 14:01) is a Qt **SVG** artwork warning about drawing
  paths, not a filesystem path — a spurious correlation. It has not been cleared
  as a cause either way; the freeze log records `$HOME` mounts specifically so a
  future occurrence can test it by unmounting first. It was mounted during #4.

## Next time it happens

1. Run `fix-desktop-icons` (or KRunner → "Fix Desktop Icons"). Do not reboot.
2. Read `~/.local/state/desktop-icon-freeze.log` and compare the new entry
   against previous ones — uptime, session type, mounts, outputs.
3. Worth trying once, before the restart, if you can spare the time: unmount
   `~/GoogleDrive` and see whether icons come back on their own. That is the one
   outstanding hypothesis with a cheap test.

Related: memory `project_plasma_desktop_icons_freeze`, [[fedora/system-changes]]
