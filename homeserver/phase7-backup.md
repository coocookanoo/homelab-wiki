# Phase 7 — Backups

## Backup Destinations

Two backup targets:

1. **Local — disk3** (`/data/disk3/backups/`) — 3.6TB HDD in the server, always available
2. **Remote — Fedora PC** (`adminbill@192.168.1.33:/home/adminbill/backups/homeserver/`) — offsite copy on LAN

## Nightly Backup to Fedora (existing)

- **Script**: `/usr/local/bin/server-backup.sh`
- **Destination**: `adminbill@192.168.1.33:/home/adminbill/backups/homeserver/`
- **Log**: `/var/log/server-backup.log`
- **Schedule**: `0 2 * * *` (root crontab on server)

## What Gets Backed Up

| Source | Destination |
|---|---|
| `/etc/` | `backups/homeserver/etc/` |
| `/etc/wireguard/` | `backups/homeserver/wireguard/` |
| `/data/pihole/etc/` | `backups/homeserver/data/pihole/` |
| `~/homeserver/configs/docker/docker-compose.yml` | `backups/homeserver/data/docker/` |
| `/data/sonarr/config/` | `backups/homeserver/data/sonarr/` |
| `/data/radarr/config/` | `backups/homeserver/data/radarr/` |
| `/data/prowlarr/config/` | `backups/homeserver/data/prowlarr/` |
| `/data/qbittorrent/config/` | `backups/homeserver/data/qbittorrent/` |
| `/data/jellyfin/config/` | `backups/homeserver/data/jellyfin/` |
| `/data/unifi/config/` | `backups/homeserver/data/unifi/` |
| `/data/squid/config/` | `backups/homeserver/data/squid/` |
| `/data/portainer/` | `backups/homeserver/data/portainer/` |
| `/data/nginx/` | `backups/homeserver/data/nginx/` |
| Nextcloud DB (mysqldump) | `backups/homeserver/data/nextcloud-db.sql.gz` |

**Not backed up**: `/data/media/` (movies, TV, downloads — too large, re-downloadable)

## disk3 Local Backup

`/data/disk3/backups/` — manual and nightly rsync to local HDD.

| Source | Destination on disk3 |
|---|---|
| Fedora `~/projects/` | `backups/fedora/projects/` |
| Server `~/homeserver/` | `backups/server/projects/` |
| `/etc/` | `backups/server/configs/etc/` |
| `/data/pihole/` | `backups/server/configs/pihole/` |
| `/data/sonarr/` | `backups/server/docker/sonarr/` |
| `/data/radarr/` | `backups/server/docker/radarr/` |
| `/data/prowlarr/` | `backups/server/docker/prowlarr/` |
| `/data/jellyfin/` | `backups/server/docker/jellyfin/` |
| `/data/nextcloud/` | `backups/server/docker/nextcloud/` |
| `/data/nginx/` | `backups/server/docker/nginx/` |
| `/data/portainer/` | `backups/server/docker/portainer/` |
| `/data/unifi/` | `backups/server/docker/unifi/` |
| `/data/qbittorrent/` | `backups/server/docker/qbittorrent/` |

**Run manually:**
```bash
# Fedora projects → disk3
rsync -a --delete ~/projects/ adminbill@10.0.0.1:/data/disk3/backups/fedora/projects/

# Server configs + Docker → disk3
sudo rsync -a --delete /etc/ /data/disk3/backups/server/configs/etc/
sudo rsync -a --delete /data/sonarr/ /data/disk3/backups/server/docker/sonarr/
# etc.
```

## SSH Auth

Server uses a passwordless SSH key to push to Fedora:
- Server key: `/home/adminbill/.ssh/id_ed25519` (also copied to `/root/.ssh/`)
- Fedora authorized_keys: `/home/adminbill/.ssh/authorized_keys`

## Run Manually

```bash
sudo /usr/local/bin/server-backup.sh
sudo tail -f /var/log/server-backup.log
```

## Restore Example

```bash
# Restore nftables config
sudo cp ~/backups/homeserver/etc/nftables.conf /etc/nftables.conf
sudo nft -f /etc/nftables.conf

# Restore a Docker app config
sudo rsync -av ~/backups/homeserver/data/sonarr/ /data/sonarr/config/
docker restart sonarr
```
