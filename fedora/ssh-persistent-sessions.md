# SSH Persistent Sessions (Fedora Desktop)

**Last updated:** 2026-04-19 — Initial setup. Never lose a session again.

## Goal

SSH sessions must never drop — not on disconnect, not on reboot. Work continues exactly where you left it every time.

## How It Works

| Layer | What it does |
|---|---|
| **sshd keepalives** | Server pings client every 60s, tolerates 10 missed pings before dropping |
| **tmux systemd service** | `main` tmux session auto-starts at boot, linger keeps it alive with no user logged in |
| **Auto-attach on SSH** | Every SSH login drops straight into the `main` tmux session |

## sshd Keepalives

Added to `/etc/ssh/sshd_config`:

```
ClientAliveInterval 60
ClientAliveCountMax 10
TCPKeepAlive yes
```

Apply changes:
```bash
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
loginctl enable-linger adminbill
```

The `enable-linger` line is critical — it makes the user service start at boot even before login.

## Auto-Attach on SSH Login

Added to `~/.bashrc`:

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

- [[fedora/PLAN]] — Fedora desktop plan
- [[homeserver/phase3-vpn]] — WireGuard VPN (remote access)
