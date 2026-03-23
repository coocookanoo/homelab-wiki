# Phase 3 — WireGuard VPN

## Overview

- **Interface**: wg0, `10.8.0.1/24`
- **Port**: UDP 51820 (port forwarded on AT&T Gateway 5)
- **DDNS**: billsserver1.duckdns.org (auto-updated every 5 min via cron)
- **Client**: iPhone — full tunnel (all traffic), DNS via Pi-hole at 10.8.0.1

## Install

```bash
sudo apt install -y wireguard-tools qrencode
```

## Key Files

| File | Purpose |
|---|---|
| `/etc/wireguard/wg0.conf` | Server config + iPhone peer |
| `/etc/wireguard/server_private.key` | Server private key |
| `/etc/wireguard/server_public.key` | Server public key |
| `/etc/wireguard/iphone_private.key` | iPhone private key |
| `/etc/wireguard/iphone_public.key` | iPhone public key |
| `/etc/wireguard/iphone_psk.key` | Pre-shared key |
| `/etc/wireguard/iphone.conf` | iPhone client config (for QR regeneration) |
| `/etc/wireguard/wg0-up.sh` | PostUp hook — adds nftables forward rules for wg0 |
| `/etc/wireguard/wg0-down.sh` | PreDown hook — restores base nftables ruleset |
| `/etc/wireguard/duckdns-update.sh` | DuckDNS IP update script |

## nftables Rules Required

These live permanently in `/etc/nftables.conf`:

```
# INPUT — must be BEFORE the WAN drop rule
iif "enp0s31f6" udp dport 51820 accept
iif "enp0s31f6" drop   # ← 51820 accept must come before this
```

The wg0 FORWARD rules are added via PostUp (wg0-up.sh) since wg0 doesn't
exist at boot when nftables loads. wg0-up.sh adds:

```bash
nft add rule inet filter forward iifname "wg0" oifname "enp0s31f6" accept
nft add rule inet filter forward iifname "enp0s31f6" oifname "wg0" ct state established,related accept
```

NAT masquerade (`oif "enp0s31f6" masquerade`) in the base config covers WireGuard traffic automatically.

### Critical Gotcha: Rule Ordering

`udp dport 51820 accept` MUST appear before `iif "enp0s31f6" drop` in the INPUT
chain. If PostUp appends the rule, it ends up after the drop and WireGuard handshakes
silently fail. Keep this rule in the base nftables.conf, not in PostUp.

## DuckDNS Auto-Update (cron)

```
*/5 * * * * /bin/bash /etc/wireguard/duckdns-update.sh
```

Token and domain stored in `/etc/wireguard/duckdns-update.sh`.

## Enable at Boot

```bash
sudo systemctl enable wg-quick@wg0
```

## Regenerate iPhone QR Code

```bash
sudo cat /etc/wireguard/iphone.conf | qrencode -t ansiutf8
```

## Add a New Client

```bash
cd /etc/wireguard && umask 077
wg genkey | tee client2_private.key | wg pubkey > client2_public.key
wg genpsk > client2_psk.key
```

Add to wg0.conf:
```ini
[Peer]
PublicKey = <client2_public_key>
PresharedKey = <client2_psk>
AllowedIPs = 10.8.0.3/32
```

Reload without dropping connections:
```bash
sudo wg syncconf wg0 <(wg-quick strip /etc/wireguard/wg0.conf)
```
