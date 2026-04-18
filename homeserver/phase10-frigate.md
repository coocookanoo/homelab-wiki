> **Wiki Notice:** Project documentation should be copied to `/home/adminbill/projects/docs/<project-name>/` and pushed to [homelab-wiki](https://github.com/coocookanoo/homelab-wiki) so it appears at https://wiki.billsserver1.duckdns.org

---

# Phase 10 — Frigate NVR + Hikvision Camera

## Goal

Run Frigate on `server2` as the NVR and move the Hikvision PoE camera onto the
homeserver LAN so `server2` can reach it directly on `10.0.0.0/24`.

## Current Status

Updated `2026-04-16`.

- `server2` running Frigate + Forgejo at `10.0.0.145`
- Frigate live, software decode (VAAPI disabled — see below)
- **All 4 cameras verified live** — jsmpeg streams and image fetches all returning 200
  - `/live/jsmpeg/hikvision_cam1` through `hikvision_cam4` active in logs
  - No ffmpeg crash lines in steady-state log window
- Per-camera stream split enabled on `2026-04-16`:
  - `Channels/102` for `detect`
  - `Channels/101` for `record`
- CPU usage dropped after the stream split:
  - before: about `495.89%` CPU and `1.722 GiB` RAM
  - after settling: about `329.49%` CPU and `873.8 MiB` RAM
- Frigate stack at `/home/adminbill/frigate/`
- Frigate recording storage: 3.54T LVM volume at `/home/adminbill/frigate/storage`
- Frigate admin user: `admin` (password set during install)
- `/dev/dri/renderD128` now accessible — discrete GPU removed `2026-04-16`, iGPU active
- i965 VAAPI not yet tested — pending SMART test result on `sdb` before next changes

## Current Network Layout

| Device | IP | Role |
|---|---|---|
| `server1` | `10.0.0.1` | Router/DNS/DHCP for the LAN |
| `server2` | `10.0.0.145` | NVR host (Frigate + Forgejo) |
| ADTRAN switch | `10.0.0.2` | PoE switch, static |
| cam1 | `10.0.0.201` | Hikvision DS-2CD2112F-I, static |
| cam2 | `10.0.0.202` | Hikvision DS-2CD2112F-I, static |
| cam3 | `10.0.0.203` | Hikvision DS-2CD2112F-I, static |
| cam4 | `10.0.0.204` | Hikvision DS-2CD2112F-I, static |

## Bootstrap History (archived)

These findings are from the initial setup on `2026-04-16` and are kept for
reference only. All cameras are now live on `10.0.0.0/24`.

- Cameras came from a previous ZoneMinder install with static IPs on `192.168.1.x`
- They were not on the intended `10.0.0.0/24` subnet and were not visible to `server2`
- Discovery required temporarily connecting the ISP modem cable to the switch so
  Fedora (192.168.1.x) could reach the cameras and reconfigure them
- After reconfiguration, all 4 cameras were assigned static IPs on `10.0.0.201–204`
  and added to Frigate on `server2`

## Frigate Docs Notes

Checked against the official Frigate docs on `2026-04-15`.

### Installation

Frigate recommends Docker Compose as the default install method. The standard
image for an amd64 host is:

- `ghcr.io/blakeblackshear/frigate:stable`

Frigate's Docker docs also show these ports as the main defaults:

- `8971` for the web UI
- `8554` for RTSP restreaming
- `8555/tcp` and `8555/udp` for WebRTC

Docs:
- https://docs.frigate.video/frigate/installation/

### Camera Configuration

Frigate recommends using separate inputs where possible:

- a lower-resolution stream for `detect`
- a higher-resolution stream for `record`

Each camera input can be assigned one role per camera stream. For a Hikvision
camera, the likely shape will be:

```yaml
cameras:
  driveway:
    ffmpeg:
      inputs:
        - path: rtsp://admin:PASSWORD@10.0.0.201:554/Streaming/Channels/102
          roles:
            - detect
        - path: rtsp://admin:PASSWORD@10.0.0.201:554/Streaming/Channels/101
          roles:
            - record
```

Docs:
- https://docs.frigate.video/configuration/cameras/
- https://docs.frigate.video/configuration/reference/

### go2rtc

Frigate's bundled `go2rtc` is optional, but the docs recommend it when you want:

- better live view than the basic jsmpeg stream
- audio in live view
- RTSP restreaming to reduce direct camera connections

The docs specifically recommend matching the `go2rtc` stream name to the camera
name in Frigate.

Docs:
- https://docs.frigate.video/guides/configuring_go2rtc/

### ONVIF / Hikvision

Frigate can use ONVIF for PTZ and camera control, typically with:

- `host`
- `port`
- `user`
- `password`

The Frigate docs also note that some Hikvision models have incomplete ONVIF
support. That matters mostly for PTZ and autotracking, not basic RTSP ingest.

For a non-PTZ or basic recording setup, RTSP is the main requirement.

Docs:
- https://docs.frigate.video/configuration/cameras/

## Next Steps

1. Add `go2rtc` for better live view now that the per-camera stream split is stable
2. Investigate i965 VAAPI in the Frigate container on i7-3770 (iHD not supported on Gen 7 — see VAAPI section)
3. Add cam5 (`10.0.0.205`) and cam6 (`10.0.0.206`) when hardware is ready — see procedure below

## server2 Stack Layout

Frigate should live in its own stack on `server2`, separate from Forgejo.

### Repo-tracked source files

- `configs/frigate/docker-compose.yml`
- `configs/frigate/config.yml`

### Runtime paths on server2

- Preferred long-term:
  - `/data/frigate/docker-compose.yml`
  - `/data/frigate/config/config.yml`
  - `/data/frigate/storage/`
- Current live bootstrap on `2026-04-15`:
  - `/home/adminbill/frigate/docker-compose.yml`
  - `/home/adminbill/frigate/config/config.yml`
  - `/home/adminbill/frigate/storage/`

### Codex bootstrap on `server2`

Verified live on `2026-04-16`.

- CLI install path:
  - `/home/adminbill/.npm-global/bin/codex`
- Installed version:
  - `codex-cli 0.121.0`
- User shell PATH bootstrap:
  - `~/.bashrc` includes
    `export PATH="$HOME/.local/bin:$HOME/.npm-global/bin:$PATH"`
- Codex state copied during bootstrap:
  - `/home/adminbill/.codex/auth.json`
  - `/home/adminbill/.codex/config.toml`
- Codex state intentionally **not** copied:
  - `/home/adminbill/.codex/memories/`
  - history, logs, sessions, and other local state

### Reaching Codex on `server2`

From the Fedora workstation, the confirmed working path on `2026-04-16` is:

```bash
ssh -F /dev/null adminbill@100.64.61.90
ssh adminbill@10.0.0.145
codex
```

### Bootstrap notes

- Stack was initially deployed to `/home/adminbill/frigate/` (sudo not needed)
- VAAPI was attempted with `preset-vaapi` and `i965` driver — failed on i7-3770 in Docker
- Final working config uses CPU-only decode (`hwaccel_args: []`)
- Move to `/data/frigate/` is still pending

## Camera Configuration

**Model:** Hikvision DS-2CD2112F-I (all cameras, same model)
**Default IP (factory):** `192.0.0.64` (static — no DHCP on this model)
**Login:** admin / 12345 (factory default)
**RTSP streams:**
- Mainstream / record: `rtsp://admin:PASSWORD@10.0.0.20x:554/Streaming/Channels/101`
- Substream / detect: `rtsp://admin:PASSWORD@10.0.0.20x:554/Streaming/Channels/102`

**Important:** These cameras came from a previous ZoneMinder install and had static IPs
on `192.168.1.x`. Factory reset does NOT switch them to DHCP — they go back to `192.0.0.64`.
To reconfigure, you must either:
1. Add a `192.0.0.x` alias to a NIC on the same switch and connect via HTTP
2. Use Hikvision SADP tool (Windows) for layer-2 broadcast discovery

**To add more cameras (full procedure):**

1. **Plug ISP modem cable into the switch** — needed so Fedora (192.168.1.x) can reach cameras on their factory/old static IPs
2. **Plug camera into switch** — wait for link light
3. **Find the camera IP** — scan from Fedora:
   ```bash
   nmap -sn 192.168.1.0/24
   ```
   Cameras from this batch came up on `192.168.1.64` or `192.168.1.70` (old ZoneMinder IPs). Factory fresh cameras use `192.0.0.64` — add alias first:
   ```bash
   sudo ip addr add 192.0.0.100/24 dev enp14s0
   ```
4. **If camera has unknown credentials** — factory reset: hold pinhole reset button during power-on for 10 seconds. Returns to `192.168.1.64` with `admin` / `12345`
5. **If two cameras share the same IP** — unplug one, configure the other, then swap
6. **Log into camera web UI** — `http://192.168.1.64` (or IP found in scan)
   - Login: `admin` / `12345`
   - Go to **Configuration → Network → TCP/IP**
   - Disable DHCP, set static IP (next in `10.0.0.205`, `10.0.0.206` etc.)
   - Subnet: `255.255.255.0`, Gateway: `10.0.0.1`, DNS: `10.0.0.1`
   - Save — camera reboots
7. **Unplug ISP modem cable from switch** — not needed once camera is on 10.0.0.x
8. **Add camera to Frigate config** on server2 at `/home/adminbill/frigate/config/config.yml` — duplicate an existing camera block, rename, update IP, keep `102` for `detect` and `101` for `record`
9. **Restart Frigate:**
   ```bash
   cd /home/adminbill/frigate && docker compose restart frigate
   ```
10. **Verify in Frigate UI** — tunnel from Fedora: `ssh -L 8972:10.0.0.145:8971 ubuntuserver -N` then `https://localhost:8972`

**Camera IP assignments:**
| Camera | IP | Status |
|---|---|---|
| cam1 | 10.0.0.201 | Live |
| cam2 | 10.0.0.202 | Live |
| cam3 | 10.0.0.203 | Live |
| cam4 | 10.0.0.204 | Live |
| cam5 | 10.0.0.205 | TBD |
| cam6 | 10.0.0.206 | TBD |

**To find unknown camera physical location:**
```bash
arp -n 192.168.1.x        # get MAC address from Fedora
# then check switch MAC table at http://10.0.0.2 (tunnel from Fedora)
# or: ssh to switch and run: show mac address-table
```

---

## VAAPI Hardware Decode — Known Issue

The Frigate config originally used `hwaccel_args: preset-vaapi` with `LIBVA_DRIVER_NAME: i965`.
This **fails** on the i7-3770 (Ivy Bridge) in Docker:

```
libva: /usr/lib/x86_64-linux-gnu/dri/i965_drv_video.so init failed
Failed to initialise VAAPI connection: -1
```

**Root cause:** i965 driver is not compatible with the i7-3770 inside the Frigate container.
The iHD driver may work but requires additional setup.

**Current workaround:** Hardware decode disabled, running CPU-only:
```yaml
ffmpeg:
  hwaccel_args: []
```

4 CPU threads allocated to the detector. After moving detection to Hikvision substreams,
the i7-3770 dropped from roughly `495.89%` CPU / `1.722 GiB` RAM to about
`329.49%` CPU / `873.8 MiB` RAM with 4 cameras live.
Monitor CPU usage — iHD VAAPI may still be worth revisiting before adding more cameras.

---

## Frigate Config (current working)

Location: `/home/adminbill/frigate/config/config.yml` on server2

```yaml
mqtt:
  enabled: false

ffmpeg:
  hwaccel_args: []

detectors:
  cpu1:
    type: cpu
    num_threads: 4

cameras:
  hikvision_cam1:
    ffmpeg:
      inputs:
        - path: rtsp://admin:PASSWORD@10.0.0.201:554/Streaming/Channels/102
          roles:
            - detect
        - path: rtsp://admin:PASSWORD@10.0.0.201:554/Streaming/Channels/101
          roles:
            - record
    detect:
      enabled: true
      width: 640
      height: 360
      fps: 5
    record:
      enabled: true
      alerts:
        retain:
          days: 14
      detections:
        retain:
          days: 14
      continuous:
        days: 0
      motion:
        days: 7
    snapshots:
      enabled: true
      retain:
        default: 14

record:
  enabled: true

objects:
  track:
    - person
    - car
```

**To add more cameras:** duplicate the `hikvision_cam1` block, rename it, update the IP, keep `102` for `detect` and `101` for `record`.

---

## Accessing Frigate from Fedora

Frigate is on the LAN (10.0.0.x) — not directly reachable from Fedora (192.168.1.x).
SSH tunnel required:

```bash
ssh -L 8971:10.0.0.145:8971 ubuntuserver -N
```

Then browse to `https://localhost:8971`

---

## Pending

- [ ] server2 storage expansion: new disk still not detected; `rescan-scsi-bus.sh` found `0 new or changed device(s)` and repeated checks only showed `sda` on `2026-04-16`
- [ ] Move Frigate stack from `/home/adminbill/frigate/` to `/data/frigate/`
- [ ] Investigate iHD VAAPI for hardware decode on i7-3770
- [ ] Add go2rtc for better live view
- [x] Add second RTSP input per camera (substream for detect, mainstream for record)
- [x] All 4 cameras configured and streaming
- [x] ISP modem cable removed from switch
- [x] Frigate admin password set
