> **Wiki Notice:** Project documentation should be copied to `/home/adminbill/projects/docs/<project-name>/` and pushed to [homelab-wiki](https://github.com/coocookanoo/homelab-wiki) so it appears at https://wiki.billsserver1.duckdns.org

---

# Phase 2 — DHCP + DNS + Pi-hole

## Overview

- **DHCP**: host dnsmasq on `enp4s0`, range `10.0.0.100–200`
- **DNS**: Pi-hole in Docker, port 53, upstream 8.8.8.8 / 8.8.4.4
- **Pi-hole UI**: http://10.0.0.1:8081

## Host dnsmasq (DHCP only)

`/etc/dnsmasq.conf`:
```
interface=enp4s0
bind-interfaces
dhcp-range=10.0.0.100,10.0.0.200,24h
dhcp-option=3,10.0.0.1    # gateway
dhcp-option=6,10.0.0.1    # DNS → Pi-hole
port=0                     # disable DNS (Pi-hole handles it)
```

## Pi-hole Docker

See `../configs/docker/docker-compose.yml` for the full service definition.

Pi-hole maps port 53 (TCP+UDP) to the host so LAN clients can use `10.0.0.1` as DNS.

### Critical: listeningMode

Pi-hole defaults to `listeningMode = "LOCAL"`, which causes it to ignore queries from
non-local subnets. Since LAN queries arrive via DNAT (source IP 10.0.0.x, not 172.18.x.x),
Pi-hole will silently drop them with:

```
WARNING: dnsmasq: ignoring query from non-local network 10.0.0.x
```

Fix — edit `/data/pihole/etc/pihole.toml` and set:
```toml
listeningMode = "ALL"
```
Then restart: `docker restart pihole`

### Verify DNS works from LAN
```bash
dig @10.0.0.1 google.com +short
```

## Unifi WiFi AP

- Controller: `https://10.0.0.1:8444` (Docker, `linuxserver/unifi-network-application`)
- AP adopted at `10.0.0.162`
- AP inform URL: `http://10.0.0.1:8080/inform`
