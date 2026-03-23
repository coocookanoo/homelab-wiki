> **Wiki Notice:** Project documentation should be copied to `/home/adminbill/projects/docs/<project-name>/` and pushed to [homelab-wiki](https://github.com/coocookanoo/homelab-wiki) so it appears at https://wiki.billsserver1.duckdns.org

---

# Phase 8 — HDD Media Storage (mergerfs + SnapRAID)

## Overview

Add 3x 4TB Seagate HDDs as a pooled media storage volume using mergerfs.
SnapRAID provides parity-based protection against a single drive failure.

- **Pool mount**: `/mnt/media-pool`
- **Drives**: /dev/sdX, /dev/sdY, /dev/sdZ (identify after plugging in)
- **Move**: `/data/media` → `/mnt/media-pool`

## Step 1 — Identify the Drives

After plugging in drives and booting:

```bash
lsblk
sudo fdisk -l | grep -E 'Disk /dev/sd|GB'
sudo smartctl -a /dev/sdX   # check health on each drive
```

## Step 2 — Check Drive Health

```bash
sudo apt install -y smartmontools
sudo smartctl -H /dev/sdX   # PASSED = good
sudo smartctl -t short /dev/sdX   # run short self-test
```

Do this for each drive before formatting.

## Step 3 — Format the Drives

```bash
sudo mkfs.ext4 -L media1 /dev/sdX
sudo mkfs.ext4 -L media2 /dev/sdY
sudo mkfs.ext4 -L media3 /dev/sdZ
```

## Step 4 — Mount Points

```bash
sudo mkdir -p /mnt/disk1 /mnt/disk2 /mnt/disk3 /mnt/media-pool
```

Add to `/etc/fstab`:
```
LABEL=media1  /mnt/disk1  ext4  defaults,nofail  0 2
LABEL=media2  /mnt/disk2  ext4  defaults,nofail  0 2
LABEL=media3  /mnt/disk3  ext4  defaults,nofail  0 2
```

```bash
sudo mount -a
```

## Step 5 — Install and Configure mergerfs

```bash
sudo apt install -y mergerfs
```

Add to `/etc/fstab`:
```
/mnt/disk1:/mnt/disk2:/mnt/disk3  /mnt/media-pool  fuse.mergerfs  defaults,allow_other,use_ino,cache.files=off,dropcacheonclose=true,category.create=mfs,nofail  0 0
```

```bash
sudo mount -a
df -h /mnt/media-pool   # should show ~12TB
```

## Step 6 — Move Media Data

```bash
# Stop Docker stack first
cd /data/docker && docker compose down

# Copy data (rsync preserves permissions)
sudo rsync -av /data/media/ /mnt/media-pool/

# Verify
ls /mnt/media-pool/

# Rename old and symlink or update fstab bind mount
sudo mv /data/media /data/media.bak
sudo mkdir /data/media
```

Add bind mount to `/etc/fstab`:
```
/mnt/media-pool  /data/media  none  bind  0 0
```

```bash
sudo mount -a

# Bring Docker back up
docker compose up -d

# Once confirmed working, remove backup
sudo rm -rf /data/media.bak
```

## Step 7 — Install SnapRAID (Optional but Recommended)

```bash
sudo apt install -y snapraid
```

Edit `/etc/snapraid.conf`:
```
parity /mnt/disk3/snapraid.parity   # use one disk for parity
content /mnt/disk1/snapraid.content
content /mnt/disk2/snapraid.content
data d1 /mnt/disk1
data d2 /mnt/disk2
```

Run initial sync:
```bash
sudo snapraid sync
```

Add cron for nightly sync:
```
0 3 * * * /usr/bin/snapraid sync >> /var/log/snapraid.log 2>&1
```

## Notes

- `nofail` in fstab is critical — prevents boot failures if a drive is missing
- mergerfs does NOT stripe data — each file lives on one drive
- SnapRAID is not real-time — only protects against drive failure between syncs
- Docker configs stay on SSD (/data/*/config) — only media moves to HDDs
