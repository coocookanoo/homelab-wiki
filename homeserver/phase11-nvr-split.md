> **Wiki Notice:** Project documentation should be copied to `/home/adminbill/projects/docs/<project-name>/` and pushed to [homelab-wiki](https://github.com/coocookanoo/homelab-wiki) so it appears at https://wiki.billsserver1.duckdns.org

---

# Phase 11 — NVR Split (server1 → server2)

## Goal

Keep all camera and NVR workloads on `server2`, not `server1`.

- `server1` owns: router, firewall, DNS, VPN, reverse proxy, arr stack, media, app hosting
- `server2` owns: NVR (Frigate), Git hosting (Forgejo)

## Status — Complete

Updated `2026-04-16`.

Frigate was deployed directly to `server2` — there was no live Frigate instance on `server1`
to migrate. The NVR workload was never anchored on `server1`.

### Current live state on server2

- Frigate running at `10.0.0.145`
- All 4 cameras live and streaming (verified `2026-04-16`):
  - `hikvision_cam1` — `10.0.0.201`
  - `hikvision_cam2` — `10.0.0.202`
  - `hikvision_cam3` — `10.0.0.203`
  - `hikvision_cam4` — `10.0.0.204`
- CPU-only decode (VAAPI disabled — see [phase10-frigate.md](phase10-frigate.md))
- Split-stream layout active:
  - `Channels/102` for `detect`
  - `Channels/101` for `record`
- Measured impact after the stream split:
  - before: about `495.89%` CPU / `1.722 GiB` RAM
  - after settling: about `329.49%` CPU / `873.8 MiB` RAM
- Stack path: `/home/adminbill/frigate/`
- Frigate storage on dedicated LVM volume — see storage section below

### Storage — LVM on server2

Set up `2026-04-16`:

- `ubuntu-vg` spans `/dev/sda3` (1.8T, Seagate ST32000641AS) + `/dev/sdb1` (1.8T, WD WD2002FAEX)
- `ubuntu-lv` — 100G — root filesystem (`/`)
- `frigate-storage` — 3.54T — ext4, mounted at `/home/adminbill/frigate/storage`
- Mount is persistent via `/etc/fstab`
- Old recordings preserved at `/home/adminbill/frigate/storage.pre-lvm-2026-04-16`

### Disk health — sdb (WD WD2002FAEX) — DEAD, REMOVED

`2026-04-16`: sdb caused 3 server crashes shortly after install — kernel log showed:
```
ata2.00: SRST failed (errno=-16)
ata2.00: reset failed, giving up
```

SMART quick check: PASSED — but 1 `Current_Pending_Sector` and 1 `Offline_Uncorrectable` detected.
Drive has 30,117 power-on hours (~3.4 years). SMART long test confirmed read failure at LBA 1335461805 (70% completion).

`2026-04-17`: sdb physically removed. Server2 crashed on next boot — initramfs could not assemble ubuntu-vg (spanned sda3 + sdb1). `vgchange -ay --partial` allowed temporary boot but system was too degraded to recover cleanly.

**Resolution:** Fresh OS reinstall on sda. See below.

### server1 — no Frigate

Frigate is not running on `server1` and was never in production there.
No migration or cutover was required.

## server2 Rebuild — 2026-04-17

sdb removal caused unrecoverable LVM failure. Fresh install required.

### Rebuild steps completed (2026-04-17)
- [x] Booted Fedora 43 live USB — wiped old LVM off sda during install
- [x] Ubuntu Server 24.04.4 installed clean on sda only
- [x] Docker 29.4.0 installed
- [x] Codex CLI 0.121.0 installed (`~/.npm-global/bin/codex`)
- [x] Static IP set to `10.0.0.145` via netplan (`enp3s0`) — cloud-init network management disabled
- [x] Frigate container running (healthy) — stack at `/home/adminbill/frigate/`
- [x] SSH key (ed25519) generated on server2, authorized on server1 — keyless auth working
- [x] Persistent reverse SSH tunnel via autossh — server1 binds `0.0.0.0:2222` → server2:22
  - Fedora access: `ssh -p 2222 adminbill@192.168.1.34`
  - Service: `autossh-tunnel.service` (enabled, starts on boot)
  - Requires `GatewayPorts clientspecified` in server1 `/etc/ssh/sshd_config` — set 2026-04-17
- [x] Frigate VAAPI disabled — `hwaccel_args: []` (CPU-only, preset-vaapi fails on i7-3770)
- [x] Frigate admin account created — cameras visible in UI (2026-04-17)
- [x] All 4 cameras confirmed live in Frigate (2026-04-17)

### Rebuild steps remaining
- [x] Fix Frigate config — `hwaccel_args: []` (CPU-only decode)
- [x] All 4 cameras back online in Frigate
- [ ] Deploy Forgejo
- [ ] Create LVM frigate-storage volume on sda (sda3 has ~5.4T free in ubuntu-vg)
- [ ] Set up automated backup to USB drive on server1
- [ ] Try iHD VAAPI once Frigate is stable
- [ ] Add go2rtc for better live view

### Config backup location
All Frigate config is preserved on the Fedora workstation:
- `/home/adminbill/frigate/docker-compose.yml`
- `/home/adminbill/frigate/config/config.yml`

## Remaining Work

- [x] server2 storage expansion — done (wiped with sdb removal; new LVM volume pending)
- [x] Add substream input per camera (substream for detect, mainstream for record)
- [x] Rebuild server2 — fresh Ubuntu 24.04 install on sda (2026-04-17)
- [x] Docker + Codex installed, static IP set (2026-04-17)
- [x] Frigate running, VAAPI disabled, all 4 cameras live (2026-04-17)
- [ ] Deploy Forgejo
- [ ] Create LVM frigate-storage volume
- [ ] Set up backups to server1 USB drive
- [ ] Try iHD VAAPI
- [ ] Add go2rtc for better live view

See [phase10-frigate.md](phase10-frigate.md) for full Frigate config and camera procedure.
