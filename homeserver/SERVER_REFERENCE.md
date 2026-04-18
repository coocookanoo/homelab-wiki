# Home Server Reference

Last updated: 2026-04-17

---

## Hardware

- **CPU:** Intel i5-6600
- **RAM:** 8GB
- **OS:** Ubuntu 24.04

---

## Network

| Interface | Address |
|-----------|---------|
| WAN (enp0s31f6) | 192.168.1.34 (DHCP from modem) |
| LAN (enp4s0) | 10.0.0.1/24 |
| Tailscale | 100.64.61.90 |

**Fedora PC** is on the modem's network (192.168.1.33) — accesses services via 192.168.1.34:PORT, NOT the LAN.

**LAN (10.0.0.x)** is for downstream devices through the switch.

## Host Split

| Host | Role | Addressing |
|------|------|------------|
| `server1` | Router, firewall, DNS, VPN, proxy, media, apps | `10.0.0.1`, `192.168.1.34`, `100.64.61.90` |
| `server2` | Forgejo, NVR target host | `10.0.0.145` (static, set 2026-04-17) |
| `server3` | Proxmox host, monitoring stack | `10.0.0.151` (Proxmox), `10.0.0.3` (node-exporter) — Intel i5-6600, 8GB RAM |

---

## Services & Ports

Access from **LAN:** `http://10.0.0.1:PORT`
Access from **Fedora/WAN:** `http://192.168.1.34:PORT`
Access via **Tailscale:** `http://100.64.61.90:PORT`

| Service | Port | Notes |
|---------|------|-------|
| **Homepage Dashboard** | 3000 | Main dashboard, live stats |
| **Portainer** | 9000 | Docker container management |
| **Cockpit** | 9090 | Host system management UI |
| **Nextcloud** | 8443 (HTTPS) | Personal cloud |
| **Jellyfin** | 8096 | Media server |
| **Pi-hole** | 8081 | DNS ad blocker |
| **Nginx Proxy Manager** | 81 (admin), 80/443 | Reverse proxy |
| **qBittorrent** | 8090 | Torrent client (via Gluetun VPN) |
| **Radarr** | 7878 | Movie management |
| **Sonarr** | 8989 | TV show management |
| **Prowlarr** | 9696 | Indexer manager |
| **Unifi Controller** | 8444 (HTTPS) | Unifi AP management |
| **Squid Proxy** | 3128 | HTTP proxy |
| **Frigate** | 8554-8555, 8971 | Running on `server2` — all 4 cameras live |
| **Prometheus** | 9090 | CT 101 on server3 (10.0.0.151) — scrapes server1, server2, server3, fedora, switch |
| **Grafana** | 3000 | CT 101 on server3 (10.0.0.151) — Node Exporter Full (1860) + SNMP Stats (11169) |
| **Node Exporter** | 9100 | Running on server1, server2, server3, Fedora workstation |
| **snmp_exporter** | 9116 | CT 101 on server3 — scrapes NetVanta switch via SNMPv2c community `homelab` |
| **Seerr** | 5055 | Media requests |
| **Wiki.js** | 3010 | Internal wiki |
| **Flaresolverr** | 8191 | Arr support service |

---

## Media Pipeline

```
Prowlarr (indexers) → Radarr/Sonarr → qBittorrent → auto-import → Jellyfin
```

- Downloads land in: `/data/media/downloads`
- Movies: `/data/media/movies`
- TV: `/data/media/tv`
- qBittorrent routes through **Gluetun** VPN container

---

## Storage

| Mount | Device | Size |
|-------|--------|------|
| /data/disk2 | /dev/sdd1 (UUID: e631742e-...) | 4TB Seagate |
| /data/disk3 | /dev/sdf1 (UUID: 31f71a58-...) | 4TB Seagate |
| /data/media | mergerfs pool (disk2 + disk3) | ~7.2TB |

**Docker volumes** live under `/data/` (e.g. `/data/radarr/config/`)

**Camera / NVR data on server1:** `/data/frigate/`

**Retired drives:** 1x2TB WD (1265 reallocated sectors), 1x2TB Seagate sdb (I/O errors) — do not use.

---

## Key File Locations

| File | Path |
|------|------|
| Docker Compose (source) | `~/homeserver/configs/docker/docker-compose.yml` |
| Docker Compose (active) | `/data/docker/docker-compose.yml` |
| Docker secrets | `/data/docker/.env` (chmod 600) |
| nftables firewall (source) | `~/homeserver/configs/nftables/nftables.conf` |
| nftables firewall (active) | `/etc/nftables.conf` |
| Homepage config | `/data/homepage/` |
| WireGuard config dir | `~/homeserver/configs/wireguard/` |
| Frigate config on server1 | `/data/frigate/config/` |
| Frigate storage on server1 | `/data/frigate/storage/` |
| Frigate config on server2 | `/home/adminbill/frigate/config/` |
| Frigate storage on server2 | `/home/adminbill/frigate/storage/` |

> **Always edit the source file in `~/homeserver/configs/`, then copy to the active location.**

---

## server2 SSH Access

| From | Command |
|------|---------|
| LAN / server1 | `ssh adminbill@10.0.0.145` |
| Fedora (WAN) | `ssh -p 2222 adminbill@192.168.1.34` |

Tunnel is persistent via `autossh-tunnel.service` on server2.
server2 → server1 auth uses ed25519 key (`~/.ssh/id_ed25519`).

---

## Network Hardware

| Device | IP | Notes |
|--------|----|-------|
| ADTRAN NetVanta 1560 switch | 10.0.0.127 | Managed PoE, 8-port |
| Unifi AP | 10.0.0.162 | MAC: 60:22:32:7c:3a:f8 |

**Switch serial access (if web UI locked out):**
1. Enable serial port in BIOS first
2. `sudo screen /dev/ttyS0 9600` — 9600 baud 8N1

---

## Common Docker Commands

```bash
# View all containers
docker ps

# Restart a container
docker restart <container-name>

# View logs
docker logs <container-name>

# Restart entire stack
cd /data/docker && docker compose up -d

# Pull latest images & restart
cd /data/docker && docker compose pull && docker compose up -d
```

---

## Security

- **Fail2ban:** Running, SSH jail active since 2026-03-18
- **Firewall:** nftables, enabled on boot
- **Tailscale:** 2 peers connected

---

## Pending / To-Do

- [ ] Set Radarr password (auth currently disabled for local addresses)
- [ ] Configure WireGuard (package installed, not set up)
- [ ] Enable automatic security updates (unattended-upgrades)
- [ ] Set up backups
- [ ] Unifi AP adoption (blocked by NetVanta VLAN issue)
- [ ] Fix NetVanta switch web UI access (may need serial console)
- [ ] Add more HDDs from work NVR — run SMART tests before adding to pool
- [x] Fresh Ubuntu 24.04 install on server2 (sda only, sdb removed) — 2026-04-17
- [x] Static IP 10.0.0.145 configured via netplan (cloud-init disabled) — 2026-04-17
- [x] Docker 29.4.0 + Codex CLI 0.121.0 installed on server2 — 2026-04-17
- [x] Frigate running, all 4 cameras live, VAAPI disabled — 2026-04-17
- [x] SSH key (ed25519) from server2 → server1, keyless auth working — 2026-04-17
- [x] Reverse SSH tunnel (autossh-tunnel.service): server1:2222 → server2:22 — 2026-04-17
- [ ] Deploy Forgejo on server2
- [ ] Create LVM frigate-storage volume (sda3 has ~5.4T free in ubuntu-vg)

---

## Nextcloud DB Info

- **DB:** DBbill
- **DB User:** adminbill
- **Backend:** MariaDB (container: nextcloud-db)
