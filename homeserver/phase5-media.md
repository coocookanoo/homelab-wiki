# Phase 5 — Media Stack (Jellyfin + Radarr + Sonarr + qBittorrent)

## Overview

Automated media pipeline: find → download via VPN → import → stream.

```
Prowlarr (indexers)
    │
    ├── Radarr (movies) ──┐
    └── Sonarr (TV)    ──┤──► qBittorrent (via Gluetun/Mullvad VPN)
                          │         │
                          │    /data/media/downloads/
                          │         │
                          └── auto-import on completion
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
            /data/media/movies/             /data/media/tv/
                    │                               │
                    └───────────────┬───────────────┘
                                    │
                              Jellyfin (stream)
```

## Storage Layout

All media lives under mergerfs mount `/data/media` (spans disk2, disk3, disk4):

| Path | Contents |
|---|---|
| `/data/media/movies` | Radarr-managed movie library |
| `/data/media/tv` | Sonarr-managed TV library |
| `/data/media/downloads` | qBittorrent download directory |

Physical location: `/data/disk2/movies`, `/data/disk2/tv`, `/data/disk2/downloads` — visible via mergerfs at `/data/media/`.

## Radarr (Movies) — http://10.0.0.1:7878

### Critical Settings
- **Settings → Media Management → Use Hardlinks instead of Copy**: **DISABLED**
  - Hardlinks fail across filesystems (ext4 disk2 → mergerfs/fuse). Copy mode works.
- **Root Folder**: `/data/media/movies` (container path via mergerfs mount)
- **Download Client**: qBittorrent — host `172.18.0.1`, port `8090`

### Container Volumes
| Host | Container |
|---|---|
| `/data/radarr/config` | `/config` |
| `/data/media` | `/data/media` |
| `/data/media/downloads` | `/downloads` |

### Auto-Import Pipeline
1. Radarr sends download to qBittorrent
2. qBittorrent saves to `/downloads` (= `/data/media/downloads`)
3. On completion, Radarr copies file to `/data/media/movies/<Movie (Year)>/`
4. Jellyfin picks it up on next library scan (automatic, within minutes)

## Sonarr (TV Shows) — http://10.0.0.1:8989

### Settings to Configure
- **Settings → Media Management → Use Hardlinks instead of Copy**: **DISABLE** (same reason as Radarr)
- **Root Folder**: `/data/media/tv`
- **Download Client**: qBittorrent — host `172.18.0.1`, port `8090`

### Container Volumes
| Host | Container |
|---|---|
| `/data/sonarr/config` | `/config` |
| `/data/media` | `/data/media` |
| `/data/media/downloads` | `/downloads` |

## Prowlarr (Indexer Manager) — http://10.0.0.1:9696

Manages torrent indexers and syncs them to Radarr and Sonarr automatically.
Add indexers here — they propagate to both arr apps.

## qBittorrent — http://10.0.0.1:8090

- Routes ALL traffic through Gluetun (Mullvad VPN) via `network_mode: service:gluetun`
- Web UI port exposed on Gluetun container, not qBittorrent directly
- Downloads land in `/downloads` = `/data/media/downloads` on host
- **Firewalled status is normal** — Mullvad removed port forwarding in 2023. Outbound connections to peers still work; downloads function but seeding is limited.

See [gluetun-vpn.md](gluetun-vpn.md) for VPN details.

## Jellyfin — http://10.0.0.1:8096

### Libraries
| Library | Container Path | Host Path |
|---|---|---|
| Movies | `/media/movies` | `/data/media/movies` |
| TV Shows | `/media/tv` | `/data/media/tv` |

> Note: Jellyfin mounts `/data/media` as `/media`. Do NOT add `/media/downloads` as a library — that's the raw downloads folder.

### Hardware Transcoding
Intel QuickSync enabled via `/dev/dri` device passthrough. Set in Jellyfin → Dashboard → Playback → Transcoding → Intel QuickSync Video.

## Troubleshooting

**Radarr import fails after download:**
- Check Settings → Media Management → Hardlinks is DISABLED
- Check root folder is `/data/media/movies` (not `/media/movies` or `/config/...`)

**Movie shows in downloads but not in Jellyfin:**
- Radarr didn't import it — check Radarr → Activity → Queue for errors
- Manually move to `/data/media/movies/<Movie Name (Year)>/` as fallback

**qBittorrent shows "Firewalled":**
- Expected — Mullvad doesn't support port forwarding. Downloads still work.

**Nothing downloading (qBittorrent stuck):**
- Check Gluetun is connected: `docker logs gluetun --tail 20`
- Restart if needed: `docker restart gluetun && docker restart qbittorrent`
- Verify VPN IP: `docker exec gluetun wget -qO- https://api.ipify.org`
