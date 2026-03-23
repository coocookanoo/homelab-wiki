> **Wiki Notice:** Project documentation should be copied to `/home/adminbill/projects/docs/<project-name>/` and pushed to [homelab-wiki](https://github.com/coocookanoo/homelab-wiki) so it appears at https://wiki.billsserver1.duckdns.org

---

# Wiki.js — Documentation Wiki

Self-hosted wiki synced from GitHub, serving docs for all projects.

- **Local URL**: http://10.0.0.1:3010
- **Public URL**: https://wiki.billsserver1.duckdns.org
- **Docker container**: `wikijs`
- **Database container**: `wikijs-db` (PostgreSQL)

## Login

| Field | Value |
|---|---|
| Email | `williammcdevitt@gmail.com` |
| Username | `adminbill` |
| Password | *(see project password manager)* |

## Git Sync

Syncs from `git@github.com:coocookanoo/homelab-wiki.git` (master branch) every 5 minutes.

- SSH deploy key: `/data/wikijs/ssh/id_ed25519`
- To force sync: Administration → Storage → Git → **Sync Now**

## Adding New Docs

1. Add `.md` files to `/docs/` in the prison-cmms repo on your dev machine
2. `git push`
3. Wiki.js pulls automatically within 5 minutes

## Data Locations (on server)

| Path | Purpose |
|---|---|
| `/data/wikijs/db/` | PostgreSQL data |
| `/data/wikijs/ssh/` | SSH deploy key for git sync |
