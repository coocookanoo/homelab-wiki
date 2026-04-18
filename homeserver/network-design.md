> **Wiki Notice:** Project documentation should be copied to `/home/adminbill/projects/docs/<project-name>/` and pushed to [homelab-wiki](https://github.com/coocookanoo/homelab-wiki) so it appears at https://wiki.billsserver1.duckdns.org

---

# Network Design

## Physical Layout
```
[ISP Modem / 192.168.1.1]
        │  192.168.1.x subnet
        ├── [Fedora PC / 192.168.1.33]   ← adminbill's workstation
        │
        └── [WAN: enp0s31f6 / 192.168.1.34]
                  [THIS SERVER — Ubuntu 24.04]
            [LAN: enp5s0 / 10.0.0.1]
                        │
          [ADTRAN NetVanta 1560-08-65W switch]
          10.0.0.2 — port 5 = server uplink
                        │
         ┌──────────────┴──────────────┐
   [UniFi AP]              [server2 + camera + other devices]
   port 1 (PoE)                  ports 2–10
   10.0.0.162
```

> **Key point:** The Fedora PC and server are on the **same ISP subnet (192.168.1.x)** —
> NOT on the server's LAN (10.0.0.x). When the Fedora PC accesses the server at
> `192.168.1.34`, traffic arrives on the WAN interface `enp0s31f6`. nftables drops
> all uninitiated WAN traffic by default, so **any service you want to reach from
> the Fedora PC must be explicitly allowed on enp0s31f6 or via the home LAN rule.**
>
> The current nftables config allows the entire 192.168.1.0/24 subnet:
> `iif $WAN_IF ip saddr 192.168.1.0/24 accept`

## IP Plan
| Device | IP | Notes |
|---|---|---|
| ISP Modem | 192.168.1.1 | Gateway for both server and Fedora PC |
| Fedora PC | 192.168.1.33 | adminbill's workstation — on ISP subnet, not server LAN |
| Server (WAN) | 192.168.1.34 | DHCP from modem — how Fedora PC reaches server |
| Server (LAN) | 10.0.0.1 | Static, gateway for switch/AP/LAN devices |
| server2 | 10.0.0.145 | Frigate NVR + Forgejo |
| server3 | 10.0.0.151 | Proxmox, Prometheus + Grafana monitoring stack (node-exporter at 10.0.0.3) |
| cam1 | 10.0.0.201 | Hikvision DS-2CD2112F-I, static |
| cam2 | 10.0.0.202 | Hikvision DS-2CD2112F-I, static |
| cam3 | 10.0.0.203 | Hikvision DS-2CD2112F-I, static |
| cam4 | 10.0.0.204 | Hikvision DS-2CD2112F-I, static |
| ADTRAN Switch | 10.0.0.2 | Static, set via web UI |
| UniFi AP | 10.0.0.162 | DHCP (reserved by MAC) |
| DHCP Range | 10.0.0.100–200 | Assigned by dnsmasq on server |
| Server Tailscale | 100.64.61.90 | Remote access |

## NICs
| Interface | Role | IP |
|---|---|---|
| enp0s31f6 | WAN (to modem) | 192.168.1.34 (DHCP) |
| enp5s0 | LAN (to switch) | 10.0.0.1/24 |

## ISP Modem Port Forwards
| External Port | Protocol | Internal IP | Internal Port | Purpose |
|---|---|---|---|---|
| 80 | TCP | 192.168.1.34 | 80 | Let's Encrypt HTTP challenge + HTTP→HTTPS redirect |
| 443 | TCP | 192.168.1.34 | 443 | HTTPS for all proxied services |
| 51820 | UDP | 192.168.1.34 | 51820 | WireGuard VPN |

## External Domains (via Nginx Proxy Manager + Let's Encrypt)
| Domain | Service | Notes |
|---|---|---|
| `billsserver1.duckdns.org` | Nextcloud | ✅ HTTPS, trusted cert, iOS app works, cert expires Jun 20 2026 |
| `jellyfin.billsserver1.duckdns.org` | Jellyfin | Planned |
| `portainer.billsserver1.duckdns.org` | Portainer | Planned |

See: [nginx-proxy-manager.md](nginx-proxy-manager.md) for setup details.

## Services & Ports
| Service | Port | Protocol | Notes |
|---|---|---|---|
| SSH | 22 | TCP | LAN only |
| Pi-hole web UI | 8081 | TCP | Docker — 8080 reserved for UniFi AP comm |
| Jellyfin | 8096 | TCP | Docker, VA-API transcoding |
| Nextcloud | 8443 | TCP | Docker, direct LAN access (use HTTPS domain for external) |
| WireGuard VPN | 51820 | UDP | wg0, 10.8.0.1/24, DuckDNS: billsserver1.duckdns.org |
| Squid HTTP proxy | 3128 | TCP | Docker |
| Nginx Proxy Manager | 80/443 | TCP | Docker — reverse proxy + SSL termination |
| Nginx Proxy Manager admin | 81 | TCP | Docker — web UI |
| Portainer | 9000 | TCP | Docker |
| Homepage dashboard | 3000 | TCP | Docker — service dashboard with live stats |
| Cockpit | 9090 | TCP | Web console |
| UniFi controller | 8444 | HTTPS | Docker |
| qBittorrent | 8090 | TCP | Docker, routed via Gluetun VPN |
| qBittorrent peers | 6881 | TCP/UDP | Open in nftables |
| Gluetun (Mullvad VPN) | — | — | Docker, WireGuard tunnel for qBittorrent |
| Sonarr | 8989 | TCP | Docker |
| Radarr | 7878 | TCP | Docker |
| Prowlarr | 9696 | TCP | Docker |

## Tailscale

- Server Tailscale IP: `100.64.61.90`
- Fedora Tailscale IP: `100.103.121.79`
- SSH via Tailscale: `ssh adminbill@100.64.61.90`

### Accessing `server2` from the Fedora workstation

Direct SSH from the Fedora workstation to `10.0.0.145` is not currently the
documented working path. The confirmed working route on `2026-04-16` is:

```bash
ssh -F /dev/null adminbill@100.64.61.90
ssh adminbill@10.0.0.145
```

That reaches `server2` by first entering `server1` over Tailscale, then hopping
from `server1` into the LAN host at `10.0.0.145`.

### Critical: Disable accept-dns on the server

The server runs dnsmasq and IS the DNS server for the network. Tailscale's MagicDNS
(`100.100.100.100`) is not a full recursive resolver — it can't look up external domains.
If accept-dns is left enabled (the default), Tailscale overrides the system resolver with
MagicDNS and external DNS lookups fail (SERVFAIL). This breaks things like Claude Code
trying to reach `api.anthropic.com`.

**Fix — run once, persists across reboots:**
```bash
sudo tailscale set --accept-dns=false
```

> If external DNS ever stops working on the server, check this setting first.
> Client machines (Fedora, phone) can leave accept-dns enabled — only the server needs this.

## VLAN Plan (Future)
| VLAN | Name | Subnet | Purpose |
|---|---|---|---|
| 1 | Management | 10.0.0.x | Server, switch, AP management |
| 10 | Trusted | 10.10.0.x | PCs, laptops |
| 20 | IoT | 10.20.0.x | Smart devices, phones |
| 30 | Guest | 10.30.0.x | Visitor WiFi |
