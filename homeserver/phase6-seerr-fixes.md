> **Wiki Notice:** Project documentation should be copied to `/home/adminbill/projects/docs/<project-name>/` and pushed to [homelab-wiki](https://github.com/coocookanoo/homelab-wiki) so it appears at https://wiki.billsserver1.duckdns.org

---

# Phase 6 — Seerr, Download Pipeline Fixes & Jellyfin Import

## Overview

This phase added a media request manager (Seerr), fixed a broken download pipeline, resolved Jellyfin import issues, and expanded the indexer library.

---

## 1. Seerr (Media Request Manager)

**What it does:** Web UI where family members can browse and request movies/TV shows. Requests auto-approve and flow through to Radarr/Sonarr → qBittorrent → Jellyfin.

**Image:** `ghcr.io/seerr-team/seerr:latest`  
**Port:** `5055`  
**Config volume:** `/data/seerr/config:/app/config`

### docker-compose entry

```yaml
seerr:
  image: ghcr.io/seerr-team/seerr:latest
  container_name: seerr
  init: true
  environment:
    TZ: '${TZ}'
    LOG_LEVEL: info
  ports:
    - "5055:5055"
  volumes:
    - /data/seerr/config:/app/config
  restart: unless-stopped
```

### Configuration

Seerr config lives at `/data/seerr/config/settings.json`. Key settings:

| Setting | Value |
|---|---|
| Jellyfin host | `192.168.1.34:8096` (no SSL) |
| Radarr host | `radarr` (container name), port `7878` |
| Sonarr host | `sonarr` (container name), port `8989` |
| Auto-approve | `defaultPermissions: 96` (32=auto-approve requests + 64=request) |

**Radarr quality profile:** HD-1080p (profile ID `4`)  
**Sonarr quality profile:** HD-1080p (profile ID `4`)  
**Radarr root folder:** `/data/media/movies`  
**Sonarr root folder:** `/data/media/tv`

### Permissions fix

Config dir must be owned by uid 1000:

```bash
sudo chown -R 1000:1000 /data/seerr/config
```

---

## 2. FlareSolverr (Cloudflare Bypass for Prowlarr)

Allows Prowlarr indexers protected by Cloudflare to work.

```yaml
flaresolverr:
  image: ghcr.io/flaresolverr/flaresolverr:latest
  container_name: flaresolverr
  environment:
    LOG_LEVEL: info
    TZ: '${TZ}'
  ports:
    - "8191:8191"
  restart: unless-stopped
```

Configure in Prowlarr: **Settings → Indexers → Add FlareSolverr proxy** → URL `http://flaresolverr:8191`.

---

## 3. qBittorrent — Removed from Gluetun VPN

**Problem:** Mullvad VPN (Gluetun) does not support port forwarding, which caused zero peers on all torrents. qBittorrent was routed through Mullvad via `network_mode: service:gluetun`.

**Fix:** Removed qBittorrent from Gluetun network. It now runs directly on the host network with its own ports.

```yaml
# BEFORE (broken — no port forwarding through Mullvad)
qbittorrent:
  network_mode: "service:gluetun"
  depends_on:
    - gluetun
  # no ports section

# AFTER (working)
qbittorrent:
  ports:
    - "8090:8090"
    - "6881:6881"
    - "6881:6881/udp"
  # no network_mode
```

**Gluetun** remains running for its original purpose (future use / split tunneling) but qBittorrent no longer routes through it.

---

## 4. qBittorrent Credentials

**Username:** `adminbill`  
**Password:** `adminadmin`  
**WebUI port:** `8090`

> If the password ever gets wiped (e.g. after a config reset), stop the container, delete the `WebUI\Password*` lines from `/data/qbittorrent/config/qBittorrent/qBittorrent.conf`, restart, grab the temp password from `docker logs qbittorrent`, then set a permanent password via the API:
> ```bash
> curl -s -c /tmp/qb.txt -X POST "http://localhost:8090/api/v2/auth/login" -d "username=adminbill&password=TMPPASS"
> curl -s -b /tmp/qb.txt -X POST "http://localhost:8090/api/v2/app/setPreferences" -d 'json={"web_ui_password":"adminadmin"}'
> ```

---

## 5. Download Path Fix (Critical)

**Problem:** qBittorrent was saving to `/config/Downloads` inside its container (`/data/qbittorrent/config/Downloads/` on host). Radarr/Sonarr mount `/downloads` → `/data/media/downloads`, so they couldn't see the downloaded files.

**Fix applied:**

1. Changed qBittorrent default save path to `/downloads`:
   ```bash
   curl -s -b /tmp/qb.txt -X POST "http://localhost:8090/api/v2/app/setPreferences" \
     -d 'json={"save_path":"/downloads"}'
   ```

2. Added volume mount to Radarr and Sonarr so they can access the old path while in-progress downloads finish:
   ```yaml
   # In both radarr and sonarr services:
   volumes:
     - /data/qbittorrent/config/Downloads:/qbit_downloads
   ```

3. Added Remote Path Mapping in Radarr and Sonarr UI (or API):
   - **Host:** `qbittorrent`
   - **Remote Path:** `/config/Downloads/`
   - **Local Path:** `/qbit_downloads/`

4. Future downloads now save correctly to `/downloads` = `/data/media/downloads`.

---

## 6. Jellyfin Import — Manual Fix

When Radarr has completed files in the right folder but `hasFile` stays `false`, use:

```bash
# Trigger disk scan for a specific movie folder
curl -s -X POST "http://localhost:7878/api/v3/command?apikey=RADARR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name":"DownloadedMoviesScan","path":"/data/media/movies/Movie Name (Year)"}'

# Or rescan a specific movie by ID
curl -s -X POST "http://localhost:7878/api/v3/command?apikey=RADARR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name":"RescanMovie","movieId":3}'
```

If Radarr can't match a movie from `/downloads`, copy the file directly to its movie folder and rescan:

```bash
sudo mkdir -p "/data/media/movies/Movie Title (Year)"
sudo cp "/data/media/downloads/Movie.File.mkv" "/data/media/movies/Movie Title (Year)/"
# Then trigger RescanMovie via API
```

---

## 7. Pi-hole v6 — Custom DNS Hostname

Pi-hole v6 dropped support for `custom.list`. Use the `hosts` array in `pihole.toml` instead:

```toml
# /data/pihole/etc/pihole.toml
hosts = ["192.168.1.34 homeserver"]
```

This adds `homeserver` as a DNS name for the server, used by Homepage's `HOMEPAGE_ALLOWED_HOSTS`.

---

## 8. Prowlarr Indexers

22 indexers added including:

**General:**
- 1337x, RARBG Mirror, The Pirate Bay, EZTV, YTS, LimeTorrents, Nyaa.si, TorrentGalaxy, RuTracker

**Asian/Anime content (for family):**
- BakaBT, dmhy, ACG.RIP, AnimeTosho, Bangumi Moe, sukebei.nyaa.si, AvistaZ, M-Team TP

**Cloudflare-protected (via FlareSolverr):**
- 1337x, TorrentGalaxy (and others requiring browser challenge bypass)

---

## 9. Homepage Dashboard — Seerr Entry

Added to `/data/homepage/services.yaml` under the Media section:

```yaml
- Seerr:
    href: http://192.168.1.34:5055
    description: Media Requests
    icon: overseerr.png
```

---

## 10. Quality Profile Fix

Radarr quality profile IDs on this server:

| ID | Name |
|---|---|
| 3 | HD-720p |
| 4 | HD-1080p |
| 5 | Ultra-HD |

All movies should use profile **4 (HD-1080p)**. To fix a movie via API:

```bash
# Get movie, update qualityProfileId to 4, PUT back
MOVIE=$(curl -s "http://localhost:7878/api/v3/movie/ID?apikey=KEY")
UPDATED=$(echo "$MOVIE" | python3 -c "import json,sys; m=json.load(sys.stdin); m['qualityProfileId']=4; print(json.dumps(m))")
curl -s -X PUT "http://localhost:7878/api/v3/movie/ID?apikey=KEY" \
  -H "Content-Type: application/json" -d "$UPDATED"
```

---

## 11. Sonarr Download Client Credentials

If Sonarr/Radarr show `downloadClientUnavailable`, check the download client credentials match qBittorrent:

```bash
# Test connection
CLIENT=$(curl -s "http://localhost:8989/api/v3/downloadclient/1?apikey=SONARR_KEY")
curl -s -X POST "http://localhost:8989/api/v3/downloadclient/test?apikey=SONARR_KEY" \
  -H "Content-Type: application/json" -d "$CLIENT"
# Returns {} on success
```

---

## 12. Custom Formats — Block Incompatible Releases (Auto)

The Intel i5-6600 (Skylake) cannot hardware-decode 10-bit HEVC, HDR, Dolby Vision, AV1, or VP9. To prevent Radarr/Sonarr from ever grabbing these formats (which would cause high-CPU software transcoding or outright playback failure), Custom Formats were created via API with a score of **-10000** and applied to the HD-1080p quality profile in both Radarr and Sonarr.

### i5-6600 QuickSync Compatibility

| Format | Hardware Support |
|---|---|
| H.264 (AVC) | ✅ Full hardware decode/encode |
| H.265 SDR 8-bit | ✅ Hardware decode/encode |
| H.265 10-bit / HDR | ❌ Falls back to CPU |
| Dolby Vision / HDR10+ | ❌ Falls back to CPU |
| AV1 | ❌ Not supported |
| VP9 | ❌ Not supported |

### Custom Formats Created (Radarr IDs 1–4, Sonarr IDs 1–4)

| Name | Regex | Score |
|---|---|---|
| HDR | `\b(HDR10?(\+\|Plus)?\|DolbyVision\|DV\|PQ\|HLG)\b` | -10000 |
| 10bit | `\b(10.?bit\|10b\|hi10p)\b` | -10000 |
| AV1 | `\bAV1\b` | -10000 |
| VP9 | `\bVP9\b` | -10000 |

### Quality Profile Setting

Both Radarr and Sonarr **HD-1080p (id=4)** profiles have `minFormatScore: 0`. Any release scoring below 0 (i.e. any of the above formats) is automatically rejected. Standard H.264/H.265 SDR releases score 0 and are grabbed normally.

This means **Seerr users never need to think about formats** — incompatible releases are silently skipped and a compatible one is grabbed instead.

### API commands used to set this up

```bash
RADARR_KEY="a454b0a9ea4840dbab826c533fda67bf"
SONARR_KEY="726281afb19144fea8202e060e8e6782"
BASE="192.168.1.34"

# Create custom format (repeat for each name/regex, adjust port for Sonarr :8989)
curl -s -X POST "http://$BASE:7878/api/v3/customformat?apikey=$RADARR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name":"HDR","includeCustomFormatWhenRenaming":false,"specifications":[{"name":"HDR","implementation":"ReleaseTitleSpecification","negate":false,"required":false,"fields":[{"name":"value","value":"\\b(HDR10?(\\+|Plus)?|DolbyVision|DV|PQ|HLG)\\b"}]}]}'

# After creating all 4, GET the HD-1080p profile, add formatItems with score -10000,
# set minFormatScore=0, then PUT it back.
```

---

## Known Issues / Outstanding

| Item | Status |
|---|---|
| **The Running Man** | Downloaded as raw Blu-ray disc (BDMV/m2ts) — needs re-grab as MKV |
| **Warcraft** | Downloaded as 3D ISO — needs re-grab as standard MKV |
| **Moonlight Mystique** | Files downloaded as "Pursuit of Jade" — Sonarr can't match; needs manual rename |
| **Leo and Tig / The Haunted House** | No indexer results yet |
| **Creature Cases S02-S07** | Some episodes have 0 seeds on public trackers |

---

## Service URLs

| Service | URL |
|---|---|
| Seerr (request movies/TV) | http://192.168.1.34:5055 |
| Jellyfin | http://192.168.1.34:8096 |
| Radarr | http://192.168.1.34:7878 |
| Sonarr | http://192.168.1.34:8989 |
| qBittorrent | http://192.168.1.34:8090 |
| Prowlarr | http://192.168.1.34:9696 |
| Homepage | http://192.168.1.34:3000 |
