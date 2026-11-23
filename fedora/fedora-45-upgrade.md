# Fedora 44 → 45 upgrade (2026-11-23)

Workstation (`fedora`) went from Fedora 44 to Fedora 45 on 2026-11-23. Fedora 45
shipped 2026-11-03 after two slipped target dates. Result: Fedora 45, kernel
`7.3.4-300.fc45`, NVIDIA 620.70.06 (akmod rebuilt, RTX 4090 works), no failed units.

Order of events, what broke, and what is still open. Change-register rows live in
[[fedora/system-changes]].

## Why we looked

Crash reports on the box. `coredumpctl` showed `kwin_wayland` SIGSEGV on 2026-11-20
(14:51) and twice on 2026-11-23 (13:49:51, 13:49:53), ~25 s after boot each time.
Also `openrgb` SIGABRT on 10-28, 11-12 and 11-18 (not investigated).

## kwin startup crash (not fixed by the upgrade)

- Same stack every time (F44 and F45):
  `LibInput::Connection::processEvents → InputDevice::pointerMotion →
  PointerInputRedirection::processMotion → InputRedirection::syncTabletWithMouse`.
  It fires while `InputRedirection` is still being constructed: a pointer-motion event
  arrives before the singleton exists and a null pointer is dereferenced.
- First seen four days after `kwin-6.8.3-1.fc44` was installed (2026-11-16).
- **Not** the GPU or drivers: NVIDIA module vermagic matched the kernel, amdgpu
  initialised normally.
- Upstream: KDE bug **528824**, "kwin crashed in KWin::InputRedirection::syncTabletWithMouse
  when logging out or logging in to Plasma 6.8.3" — RESOLVED FIXED, fixed in **6.8.4**,
  12 duplicates. The cause is the 6.8.3 change "backends/libinput: list devices during
  setup"; the fix is "input: only init backends after InputRedirection" (cherry-picked
  to the Plasma/6.8 branch). Bug page read via a fetch agent, part of it as a summary;
  the 6.8.4 release tag itself was not verified.
- As of 2026-11-23, F44 and F45 repos (including `updates-testing`) both carry only
  kwin 6.8.3. The first F45 login (14:56:21) crashed the same way; kwin auto-restarted
  and the session was stable afterwards.
- **Action:** none. `dnf upgrade` picks up 6.8.4 when Fedora packages it. No new bug
  report needed.

## The upgrade

- Snapshots: snapper **#1895** ("pre-F45 upgrade") and **#1897** ("pre-F45 upgrade
  (attempt 2)"). Note: grub-btrfs rollback has black-screened on this box before; do not
  rely on it for display problems.
- Method: `dnf system-upgrade download --releasever=45`, then `dnf5 offline reboot -y`.
- Transaction: 3840 upgrades, 61 installs, 8 removals, 11 downgrades, ~10 GiB.
  Removals: kernel `7.2.15-200.fc44` (+ its hand-installed `kmod-nvidia`) and
  `python3-simpleaudio`. Downgrades to F45's older builds include blender (5.2.4 → 5.2.2)
  and hplip.

### Attempt 1 failed (boot-time solver, ~4 s)

`/etc/dnf/dnf.conf` had `excludepkgs=kf6-kimageformats plasma-discover-snap`. The pinned
6.25 needs `libOpenEXR-3_2`; F45 ships openexr 3.4, which produced hundreds of
conflicts across Qt, KDE, ImageMagick and plasma-workspace.

**Gotcha:** `--setopt=disable_excludes=main` on the *download* command made the
download-time dry run resolve cleanly, but the offline boot service re-reads
`/etc/dnf/dnf.conf` and ignores that override, so it failed. The override must be in the
config file itself. Check with `dnf5 offline log` and
`journalctl -b -1 -u dnf5-offline-transaction`.

### Attempt 2 succeeded

- Removed `kf6-kimageformats` from `excludepkgs` in `/etc/dnf/dnf.conf` (backup:
  `/etc/dnf/dnf.conf.bak-20261123-f45`). `plasma-discover-snap` stays excluded.
- `dnf5 offline clean`, re-download, reboot into the offline upgrade.
- **Risk reinstated:** kimageformats is no longer pinned. The reason for the pin was
  6.28 segfaulting `kioworker`/`QImageReader` on HEIC decode (libheif too old). F45 ships
  6.31/6.32; whether that is fixed was **not tested**. Test:
  `QImageReader` on `~/iphone_export/images/*.HEIC`, or open a HEIC in Dolphin.

## Post-upgrade fallout

### Plasma session came up half-started

`openrgb-verify.service` (user unit) hung "starting" for ~14 minutes and everything
queued behind it never started: powerdevil, polkit agent, xdg-desktop-portal, ssh-agent,
Solaar, baloo, `system-monitor-rgb`, `graphical-session.target`. Cause: Python went
3.14 → 3.15, so `from openrgb import OpenRGBClient` failed (`ModuleNotFoundError`);
the verifier saw 0 devices and kept restarting openrgb-server until its own 960 s
timeout. Fix: `python3 -m pip install --user openrgb-python`. Once the verifier
finished, the session targets came up by themselves.

`plasma-workspace.target` refuses manual start; it can only be pulled in by a dependency.
Use `systemctl --user list-jobs` to find the blocking job.

### Python 3.14 is gone

- `system_monitor_rgb.py` crash-looped (restart counter 84): missing `pynvml`.
  Fixed with `pip install --user nvidia-ml-py`; it now connects to 4 devices.
- `yt-dlp` reinstalled with `pip install --user yt-dlp` (wrapper shebang is
  `/usr/bin/python3`).
- **MemPalace** (MCP server and recall hook) did not connect: its package lived in
  `~/.local/lib/python3.14`. Rebuilt in a **Python 3.12 venv**,
  `~/.local/share/mempalace-venv` (`mempalace` 3.5.0, chromadb 1.5.9); the wrappers
  `~/.local/bin/mempalace` and `mempalace-mcp` now start with that venv's python.
  Backups: `~/.local/bin/*.bak-py314`. Palace data untouched (27,270 drawers).
  Venv on 3.12 on purpose, so the next system Python bump does not break it.
- `~/.local/lib/python3.14` (8.3 GB: torch, diffusers, transformers, chromadb, …)
  was **deleted**. ComfyUI is no longer on disk. Nothing running referenced it.
- ~20 other wrappers in `~/.local/bin` (black, isort, pytest, alembic, fastapi, uvicorn,
  hf, chroma, torchrun, …) still have a dead `#!/usr/bin/python3` shebang. Not rebuilt:
  nothing running or scheduled calls them. Reinstall on demand.
- Virtualenvs (hermes-agent, ChatDev, vllm, unsloth studio, piper-tts,
  comfyui-mcp-server, AUTOMATIC1111, logilinux) were checked and still run.
  Stdlib-only hooks and services (wiki-hub, recall/record hooks, TTS hook) were
  unaffected.

## GitHub SSH key replaced

The old key `~/.ssh/id_ed25519_github` (passphrase-protected, passphrase lost) could not
be loaded into ssh-agent. Generated `~/.ssh/id_ed25519_github2` (ed25519, **no
passphrase** so it can be used unattended), added it on GitHub, and pointed the
`github.com` entry in `~/.ssh/config` at it with `IdentitiesOnly yes`
(backup: `~/.ssh/config.bak-20261123-github2`).
Add a passphrase later with `ssh-keygen -p -f ~/.ssh/id_ed25519_github2` (then
`ssh-add` is needed each login). The old key was left on the GitHub account until the
owner deletes it.

Pushed this session: `lm-model-manager` (`bfec42f`, private) and `docs` → public
`homelab-wiki` (13 commits, `2dd7c65..f8f8dfb`). Anything uncommitted in `docs` at that
time was not pushed.

## Still open

- kwin startup crash can recur on any login until kwin 6.8.4 lands.
- kimageformats/HEIC thumbnails untested on the unpinned version.
- Dead `~/.local/bin` wrappers and the openrgb SIGABRTs (10-28, 11-12, 11-18) not investigated.
- Old GitHub key still on the account.

## Revert / roll back

Snapper #1895 / #1897 (see caveat above). Config: restore `/etc/dnf/dnf.conf.bak-20261123-f45`
(this re-pins kimageformats; only sensible if staying on F44).
