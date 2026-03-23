> **Wiki Notice:** Project documentation should be copied to `/home/adminbill/projects/docs/<project-name>/` and pushed to [homelab-wiki](https://github.com/coocookanoo/homelab-wiki) so it appears at https://wiki.billsserver1.duckdns.org

---

# Phase 0 — Install Debian 12

## Get Debian
Download Debian 12 netinstall ISO:
https://www.debian.org/distrib/netinst

Use the amd64 netinst image (~400MB).

## Flash to USB
From another Linux machine:
```bash
# Find your USB device first
lsblk

# Flash it (replace sdX with your USB device — BE CAREFUL)
sudo dd if=debian-12-netinst-amd64.iso of=/dev/sdX bs=4M status=progress conv=fsync
```

Or use Balena Etcher (GUI tool, easier).

## Install Options
Boot from USB, select "Install" (not graphical).

### Partition Setup (240GB HDD)
| Partition | Size | Mount | Type |
|---|---|---|---|
| /boot | 512MB | /boot | ext4 |
| swap | 4GB | swap | swap |
| / (root) | 60GB | / | ext4 |
| /data | rest (~175GB) | /data | ext4 |

Put services + Docker data on /data so it's separate from OS.

### Software Selection
At the tasksel screen, deselect everything EXCEPT:
- [x] SSH server
- [x] Standard system utilities

No desktop environment. No print server. Minimal is best.

## First Boot Checks
```bash
# Confirm you can SSH in
ssh admin@<ip>

# Check interfaces
ip link

# Check disk
lsblk

# Update system
sudo apt update && sudo apt upgrade -y
```

## Identify Your NICs
```bash
ip link show
```
You'll see something like eth0, eth1 or enp2s0, enp3s0.
Plug a cable into each one individually and run `ip link` to see which lights up.
Note which is WAN (to modem) and which is LAN (to your network).
