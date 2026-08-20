# rclone Google Drive mount — crash leaves a stale FUSE mount and locks itself out

**Status: fixed 2026-08-18, verified by reproducing the failure.**

## Problem

`~/GoogleDrive` was empty/unusable from 2026-08-16 16:44 until the 2026-08-18
11:32 reboot — roughly 43 hours, with a single 104-minute working window in the
middle. `systemctl --user status rclone-gdrive.service` showed it failing and
restarting endlessly.

Two *different* causes stacked up, which is why it looked like one long outage.

## Cause 1 — DNS outage (Aug 16 16:44 → Aug 17 13:22, 499 restarts)

```
ERROR+4: Failed to create file system for "gdrive:": couldn't find root directory ID:
  ... couldn't fetch token: Post "https://oauth2.googleapis.com/token":
  dial tcp: lookup oauth2.googleapis.com: no such host
```

Name resolution was dead, so rclone couldn't get an OAuth token and exited 1 on
every start. Benign in itself — it recovered when the network came back. Related
to the network work tracked in memory `project_netmode_panic_button`.

## Cause 2 — the actual bug (Aug 17 15:06, 1,158 restarts)

At 15:06:08 rclone aborted with a Go **runtime heap corruption** fatal error:

```
runtime: marked free object in span 0x7f4bc41980a8, elemsize=96 freeindex=0
  (bad use of unsafe.Pointer or having race conditions? try -d=checkptr or -race)
fatal error: found pointer to free object

goroutine 3 [running]:
runtime.(*mspan).reportZombies   → mgcsweep.go:893
runtime.(*sweepLocked).sweep     → mgcsweep.go:668
runtime.bgsweep
```

The GC background sweeper found a live pointer into an already-freed 96-byte
object and aborted the process. This is not an rclone error path — it's the Go
runtime detecting corrupted heap state. rclone 1.74.3-1.fc44, kernel
7.1.8-200.fc44, process idle and up 104 minutes (vfs cache: 2 objects, 25 KiB).

**Cause unresolved.** Either an rclone/bazil-fuse race or hardware. Evidence
against jumping to hardware: no MCE / machine-check / EDAC corrected-error
entries in the kernel log, and this is the only Go/heap-corruption crash among
the 15 ABRT dumps on the box (the rest are the known Firefox SIGSEGV/SIGSYS
cluster plus assorted one-offs). Not enough to blame RAM, not enough to clear it.
**A second Go process aborting this way is the signal worth acting on.**

Core dump: `/var/spool/abrt/ccpp-2026-08-17-15:06:09.715598-184217`.

## Why the crash turned into a 20-hour outage

The dead process left a stale FUSE mount on `~/GoogleDrive`. rclone refuses to
mount over it:

```
ERROR+4: Fatal error: failed to mount FUSE fs: directory already mounted,
         use --allow-non-empty to mount anyway: /home/adminbill/GoogleDrive
```

The unit had `ExecStop=/bin/fusermount3 -uz %h/GoogleDrive` — but **systemd does
not run `ExecStop` when restarting after the main process died.** So nothing ever
cleared the mountpoint, and `Restart=on-failure` + `RestartSec=10` retried 1,158
times over six hours (counter 500 → 1658) with a guaranteed-identical failure.

This is the general shape of the bug, not an rclone quirk: **any FUSE-mount unit
with `Restart=` and cleanup only in `ExecStop` can lock itself out this way.**

## Fix

In `~/.config/systemd/user/rclone-gdrive.service`:

```ini
[Service]
ExecStartPre=-/usr/bin/fusermount3 -uz %h/GoogleDrive   # leading - : ignore "nothing to unmount"
...
Restart=on-failure
RestartSec=10
RestartSteps=10          # exponential backoff 10s → 5min
RestartMaxDelaySec=5min  # instead of a 10s storm during outages
```

`ExecStartPre` clears any stale mount before each attempt. The leading `-` makes
a failure non-fatal, which is the normal case on a clean start (nothing mounted →
`fusermount3` exits 1 → ignored).

Backoff addresses cause 1: a future DNS outage walks 10s → 5min instead of
hammering every 10 seconds for hundreds of attempts.

Requires systemd 254+ for `RestartSteps`/`RestartMaxDelaySec`; this box is on 259.

## Verification

Reproduced the exact failure mode rather than assuming — `kill -9` on the main
PID leaves the same stale mount the crash did:

```
Process: 14776 ExecStartPre=/usr/bin/fusermount3 -uz /home/adminbill/GoogleDrive (code=exited, status=0/SUCCESS)
Main PID: 14778 (rclone)
Active: active (running)
```

`ExecStartPre` exited **0** — it really did clear the stale mount — and rclone
remounted ~10s later. `~/GoogleDrive` lists normally. On a clean start the same
step exits 1 and is ignored, so the normal path is unaffected.

Pre-fix unit backed up at `~/.config/systemd/user/rclone-gdrive.service.bak.20260818`.

## Follow-up

Any other FUSE-mount user unit on this box has the identical lockout bug and
wants the same `ExecStartPre` line. Not yet audited.
