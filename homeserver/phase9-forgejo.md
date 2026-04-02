> **Wiki Notice:** Project documentation should be copied to `/home/adminbill/projects/docs/<project-name>/` and pushed to [homelab-wiki](https://github.com/coocookanoo/homelab-wiki) so it appears at https://wiki.billsserver1.duckdns.org

---

# Phase 9 — Forgejo (Self-Hosted Git Server)

## Overview

Self-hosted Git service on the homeserver — replaces or supplements GitHub.

**Why Forgejo?**
- Lightweight community fork of Gitea (fully free software, no corporate lock-in)
- GitHub-like web UI: repos, issues, pull requests, Kanban project boards, releases, wiki
- Runs in Docker — fits the existing stack
- SSH + HTTPS via Nginx Proxy Manager
- Low resource use — fine on current hardware

**Access:**
- Internal: `http://10.0.0.1:3000`
- External: `https://git.billsserver1.duckdns.org`

## Prerequisites

- Docker + docker compose already working
- Nginx Proxy Manager running
- New DuckDNS subdomain: `git.billsserver1.duckdns.org` (add at duckdns.org, point to your WAN IP)
- Port 3000 free on host

## Step 1 — Create Directory

On the server:

```bash
sudo mkdir -p /data/forgejo/data
sudo chown -R 1000:1000 /data/forgejo
```

## Step 2 — docker-compose.yml

Create `/data/forgejo/docker-compose.yml`:

```yaml
services:
  forgejo:
    image: codeberg.org/forgejo/forgejo:14
    container_name: forgejo
    restart: unless-stopped
    environment:
      - USER_UID=1000
      - USER_GID=1000
    volumes:
      - ./data:/data
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"   # web UI (proxied via NPM)
      - "222:22"      # SSH for git push/pull
```

No custom network needed — NPM proxies to `10.0.0.1:3000` (server LAN IP) rather than container name.

## Step 3 — Start Forgejo

```bash
cd /data/forgejo
docker compose up -d
docker logs forgejo   # confirm it started
```

## Step 4 — Initial Setup

Open `http://10.0.0.1:3000` in a browser from the Fedora PC.

Run the installer:
- **Database:** SQLite (simple, fine for personal use — can migrate to Postgres later)
- **Site title:** Bill's Forgejo
- **Domain:** `git.billsserver1.duckdns.org`
- **SSH port:** `222`
- **HTTP port:** `3000`
- **Admin username:** `adminbill`
- **Email:** `williammcdevitt@gmail.com`
- **Password:** (strong)

Finish setup.

## Step 5 — Nginx Proxy Manager

In NPM Admin (`http://10.0.0.1:81`):

- Add Proxy Host
- **Domain:** `git.billsserver1.duckdns.org`
- **Scheme:** http
- **Forward Hostname:** `10.0.0.1`
- **Forward Port:** `3000`
- **SSL:** Let's Encrypt — DNS challenge with DuckDNS token
- **Force SSL:** on
- **HTTP/2:** on

Save. Access securely at `https://git.billsserver1.duckdns.org`.

## Step 6 — SSH Key Setup

Add your public key in Forgejo → Settings → SSH / GPG Keys → Add Key.

Test the connection:
```bash
ssh -T -p 222 git@10.0.0.1
# Expected: Hi adminbill! You've successfully authenticated...
```

## Step 7 — Migrate Existing Repos

Create repos in Forgejo web UI first, then push from Fedora:

```bash
# prison_cmms
cd ~/projects/prison_cmms
git remote add forgejo ssh://git@10.0.0.1:222/adminbill/prison_cmms.git
git push forgejo --all --tags

# axel-harness
cd ~/projects/axel-harness
git remote add forgejo ssh://git@10.0.0.1:222/adminbill/axel-harness.git
git push forgejo --all --tags
```

To make Forgejo the default remote later:
```bash
git remote set-url origin ssh://git@10.0.0.1:222/adminbill/prison_cmms.git
```

Via domain (external):
```bash
ssh://git@git.billsserver1.duckdns.org:222/adminbill/repo.git
```

## Step 8 — Backups

Add to `/usr/local/bin/server-backup.sh` alongside existing entries:

```bash
# Forgejo
rsync -a --delete /data/forgejo/data/ adminbill@192.168.1.33:/home/adminbill/backups/homeserver/data/forgejo/
```

Disk3 local backup:
```bash
sudo rsync -a --delete /data/forgejo/ /data/disk3/backups/server/docker/forgejo/
```

Update the backup table in `phase7-backup.md`:

| Source | Destination |
|--------|-------------|
| `/data/forgejo/data/` | `backups/homeserver/data/forgejo/` |

## Firewall Notes

- LAN (10.0.0.0/24 and 192.168.1.0/24) already allowed by nftables rules.
- External: only expose via NPM (80/443). Do **not** open port 3000 or 222 directly to the internet.
- If you want external SSH git access via port 222 in the future, add a specific nftables rule and consider fail2ban first.

## VS Code Workflow

- GitLens works with any remote — no changes needed
- Push/pull using the `forgejo` remote as above
- Project boards: manage in the Forgejo web UI (Kanban — same idea as GitHub Projects)
- Keep GitHub as a mirror for public repos if wanted

## Optional Improvements

- **PostgreSQL:** Replace SQLite by adding a `db` service — better for larger repos or multiple users
- **Forgejo Actions:** Add a runner for CI/CD (lightweight, similar to GitHub Actions)
- **Email notifications:** Configure SMTP in Forgejo admin panel
- **Mirror from GitHub:** Forgejo can auto-mirror a GitHub repo on a schedule

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Container won't start | Check `docker logs forgejo` — usually a permissions issue on `/data/forgejo/data` |
| 502 in NPM | Verify container is running and NPM forward IP/port is correct (`10.0.0.1:3000`) |
| SSH push fails | Confirm key is added in Forgejo settings; test with `ssh -T -p 222 git@10.0.0.1` |
| Wrong domain in clone URLs | Re-check the domain set during initial install (edit `app.ini` in `data/gitea/conf/`) |
