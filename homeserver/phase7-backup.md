> **Wiki Notice:** Project documentation should be copied to `/home/adminbill/projects/docs/<project-name>/` and pushed to [homelab-wiki](https://github.com/coocookanoo/homelab-wiki) so it appears at https://wiki.billsserver1.duckdns.org

---

# Phase 7 — Backups

## Backup Destinations

Three backup targets:

1. **Local — disk3** (`/data/disk3/backups/`) — 3.6TB HDD, always available
2. **Local — disk4** (`/data/disk4/backups/`) — 3.6TB HDD (`/dev/sde3`, UUID `ac927c15-1d1b-490c-8679-40a3b08f5a35`), mounted via fstab
3. **Remote — Fedora PC** (`adminbill@192.168.1.33:/home/adminbill/backups/homeserver/`) — copy on LAN, only when Fedora is reachable

## Nightly Backup Script

- **Script**: `/usr/local/bin/server-backup.sh`
- **Log**: `/var/log/server-backup.log`
- **Schedule**: `0 18 * * *` (root crontab on server — daily at 6pm)

**What it does:**
1. Always backs up to disk3 and disk4
2. If Fedora is reachable — pulls Fedora projects to disk4, pushes server backup to Fedora

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

## server2 Recovery Backup on USB (disk4)

As of `2026-04-17`, the nightly `server1` backup also stores a recoverable
`server2` config archive on the USB backup disk:

- Destination: `/data/disk4/backups/server2/server2-configs.tar.gz`
- Triggered by: `/usr/local/bin/server-backup.sh` on `server1`
- Source host: `server2` (`10.0.0.145`)

The archive is streamed over SSH from `server2` and includes:

- `/etc/`
- `/home/adminbill/docs/`
- `/home/adminbill/frigate/docker-compose.yml`
- `/home/adminbill/frigate/config/`
- `/home/adminbill/restore-server2.sh`
- `/home/adminbill/.ssh/authorized_keys`

Excluded from the archive:

- `home/adminbill/frigate/config/model_cache`
- `home/adminbill/frigate/config/frigate.db-shm`
- `home/adminbill/frigate/config/frigate.db-wal`
- `home/adminbill/frigate/config/.timeline`
- Frigate recordings and media storage under `/home/adminbill/frigate/storage/`

Implementation details:

- `server2` provides `/usr/local/bin/server2-backup-stream.sh`
- `adminbill` on `server2` is allowed to run that one script via sudo without a password
- `server1` connects over SSH and writes the resulting `tar.gz` directly to disk4

Manual test from `server1`:

```bash
ssh adminbill@10.0.0.145 "sudo /usr/local/bin/server2-backup-stream.sh" > /tmp/server2-configs.tar.gz
tar -tzf /tmp/server2-configs.tar.gz | head
```

## server3 Recovery Backup on USB (disk4)

As of `2026-04-18`, the nightly `server1` backup also stores a recoverable
`server3` config archive on the USB backup disk:

- Destination: `/data/disk4/backups/server3/server3-configs.tar.gz`
- Triggered by: `/usr/local/bin/server-backup.sh` on `server1`
- Source host: `server3` (`10.0.0.3`)

The archive is streamed over SSH (as root) from `server3` and includes:

- `/etc/pve/lxc/101.conf` — Proxmox CT 101 config
- CT 101 `/etc/prometheus/prometheus.yml`
- CT 101 `/etc/grafana/grafana.ini`
- CT 101 `/var/lib/grafana/grafana.db`

Implementation details:

- `server3` provides `/usr/local/bin/server3-backup-stream.sh`
- `server1` root SSH key (`/root/.ssh/id_ed25519`) is in `server3` root `authorized_keys`
- `server1` connects as `root@10.0.0.3` and writes the resulting `tar.gz` directly to disk4

Manual test from `server1`:

```bash
ssh root@10.0.0.3 "/usr/local/bin/server3-backup-stream.sh" > /tmp/server3-configs.tar.gz
tar -tzf /tmp/server3-configs.tar.gz | head
```

## disk3 and disk4 Local Backups

Both disks receive identical server backups nightly. disk4 also stores the Fedora projects pull.


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

---

## Incident: disk4 (sde3) unmounted — backups filled root FS (2026-04-16)

**What happened:**
`/dev/sde3` (disk4, 3.6TB) was not mounted. The backup script writes to `/data/disk4/backups/`
which, without the mount, writes to the root filesystem. Over time 138GB of backups accumulated
on root, eventually filling it to 100% and breaking dnsmasq (couldn't write lease file),
which prevented new devices from getting DHCP addresses.

**How it was caught:**
New Hikvision cameras wouldn't appear on the network. dnsmasq logs showed:
```
failed to write /var/lib/misc/dnsmasq.leases: No space left on device
```

**Fix applied:**
```bash
# Mount sde3 temporarily
sudo mount /dev/sde3 /mnt

# Sync newer backup data from root → disk
sudo rsync -a --update /data/disk4/backups/ /mnt/backups/

# Delete from root to free space
sudo rm -rf /data/disk4/backups

# Unmount temp, mount permanently
sudo umount /mnt
sudo mount /dev/sde3 /data/disk4

# Add to fstab (UUID from blkid)
echo 'UUID=ac927c15-1d1b-490c-8679-40a3b08f5a35 /data/disk4 ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab
```

**Result:** Root dropped from 100% → 28% (150GB free). Backup script verified working to
disk3, disk4, and Fedora.

**Prevention:** If root fills up again check:
1. `df -h` — identify the filesystem
2. `sudo du -sh /data/* 2>/dev/null | sort -rh` — find what's on root vs mounted disks
3. `mount | grep /data` — verify disk2/disk3/disk4 are all mounted
4. `sudo journalctl -u dnsmasq | grep failed` — dnsmasq lease errors are a symptom of full root
