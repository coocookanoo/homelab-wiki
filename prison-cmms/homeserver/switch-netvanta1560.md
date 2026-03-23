# ADTRAN NetVanta 1560-08-65W Switch

## Hardware
- **Model:** NetVanta 1560-08-65W
- **Ports:** 10x 10/100/1000Base-T + 2x 1Gbps SFP
- **PoE:** 65W total budget (can power UniFi AP)
- **Layer:** Layer 3 (can route between VLANs)
- **Console:** RJ-45 + Micro USB on front panel

## Access
| Method | Address | Notes |
|---|---|---|
| Web UI | http://10.0.0.2 | HTTPS also available ✅ Active |
| SSH | ssh admin@10.0.0.2 | Requires legacy SSH flags (see below) |
| Console | /dev/ttyUSB0 or /dev/ttyACM0 | USB-A to Micro USB cable, 9600 baud |

## Credentials
| | |
|---|---|
| Username | `admin` |
| Password | `adminbill` |

## SSH Legacy Flags (Fedora)
The switch uses old SSH algorithms. Connect with:
```bash
ssh -oHostKeyAlgorithms=+ssh-rsa \
    -oPubkeyAcceptedKeyTypes=+ssh-rsa \
    -oKexAlgorithms=+diffie-hellman-group14-sha1 \
    -oCiphers=+aes256-cbc,aes128-cbc,3des-cbc \
    -oMACs=+hmac-sha1 \
    admin@10.0.0.2
```

## Port Layout
| Port | Connected To | VLAN | Notes |
|---|---|---|---|
| GigabitEthernet 1/5 | Ubuntu Server (enp4s0) | Trunk | Uplink to router |
| GigabitEthernet 1/1 | UniFi AP | VLAN 1 | PoE powered |
| GigabitEthernet 1/3 | Fedora PC | VLAN 1 | Dev workstation |
| GigabitEthernet 1/4–1/10 | — | — | Unused |
| SFP 1–2 | — | — | Unused |

> Update this table as you plug things in.

## Factory Reset
1. Power on the switch
2. Hold **MODE button** on front panel for ~5 seconds
3. Release when port LEDs change
4. Wait 60 seconds for reboot
5. Default IP: DHCP from network, fallback to `10.10.10.1` after 60s

## Default Config Behavior
- Tries DHCP on boot, falls back to `10.10.10.1/24` after 60 seconds
- All ports on VLAN 1 untagged
- SNMP disabled by default (good)
- HTTPS enabled by default

## Static IP Configuration
Set via web UI: System → IP Configuration → VLAN 1 interface
- IP: `10.0.0.2` ✅ Configured
- Mask: `255.255.255.0` (mask length 24)
- Gateway: `10.0.0.1`
- DNS: `10.0.0.1`

## VLAN Plan (Future)
| VLAN | Name | Subnet | Purpose |
|---|---|---|---|
| 1 | Management | 10.0.0.x | Server, switch, AP management |
| 10 | Trusted | 10.10.0.x | PCs, laptops |
| 20 | IoT | 10.20.0.x | Smart devices, phones |
| 30 | Guest | 10.30.0.x | Visitor WiFi |

## Key Features Available
- 4,095 VLANs, 802.1Q trunking
- DHCP Snooping (prevents rogue DHCP)
- ARP Inspection
- 802.1x port authentication
- Port mirroring (traffic monitoring)
- LACP link aggregation (bond ports for more bandwidth)
- QoS with 8 hardware queues per port
- Static routing between VLANs (Layer 3)
