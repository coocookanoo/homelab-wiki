> **Wiki Notice:** Project documentation should be copied to `/home/adminbill/projects/docs/<project-name>/` and pushed to [homelab-wiki](https://github.com/coocookanoo/homelab-wiki) so it appears at https://wiki.billsserver1.duckdns.org

---

# Gluetun — Mullvad VPN Tunnel for qBittorrent

## Overview

Gluetun is a VPN client container that routes qBittorrent's traffic through Mullvad
WireGuard. Your ISP sees only encrypted WireGuard traffic — torrent peer connections
and tracker requests are hidden.

**qBittorrent uses `network_mode: service:gluetun`** — it has no network of its own.
All traffic exits via Gluetun's tunnel. If Gluetun goes down, qBittorrent loses internet.

## Mullvad Account

- **Account number**: stored in `/home/adminbill/homeserver/configs/docker/.env` as `MULLVAD_ACCOUNT`
- **WireGuard device name**: Strong Insect
- **Configs downloaded to**: `/home/adminbill/Downloads/mullvad_wireguard_linux_all_all/`
- **Renewal date**: April 21, 2026

## Credentials in .env

```
MULLVAD_ACCOUNT=<account number>
MULLVAD_PRIVATE_KEY=<wireguard private key>
MULLVAD_ADDRESS=10.67.53.171/32
```

## docker-compose Config

```yaml
gluetun:
  image: qmcgaw/gluetun:latest
  container_name: gluetun
  cap_add:
    - NET_ADMIN
  devices:
    - /dev/net/tun:/dev/net/tun
  ports:
    - "8090:8090"       # qBittorrent web UI
    - "6881:6881"       # torrent TCP
    - "6881:6881/udp"   # torrent UDP
  environment:
    VPN_SERVICE_PROVIDER: mullvad
    VPN_TYPE: wireguard
    WIREGUARD_PRIVATE_KEY: "${MULLVAD_PRIVATE_KEY}"
    WIREGUARD_ADDRESSES: "${MULLVAD_ADDRESS}"
    SERVER_COUNTRIES: USA
  restart: unless-stopped

qbittorrent:
  ...
  network_mode: "service:gluetun"
  depends_on:
    - gluetun
  # No ports section — ports are on gluetun
```

## Verify VPN is Working

```bash
docker exec gluetun wget -qO- https://am.i.mullvad.net/json
```

Should return `"mullvad_exit_ip": true` and a Mullvad IP, not your real WAN IP.

## Change Exit Country

Edit `SERVER_COUNTRIES` in docker-compose and restart:
```bash
cd ~/homeserver/configs/docker
docker compose up -d gluetun
```

Valid options include: USA, Netherlands, Sweden, Germany, Switzerland, UK, Canada

## Restart / Troubleshoot

```bash
# Restart both
docker compose restart gluetun qbittorrent

# Check VPN tunnel logs
docker logs gluetun

# Check qBittorrent is running inside tunnel
docker exec qbittorrent wget -qO- ifconfig.me
```

## Notes

- Mullvad WireGuard does **not support port forwarding** — incoming torrent connections
  on 6881 won't work from external peers. Outbound connections still work fine.
- Port 6881 is open in nftables for LAN access to the qBittorrent web UI and peers
  on the local network.
- If Gluetun fails to connect (wrong keys, expired account), qBittorrent goes offline —
  this is the intended kill-switch behavior.
