# Homelab Wiki

Central documentation for all projects.

## Prison CMMS

- [[prison-cmms/setup/local-dev]] — Local development setup
- [[prison-cmms/setup/docker]] — Docker deployment
- [[prison-cmms/architecture/entities]] — Data model & entity relationships
- [[prison-cmms/architecture/api-patterns]] — URL patterns, auth flow, conventions
- [[prison-cmms/architecture/asset-hierarchy]] — 14 CMMS asset categories
- [[prison-cmms/features/authentication]] — JWT auth system
- [[prison-cmms/features/work-orders]] — Work orders & AI suggestions
- [[prison-cmms/changelog]] — Change log & decisions

## Homeserver (server1)

- [[homeserver/network-design]] — Network topology & IP plan
- [[homeserver/nginx-proxy-manager]] — Reverse proxy & SSL
- [[homeserver/wikijs]] — This wiki
- [[homeserver/phase0-install]] — OS installation
- [[homeserver/phase1-routing]] — Routing & firewall (nftables)
- [[homeserver/phase2-dns]] — DNS configuration
- [[homeserver/phase2-dns-pihole]] — Pi-hole ad blocking
- [[homeserver/phase3-vpn]] — WireGuard VPN
- [[homeserver/phase4-proxy]] — Squid HTTP proxy
- [[homeserver/phase5-media]] — Media stack (Jellyfin/Radarr/Sonarr)
- [[homeserver/phase7-backup]] — Backup strategy
- [[homeserver/phase8-storage]] — Storage & mergerfs
- [[homeserver/gluetun-vpn]] — Mullvad VPN tunnel
- [[homeserver/switch-netvanta1560]] — ADTRAN switch

## Java Learning

- [[javaplay/README]] — Java notes

## Fedora Desktop

- [[fedora/system-changes]] — **Change register: every program/service we've adjusted and why (start here)**
- [[fedora/PLAN]] — Fedora desktop plan
- [[fedora/ollama-setup]] — Ollama + Open WebUI setup (local LLM)
- [[fedora/ssh-persistent-sessions]] — SSH never drops: sshd keepalives + tmux auto-start on boot
- [[fedora/ram-stability-bios-settings]] — BIOS settings for the 4×dual-rank mixed-kit RAM (stability)
- [[fedora/steam-troubleshooting]] — Steam: log map, `compatdata` anatomy, launch-failure triage
- [[fedora/plasma-desktop-icon-freeze]] — desktop icons stop responding to clicks (recurring, root cause unknown)
- [[fedora/rclone-gdrive-crash-lockout]] — rclone Drive mount: crash left a stale FUSE mount, locked out 1,158 restarts
- [[fedora/dnf-update-snapd-crash-loop]] — snapd 203/EXEC crash-loop after selinux-policy bumps
- [[fedora/fedora-45-upgrade]] — F44 → F45 (2026-11-23): kimageformats pin blocks the boot-time run, Python 3.15 breaks ~/.local tools, kwin 6.8.3 crash (KDE bug 528824)
- [[fedora/firefox-amd-igpu-decode]] — hw video decode on the AMD iGPU (tried, reverted)
- [[fedora/lmstudio-net-fix]] — LM Studio network fix
- [[fedora/lmstudio-to-ollama-migration]] — LM Studio → Ollama migration
