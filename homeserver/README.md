# Home Server Build — Split Stack Overview

## Hardware
- **CPU:** Intel i5-6600 @ 3.3GHz (quad core, Intel HD 530 GPU w/ QuickSync)
- **RAM:** 8GB
- **Storage:** 218GB (OS + services, /data for Docker volumes) — plan to add HDD for media
- **NICs:** enp0s31f6 (WAN), enp4s0 (LAN)

## Goal
Operate a split home stack:
- `server1` is the router, firewall, DNS, VPN, proxy, media, and app host
- `server2` handles workloads better suited to its hardware, including Forgejo and the NVR role

## Server Roles
- **server1** (`10.0.0.1`, `192.168.1.34`, Tailscale `100.64.61.90`): router/firewall, dnsmasq, Pi-hole, Tailscale, WireGuard staging, Squid, Jellyfin, Nextcloud, Nginx Proxy Manager, Portainer, arr stack, UniFi controller, Homepage
- **server2** (`10.0.0.145`): Forgejo, Frigate NVR — **rebuild in progress 2026-04-17** (Frigate live, all 4 cameras confirmed; Forgejo not yet deployed)
- **server3** (`10.0.0.151`): Proxmox host — Prometheus + Grafana monitoring stack, scraping all 3 servers via node-exporter. Intel i5-6600, 8GB RAM.

## Software Stack
- **OS:** Ubuntu 24.04.4 LTS (Noble Numbat)
- **Routing/Firewall:** nftables
- **DHCP/DNS:** dnsmasq (host) + Pi-hole (Docker)
- **Ad Blocking:** Pi-hole
- **VPN (self-hosted):** WireGuard — *installed, not yet configured*
- **VPN (mesh overlay):** Tailscale — *active, 2 peers connected*
- **Proxy:** Squid HTTP proxy (Docker)
- **Media:** Jellyfin (Docker, hardware transcode via QuickSync)
- **Cloud Storage:** Nextcloud (Docker)
- **Reverse Proxy:** Nginx Proxy Manager (Docker)
- **Container Management:** Portainer (Docker)
- **Web Console:** Cockpit
- **Git Hosting:** Forgejo on `server2`
- **NVR / Cameras:** Frigate on `server2` — 4 Hikvision cameras live (10.0.0.201–204)

## Network Layout
```
[ISP Modem] ──── [WAN: enp0s31f6 / 192.168.1.34]
                          │
                      server1
                          │
              [LAN: enp4s0 / 10.0.0.1]
                          │
                  [Switch/WiFi AP]
                          │
              [Devices: 10.0.0.100–200]
```

## Build Phases
| Phase | Task | Status |
|---|---|---|
| 0 | Install OS | ✅ Done — Ubuntu 24.04 |
| 1 | Base routing + firewall | ✅ Done — nftables active |
| 2 | DHCP + DNS + Pi-hole | ✅ Done |
| 3 | WireGuard VPN | 🔧 Pending — package installed, not configured |
| 4 | Proxy (Squid) | ✅ Done — running on :3128 |
| 5 | Media & Services | ✅ Done — Jellyfin up, Nextcloud up (MariaDB) |
| 6 | Arr stack (Prowlarr + Sonarr + Radarr + qBittorrent) | ✅ Done — full pipeline working |
| 7 | Hardening + backups | 🔧 In progress — fail2ban done, .env done |
| 9 | Forgejo on server2 | ✅ Done — migrated 2026-04-15 |
| 10 | Docs sync to server2 | ✅ Done — manual sync workflow documented |
| 11 | NVR (Frigate on server2) | 🔧 In progress — Frigate live, all 4 cameras up; Forgejo + storage LVM pending |
| 12 | Monitoring (Prometheus + Grafana on server3) | ✅ Done — server1, server2, server3, Fedora all scraped; Node Exporter Full dashboard live |

## Known Issues / TODO
- [x] Apply nftables fix — rules persisted, service enabled on boot
- [x] Complete Nextcloud web setup (MariaDB backend, accessible via Tailscale at http://100.64.61.90:8443)
- [x] Change Pi-hole web password
- [x] Deploy arr stack — full pipeline working (Radarr/Sonarr → qBittorrent → Jellyfin)
- [x] Configure qBittorrent: download path /data/media/downloads
- [x] Add indexers in Prowlarr (The Pirate Bay added)
- [x] Sync Prowlarr to Sonarr + Radarr
- [x] Add qBittorrent as download client in Sonarr + Radarr
- [x] Set root folders: Sonarr → /data/media/tv, Radarr → /data/media/movies
- [x] Fail2ban installed and running (SSH jail active)
- [x] Secrets moved to .env file (passwords out of docker-compose)
- [ ] Finish Unifi AP adoption — SSH to 10.0.0.162, run: `set-inform http://10.0.0.1:8080/inform`
- [ ] Set Radarr password (auth currently disabled for local addresses)
- [ ] Deploy WireGuard (Phase 3)
- [ ] Automatic security updates (unattended-upgrades)
- [ ] Backups
- [x] Rebuild server2 — fresh Ubuntu 24.04 install on sda only (2026-04-17)
- [x] Static IP set to 10.0.0.145, cloud-init network management disabled (2026-04-17)
- [x] Docker 29.4.0 installed on server2 (2026-04-17)
- [x] Codex CLI 0.121.0 installed on server2 (2026-04-17)
- [x] Codex sandbox fixed on server2 — `bubblewrap` installed and AppArmor profile added for `/usr/bin/bwrap` (2026-04-17)
- [x] Frigate running on server2 — VAAPI disabled, all 4 cameras live (2026-04-17)
- [x] server1 USB backup now includes recoverable server2 config archive on disk4 (2026-04-17)
- [ ] Deploy Forgejo on server2
- [ ] Create LVM frigate-storage volume on sda (sda3 has ~5.4T free in ubuntu-vg)
- [ ] Set up automated backups to USB drive on server1
- [ ] Add go2rtc for better Frigate live view
- [x] Add substream input per camera (substream for detect, mainstream for record)

## Docs
- [Hardware & Network Design](docs/network-design.md)
- [Phase 0 — Install Ubuntu](docs/phase0-install.md)
- [Phase 1 — Routing & Firewall](docs/phase1-routing.md)
- [Phase 2 — DNS & Ad Blocking](docs/phase2-dns.md)
- [Phase 3 — WireGuard VPN](docs/phase3-vpn.md)
- [Phase 4 — Proxy](docs/phase4-proxy.md)
- [Phase 5 — Media & Services](docs/phase5-media.md)
- [Phase 9 — Forgejo](phase9-forgejo.md)
- [Phase 10 — Docs Sync](phase10-doc-sync.md)
- [Phase 11 — NVR Split](phase11-nvr-split.md)
- [Phase 12 — Monitoring](phase12-monitoring.md)
