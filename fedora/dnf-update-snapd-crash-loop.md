# dnf upgrade — snapd crash-loops after selinux-policy update

## Problem

After a routine `sudo dnf upgrade` that includes a `selinux-policy` bump (and
usually a kernel bump alongside it), `snapd.service` can end up crash-looping:

```
systemctl status snapd
# Active: failed (Result: exit-code)
# snapd.service: Unable to locate executable '/usr/libexec/snapd/snapd': Permission denied
# Main process exited, code=exited, status=203/EXEC
```

Desktop notifications (`setroubleshoot`) flood in — "SELinux is preventing
snapd from read access on the netlink_kobject_uevent_socket", once or more
per second, alongside unrelated `dockerd` "use access on the fd fd" denials.

## Root cause

`ausearch -m avc` around the failure timestamp shows the real denial:

```
avc: denied { execute } for pid=... comm="(snapd)" name="snapd" ...
  scontext=system_u:system_r:init_t:s0 tcontext=system_u:object_r:unlabeled_t:s0
  tclass=file permissive=0 trawcon="system_u:object_r:snappy_exec_t:s0"
```

`tcontext=unlabeled_t` is the tell: mid-transaction, the freshly-reinstalled
`/usr/libexec/snapd/snapd` binary was briefly unlabeled before SELinux
context restoration caught up, and systemd tried to (re)start the
socket-activated service *during* that exact window — repeatedly, since
`snapd.socket` retriggers it on every connection attempt. By the time you
check `ls -Z /usr/libexec/snapd/snapd`, the context is already correct
(`snappy_exec_t`) — the race has already resolved itself, snapd just never
noticed and stays wedged in `failed`.

The netlink/fd denials from snapd and dockerd are a **separate, unrelated,
non-fatal** long-standing Fedora policy gap — noisy but don't break either
service (both stay `active`). Not worth chasing; leave them.

## Fix

```
sudo systemctl reset-failed snapd.service
sudo systemctl restart snapd.service
systemctl status snapd.service   # should show active (running), no more crash loop
```

No `restorecon` needed — by the time you're fixing this, the context has
already self-corrected; the service just needs a kick.

## Also check after any kernel-bumping update

`akmod-nvidia` needs to rebuild against the new kernel before reboot — it
does this automatically via `akmods`, but confirm before rebooting if this
box has previously had NVIDIA/kernel-mismatch boot lockouts:

```
rpm -q kernel --last | head -1        # newest installed kernel
modinfo nvidia | grep filename        # which kernel the current .ko is built for
```
