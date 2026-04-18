# Phase 10 — Docs Sync (server1 -> server2)

## Overview

`server1` is the source of truth for the homeserver docs.
`server2` keeps a local copy at `/home/adminbill/projects/docs/`, but docs should not
change on `server2` unless a sync is run directly by the user.

## Current State

- Source docs host: `server1` (`10.0.0.1`)
- Source docs path on `server1`: `/home/adminbill/homeserver/docs/`
- Source README on `server1`: `/home/adminbill/homeserver/README.md`
- Fallback SSH host: `100.64.61.90`
- Destination on `server2`: `/home/adminbill/docs/`
- Initial sync from `server1` to `server2` completed on `2026-04-16`
- Full homeserver docs were pulled to `server2` on `2026-04-16`
- `server2` was rebuilt on `2026-04-17`; docs currently live directly in `/home/adminbill/docs/`
- No local `~/.local/bin/sync-docs` helper is present on the rebuilt `server2` as of `2026-04-17`

## Sync Policy

- `server1` remains the main doc set
- Sync is one-way: `server1` -> `server2`
- Sync is manual only
- Default behavior must preview changes first
- Deletions from `server1` should not be mirrored unless explicitly requested

## Current Sync Method

Use manual `rsync` from `server1` until a new helper is recreated on `server2`.
Preview first:

```bash
rsync -avzn adminbill@10.0.0.1:/home/adminbill/homeserver/docs/ /home/adminbill/docs/
```

Apply:

```bash
rsync -avz adminbill@10.0.0.1:/home/adminbill/homeserver/docs/ /home/adminbill/docs/
```

Fallback over Tailscale:

```bash
rsync -avz adminbill@100.64.61.90:/home/adminbill/homeserver/docs/ /home/adminbill/docs/
```

## Commands

Preview only:

```bash
rsync -avzn adminbill@10.0.0.1:/home/adminbill/homeserver/docs/ /home/adminbill/docs/
```

Apply sync after confirmation:

```bash
rsync -avz adminbill@10.0.0.1:/home/adminbill/homeserver/docs/ /home/adminbill/docs/
```

Apply an exact mirror after confirmation:

```bash
rsync -avz --delete adminbill@10.0.0.1:/home/adminbill/homeserver/docs/ /home/adminbill/docs/
```

## Ubuntu 24.04 Codex Sandbox Fix on server2

After the `server2` rebuild on `2026-04-17`, Codex could see only its vendored
`bubblewrap` and sandbox startup failed with:

```text
bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted
```

Fix applied on `server2`:

```bash
sudo apt update
sudo apt install -y bubblewrap
```

Ubuntu 24.04 had `kernel.apparmor_restrict_unprivileged_userns=1`, so `/usr/bin/bwrap`
needed its own AppArmor profile instead of falling back to the restrictive
`unprivileged_userns` profile.

Installed profile:

```text
/etc/apparmor.d/bwrap
```

Profile contents:

```text
# This profile exists so bubblewrap can create user/network namespaces
# without being forced into Ubuntu's restrictive unprivileged_userns profile.

abi <abi/4.0>,
include <tunables/global>

profile bwrap /usr/bin/bwrap flags=(unconfined) {
  userns,

  include if exists <local/bwrap>
}
```

Load or reload it with:

```bash
sudo apparmor_parser -r /etc/apparmor.d/bwrap
```

Verify:

```bash
which bwrap
bwrap --version
```

## Notes

- This keeps docs on `server2` available locally without making `server2` authoritative
- Avoid using raw `rsync --delete` unless an exact mirror is intentionally wanted
- If additional homeserver docs are created on `server1`, pull from `/home/adminbill/homeserver/docs/`
- Recreate `~/.local/bin/sync-docs` later if a guarded wrapper is still wanted on `server2`
