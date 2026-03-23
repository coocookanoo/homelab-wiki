> **Wiki Notice:** Project documentation should be copied to `/home/adminbill/projects/docs/<project-name>/` and pushed to [homelab-wiki](https://github.com/coocookanoo/homelab-wiki) so it appears at https://wiki.billsserver1.duckdns.org

---

# Nginx Proxy Manager — Reverse Proxy & HTTPS

Nginx Proxy Manager (NPM) handles reverse proxying and Let's Encrypt SSL certs for all
services exposed to the internet or accessed by domain name.

- **Admin UI**: http://10.0.0.1:81
- **Login**: `admin@bill.com`
- **HTTP**: port 80 (Let's Encrypt challenge + redirect to HTTPS)
- **HTTPS**: port 443 (all proxied services)
- **Docker container**: `nginx-proxy-manager`

## ISP Modem Port Forwards Required

These must be set in the ISP modem's port forwarding settings (modem admin page, usually at 192.168.1.1).
Forward to server WAN IP: **192.168.1.34**

| External Port | Protocol | Internal Port | Purpose |
|---|---|---|---|
| 80 | TCP | 80 | Let's Encrypt HTTP challenge + HTTP→HTTPS redirect |
| 443 | TCP | 443 | HTTPS for all proxied services |
| 51820 | UDP | 51820 | WireGuard VPN |

## DuckDNS Domain

Base domain: `billsserver1.duckdns.org` — must point to the server's public IP.

Check/update at https://www.duckdns.org

To verify it's correct:
```bash
curl ifconfig.me          # your public IP
nslookup billsserver1.duckdns.org   # should match
```

## Active Proxy Hosts

| Domain | Internal Target | Port | SSL | Notes |
|---|---|---|---|---|
| `billsserver1.duckdns.org` | `nextcloud` | 80 | ✅ Let's Encrypt | HTTPS forced, cert expires Jun 20 2026 |
| `jellyfin.billsserver1.duckdns.org` | `jellyfin` | 8096 | planned | — |
| `portainer.billsserver1.duckdns.org` | `portainer` | 9443 | planned | — |

> Note: DuckDNS does not support subdomains of subdomains (e.g. `nextcloud.billsserver1.duckdns.org`
> won't resolve). Use the base domain for the primary service, or register separate DuckDNS domains.

## Getting a Certificate (DNS Challenge — Recommended)

HTTP challenge (port 80) may be blocked by ISP. DNS challenge works without port 80.

1. **SSL Certificates** → **Add SSL Certificate** → **Let's Encrypt**
2. Domain: `billsserver1.duckdns.org`
3. Switch to **DNS** challenge
4. DNS Provider: **DuckDNS**
5. Credentials: `dns_duckdns_token=<your-duckdns-token>`
6. Agree to ToS → Save

DuckDNS token found at https://www.duckdns.org after login.

## Adding a New Proxy Host (Step-by-Step)

1. Open NPM admin at http://10.0.0.1:81
2. **Proxy Hosts** → **Add Proxy Host**
3. **Details tab**:
   - Domain Names: type domain then press **Enter** to confirm as a tag
   - Scheme: `http`
   - Forward Hostname: Docker container name (e.g. `nextcloud`, `jellyfin`)
   - Forward Port: container's internal port
4. **SSL tab**:
   - SSL Certificate: select existing cert
   - Force SSL: ✅
   - HTTP/2 Support: ✅
5. Save

## Nextcloud HTTPS Setup (Done)

Nextcloud is accessible at `https://billsserver1.duckdns.org`. occ commands already run:

```bash
sudo docker exec nextcloud php occ config:system:set trusted_domains 1 --value=10.0.0.1
sudo docker exec nextcloud php occ config:system:set trusted_domains 2 --value=100.64.61.90
sudo docker exec nextcloud php occ config:system:set trusted_domains 3 --value=billsserver1.duckdns.org
sudo docker exec nextcloud php occ config:system:set overwrite.cli.url --value=https://billsserver1.duckdns.org
sudo docker exec nextcloud php occ config:system:set overwriteprotocol --value=https
```

Mobile app: use `https://billsserver1.duckdns.org` — trusted cert, no TLS errors.
Photo auto-upload working. iOS setup notes:
- Account must use `https://billsserver1.duckdns.org` (not the LAN IP)
- Settings → Auto Upload → disable **"Only upload new photos"** to upload existing library
- Background App Refresh must be ON (Settings → General → Background App Refresh → Nextcloud)
- Low Power Mode must be OFF for background uploads to run

## Troubleshooting

**Domain auto-deletes in proxy host form:**
- Type the domain then press **Enter** — it must become a blue tag before saving

**Let's Encrypt DNS challenge fails:**
- Verify DuckDNS token is correct (no extra spaces)
- Check NPM logs: `docker logs nginx-proxy-manager`

**502 Bad Gateway:**
- Container name in "Forward Hostname" is wrong, or container is down
- Check: `docker ps | grep <name>`

**Nextcloud redirect loop:**
- Missing `overwriteprotocol` setting — run the occ commands above
