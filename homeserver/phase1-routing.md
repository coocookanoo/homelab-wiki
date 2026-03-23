> **Wiki Notice:** Project documentation should be copied to `/home/adminbill/projects/docs/<project-name>/` and pushed to [homelab-wiki](https://github.com/coocookanoo/homelab-wiki) so it appears at https://wiki.billsserver1.duckdns.org

---

# Phase 1 — Routing & Firewall

## Enable IP Forwarding
```bash
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Install nftables
```bash
sudo apt install -y nftables
sudo systemctl enable nftables
```

## Configure nftables
Edit /etc/nftables.conf — replace WAN_IF and LAN_IF with your actual interface names.

See: ../configs/nftables/nftables.conf

```bash
sudo cp configs/nftables/nftables.conf /etc/nftables.conf
sudo systemctl restart nftables
sudo nft list ruleset  # verify rules loaded
```

## Configure Network Interfaces

Ubuntu 24.04 uses netplan. Edit `/etc/netplan/50-cloud-init.yaml`:
```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp0s31f6:       # WAN — to ISP modem
      dhcp4: true
    enp4s0:          # LAN — to switch
      addresses:
        - 10.0.0.1/24
```

**Important:** Use `renderer: NetworkManager` — without it, NetworkManager reports "disconnected"
and Cockpit's package manager will refuse to refresh (even though networking works fine).
Also disable the now-redundant networkd wait service to avoid a 2-minute boot delay:

```bash
sudo netplan apply
sudo systemctl disable systemd-networkd-wait-online.service
```

## Install dnsmasq (DHCP + DNS)
```bash
sudo apt install -y dnsmasq
```

dnsmasq starts before the LAN interface is fully up at boot and fails with "unknown interface".
Fix with a systemd drop-in:
```bash
sudo mkdir -p /etc/systemd/system/dnsmasq.service.d
sudo tee /etc/systemd/system/dnsmasq.service.d/wait-for-network.conf << 'EOF'
[Unit]
After=network-online.target
Wants=network-online.target
EOF
sudo systemctl daemon-reload
```

See: ../configs/dnsmasq.conf for full config.

Key settings:
```
interface=enp4s0
dhcp-range=10.0.0.100,10.0.0.200,24h
dhcp-option=3,10.0.0.1    # gateway
dhcp-option=6,10.0.0.1    # DNS server (points to Pi-hole later)
```

## nftables: Virtual Interface Gotcha (tailscale0, wg0)

`iif "interface"` requires the interface to **exist when nftables loads**. Virtual interfaces
like `tailscale0` are created by their daemons after nftables starts — causing nftables to
fail at boot, which cascades into wg-quick also failing.

Use `iifname` instead (string match at packet time, no existence check at load time):
```
# Wrong — fails at boot if interface doesn't exist yet
iif "tailscale0" accept

# Correct
iifname "tailscale0" accept
```

Physical interfaces (enp0s31f6, enp4s0) are fine with `iif` since they exist at boot.

## nftables: Home Devices Are on the WAN Interface

The Fedora PC (192.168.1.33) and server WAN (192.168.1.34) are on the **same ISP subnet**.
Traffic from the Fedora PC hits `enp0s31f6` (WAN interface) — not the LAN interface.

nftables drops all uninitiated traffic on the WAN interface by default. Without an explicit
allow rule, services like Cockpit (9090), NPM admin (81), etc. are unreachable from the
Fedora PC even though it's on the same home network.

The current config allows all 192.168.1.x devices full access:
```
iif $WAN_IF ip saddr 192.168.1.0/24 accept
```

This rule must appear **before** the `iif $WAN_IF drop` rule in the input chain.
It's already in `/etc/nftables.conf` — don't remove it.

## Docker + nftables: Forward Chain Gotcha

When Docker services (e.g. Pi-hole DNS on port 53) are accessed from LAN clients, iptables
DNATs the traffic to the container's bridge IP (172.18.x.x). nftables sees this as a
forward from `enp4s0` → docker bridge and drops it if there's no matching rule.

Add these two rules to the `chain forward` block in nftables.conf:

```
# Allow LAN clients to reach Docker containers (post-DNAT)
ip daddr 172.16.0.0/12 accept

# Allow Docker container responses back to LAN
ip saddr 172.16.0.0/12 oif "enp4s0" accept
```

Without these, LAN devices get DHCP leases and can route to the internet, but cannot
reach any Dockerized service (DNS, Jellyfin, Nextcloud, etc.) via the host IP.

## Test Routing
From a device on your LAN:
- Can you get an IP via DHCP? `ip a`
- Can you ping the gateway? `ping 10.0.0.1`
- Can you reach the internet? `ping 8.8.8.8`
- Can you resolve DNS? `nslookup google.com 10.0.0.1`
