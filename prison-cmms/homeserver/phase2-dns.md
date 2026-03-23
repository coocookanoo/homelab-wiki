# Phase 2 — DHCP, DNS & Pi-hole

**Status: ✅ Complete**

## What's Running
- **dnsmasq** on host, bound to enp4s0 (10.0.0.1)
  - DHCP: 10.0.0.100 – 10.0.0.200, 24h leases
  - DNS: forwarded to Pi-hole, upstream 1.1.1.1 / 8.8.8.8
- **Pi-hole** in Docker on :53 and :8081
  - Status: running, blocklists active (362,847 domains blocked)
  - Blocklists: oisd (~280k) + StevenBlack (~81k)

## Pi-hole Web UI
- URL: http://10.0.0.1:8081
- Password: stored in `/data/docker/.env` as `PIHOLE_PASSWORD`

> Note: Port moved from 8080 → 8081 to free 8080 for UniFi AP communication.

## Pi-hole v6 Login Notes
- Pi-hole v6 changed its auth system significantly
- If login silently fails (enter password, nothing happens): **clear browser cookies** for `10.0.0.1:8081`
- To reset password: `docker exec pihole pihole setpassword 'newpassword'`

## Docker Compose Snippet
```yaml
pihole:
  image: pihole/pihole:latest
  ports:
    - "53:53/tcp"
    - "53:53/udp"
    - "8081:80/tcp"               # 8080 reserved for UniFi AP communication
  environment:
    TZ: '${TZ}'
    WEBPASSWORD: '${PIHOLE_PASSWORD}'
  volumes:
    - /data/pihole/etc:/etc/pihole
    - /data/pihole/dnsmasq:/etc/dnsmasq.d
  restart: unless-stopped
  cap_add:
    - NET_ADMIN
```

## Blocklists
Managed via Pi-hole web UI → Adlists. Current lists:
| List | Domains |
|---|---|
| oisd (big) | ~280,000 |
| StevenBlack hosts | ~81,000 |

To refresh: `docker exec pihole pihole -g`

## Manual Pi-hole Commands
```bash
# Update blocklists
docker exec pihole pihole -g

# Check status
docker exec pihole pihole status

# Whitelist a domain
docker exec pihole pihole -w example.com

# View recent queries
docker exec pihole pihole -t

# Reset password
docker exec pihole pihole setpassword 'newpassword'
```

## dnsmasq Config Location
`/etc/dnsmasq.conf` — managed on the host, not in Docker.
