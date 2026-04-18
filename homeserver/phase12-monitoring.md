# Phase 12 — Monitoring (Prometheus + Grafana)

## Overview

Centralised metrics monitoring for all servers and the Fedora workstation.
Stack runs in **Proxmox LXC container 101** (`monitoring`) on server3.

```
node-exporter (server1, server2, server3, fedora)
        ↓  :9100
snmp_exporter (CT 101 — scrapes switch via UDP 161)
        ↓
  Prometheus (CT 101 — 10.0.0.151:9090)
        ↓
    Grafana (CT 101 — 10.0.0.151:3000)
        ↓
       You
```

## Access

| Service | URL |
|---------|-----|
| Grafana | http://10.0.0.151:3000 |
| Prometheus | http://10.0.0.151:9090 |
| Proxmox Web UI | https://10.0.0.3:8006 — user: `admin@pve` |

**Grafana login:** admin / 21217Wam5085

**Dashboards:**
- Node Exporter Full (ID 1860) — server/workstation metrics
- SNMP Stats (ID 11169) — switch port traffic, errors, uptime

## Prometheus Config

Located at `/etc/prometheus/prometheus.yml` inside CT 101.

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node"
    static_configs:
      - targets: ["10.0.0.1:9100"]
        labels:
          alias: server1
      - targets: ["10.0.0.145:9100"]
        labels:
          alias: server2
      - targets: ["10.0.0.3:9100"]
        labels:
          alias: server3
      - targets: ["192.168.1.33:9100"]
        labels:
          alias: fedora
```

All targets share the `node` job so the Grafana dashboard instance dropdown works cleanly. Use the `alias` label to identify hosts.

## SNMP Switch Monitoring

The NetVanta switch (`10.0.0.2`) is monitored via `snmp_exporter` running in CT 101.

### Switch SNMP Config

- **Mode:** SNMPv2c, read-only
- **Community:** `homelab`
- **Allowed source:** `10.0.0.0/24` (LAN only)
- Configured via switch web UI at `http://10.0.0.2` → SNMP → Communities

To reach the switch web UI from Fedora (it's LAN-only):
```bash
ssh -L 8080:10.0.0.2:80 adminbill@192.168.1.34 -N &
# then open http://localhost:8080
```

### snmp_exporter Config

- **Service:** `prometheus-snmp-exporter` (systemd, inside CT 101)
- **Port:** 9116
- **Config:** `/etc/prometheus/snmp.yml` inside CT 101
- **Auth name:** `homelab_v2` (SNMPv2c, community: homelab)
- **Module:** `if_mib` — walks interface table OIDs for port stats

Prometheus scrape job in `prometheus.yml`:
```yaml
  - job_name: "snmp"
    static_configs:
      - targets: ["10.0.0.2"]
        labels:
          alias: switch
    metrics_path: /snmp
    params:
      module: [if_mib]
      auth: [homelab_v2]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9116
```

### Metrics collected

| Metric | OID | Description |
|--------|-----|-------------|
| ifOperStatus | 1.3.6.1.2.1.2.2.1.8 | Port up/down (1=up 2=down) |
| ifAdminStatus | 1.3.6.1.2.1.2.2.1.7 | Admin enabled/disabled |
| ifSpeed | 1.3.6.1.2.1.2.2.1.5 | Port speed (bps) |
| ifHCInOctets | 1.3.6.1.2.1.31.1.1.1.6 | Bytes in (64-bit) |
| ifHCOutOctets | 1.3.6.1.2.1.31.1.1.1.10 | Bytes out (64-bit) |
| ifInErrors | 1.3.6.1.2.1.2.2.1.14 | Inbound errors |
| ifOutErrors | 1.3.6.1.2.1.2.2.1.20 | Outbound errors |
| sysUpTime | 1.3.6.1.2.1.1.3 | Switch uptime |

### Grafana switch dashboard

URL: `http://10.0.0.151:3000/d/7qKD6I1Wk/snmp-stats`

Set the **target** variable to `10.0.0.2` in the dashboard dropdowns.

### Adding another SNMP device

1. Enable SNMP on the device with community `homelab`
2. Add to Prometheus `snmp` job targets in `/etc/prometheus/prometheus.yml`
3. Reload Prometheus: `pkill -HUP prometheus` inside CT 101

## node-exporter Installs

| Host | Install method | Service |
|------|---------------|---------|
| server1 | apt | `node_exporter` |
| server2 | apt | `node_exporter` |
| server3 | apt (inside CT 101) | `node_exporter` |
| Fedora | `dnf install prometheus-node-exporter` | `prometheus-node-exporter` |

## Adding a New Host

1. Install node-exporter on the new host
2. Edit `/etc/prometheus/prometheus.yml` inside CT 101
3. Add a new entry under the `node` job with an `alias` label
4. Reload: `pkill -HUP prometheus` inside CT 101

## Accessing CT 101 (monitoring container)

From server3 (Proxmox host):
```bash
pct exec 101 -- bash
```

From Fedora:
```bash
ssh ubuntuserver3
pct exec 101 -- bash
```

## Proxmox SSH Access

```
ssh ubuntuserver3   # alias in ~/.ssh/config → root@10.0.0.3 via ubuntuserver1
```
