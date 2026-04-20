# SSH Persistent Sessions

**Last updated:** 2026-04-19 — Applied to Fedora workstation + all 3 servers.

## Scope

| Host | Address | User | Status |
|---|---|---|---|
| Fedora workstation | local | adminbill | Done |
| server1 | 192.168.1.34 | adminbill | Done |
| server2 | 10.0.0.145 | adminbill | Done |
| server3 (Proxmox) | 10.0.0.3 | root | Done |

## Goal

SSH sessions never drop — not on disconnect, not on reboot. Reconnect and pick up exactly where you left off.

## How It Works

| Layer | What it does |
|---|---|
| **sshd keepalives** | Server pings client every 60s, tolerates 10 missed pings before dropping |
| **tmux systemd service** | `main` tmux session auto-starts at boot, linger keeps it alive with no user logged in |
| **Auto-attach on SSH** | Every SSH login drops straight into the `main` tmux session |

## sshd Keepalives

Added to `/etc/ssh/sshd_config` on each host:

```
ClientAliveInterval 60
ClientAliveCountMax 10
TCPKeepAlive yes
```

Apply changes:
```bash
# Ubuntu/Debian (server1, server2, server3)
sudo systemctl restart ssh

# Fedora (workstation)
sudo systemctl restart sshd
```

## tmux Systemd User Service

File: `~/.config/systemd/user/tmux-main.service`

```ini
[Unit]
Description=tmux main session
After=network.target

[Service]
Type=forking
ExecStart=/usr/bin/tmux new-session -d -s main
ExecStop=/usr/bin/tmux kill-session -t main
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

Enable and start:
```bash
systemctl --user daemon-reload
systemctl --user enable --now tmux-main.service
loginctl enable-linger $USER
```

The `enable-linger` line is critical — it makes the user service start at boot even before login.

## Auto-Attach on SSH Login

Added to `~/.bashrc` on each host:

```bash
if [ -n "$SSH_CONNECTION" ] && [ -z "$TMUX" ]; then
    tmux attach-session -t main || tmux new-session -s main
fi
```

## Common Commands

```bash
systemctl --user status tmux-main.service   # check tmux service
tmux ls                                      # list sessions
tmux attach -t main                          # manually attach
tmux new-window                              # open new window (Ctrl+b c)
tmux kill-session -t main                   # nuclear option — destroys session
```

## tmux Basics

| Key | Action |
|---|---|
| `Ctrl+b c` | New window |
| `Ctrl+b n` | Next window |
| `Ctrl+b p` | Previous window |
| `Ctrl+b d` | Detach (session keeps running) |
| `Ctrl+b [` | Scroll mode (q to exit) |

## See Also

- [[homeserver/SERVER_REFERENCE]] — Server addresses and users
- [[homeserver/phase3-vpn]] — WireGuard VPN (remote access)
