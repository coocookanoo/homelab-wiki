# Phase 4 — Proxy (Squid)

**Status: ✅ Complete** — Squid HTTP proxy running on :3128

## What's Running
- **Squid** Docker container, port 3128
- Allows LAN (10.0.0.0/24) and Docker networks (172.16.0.0/12) to use the proxy
- Cache stored at /data/squid/cache

## Docker Compose Snippet
```yaml
squid:
  image: ubuntu/squid:latest
  container_name: squid
  ports:
    - "3128:3128"
  volumes:
    - /data/squid/config:/etc/squid
    - /data/squid/cache:/var/spool/squid
  restart: unless-stopped
```

## Config Location
`/data/squid/config/squid.conf`

Key settings:
- Listens on port 3128
- Allows localnet ACL: 10.0.0.0/24 + 172.16.0.0/12
- Cache: 1000MB UFS at /var/spool/squid
- Blocks unsafe ports, allows HTTPS CONNECT

## Configure Devices to Use Proxy
Set HTTP proxy on devices to `10.0.0.1:3128`.

On Linux:
```bash
export http_proxy=http://10.0.0.1:3128
export https_proxy=http://10.0.0.1:3128
```

## Squid Management
```bash
# View access log
docker exec squid tail -f /var/log/squid/access.log

# Reload config without restart
docker exec squid squid -k reconfigure

# Check status
docker exec squid squid -k check
```

## Notes
- The cache directory (`/data/squid/cache`) must be owned by the `proxy` user inside
  the container. If Squid fails to start after a fresh deploy, run:
  ```bash
  docker run --rm --entrypoint="" -v /data/squid/cache:/var/spool/squid ubuntu/squid:latest \
    bash -c "chown -R proxy:proxy /var/spool/squid && squid -z --foreground"
  docker restart squid
  ```
