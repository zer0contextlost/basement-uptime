---
title: "Fixing UID/GID Permission Hell When Bind-Mounting Host Storage Into Unprivileged LXC"
date: 2026-08-06
description: "Why files bind-mounted into an unprivileged Proxmox LXC show up as nobody:nogroup, and how to fix it properly with subuid/subgid mapping instead of going privileged."
tags: ["proxmox", "lxc"]
---

Bind-mount a host directory into an unprivileged LXC container and watch
every file inside turn into `nobody:nogroup`, or worse, `65534:65534`.
That's the UID mapping problem. It looks like a bug the first time you hit
it. It's actually the container isolation working exactly as designed,
just not the way most people expect.

## Why this happens

Unprivileged LXC containers remap UIDs/GIDs so that root inside the
container (UID 0) is *not* root on the host. Container UID 0 maps to some
high UID on the host, by default something like 100000. The whole UID
range inside the container (0-65535) shifts up by that offset on the host
side.

That's good for security. If something breaks out of the container, it
doesn't get host root. But it means a bind-mounted host directory shows up
with whatever the *mapped* UID/GID resolves to, not the UID/GID you're
used to. A file owned by `1000:1000` on the host shows up as some huge,
meaningless UID inside the container, and Proxmox renders that as
`nobody:nogroup` because there's no matching entry in the container's
passwd file.

The two bad fixes people reach for: making the container privileged
(defeats the point of using unprivileged containers) or `chmod -R 777` on
the mount (defeats the point of having permissions at all). Neither is
necessary.

## The real fix: idmap entries

Proxmox lets you add explicit UID/GID mapping entries per container so
specific host UID ranges pass through unshifted, or shifted to a value you
choose, instead of using the default offset for everything.

Check what subuid/subgid ranges are allocated to the container's user
(usually `root` on the Proxmox host) in `/etc/subuid` and `/etc/subgid`.
You'll see something like:

```
root:100000:65536
```

That's the default 65536-UID block starting at 100000. To carve out an
exception for a specific UID, say your media stack runs as UID 1000 on
both host and container, add `lxc.idmap` lines to the container config
(`/etc/pve/lxc/<vmid>.conf`):

```
lxc.idmap: u 0 100000 1000
lxc.idmap: u 1000 1000 1
lxc.idmap: u 1001 101001 64535
lxc.idmap: g 0 100000 1000
lxc.idmap: g 1000 1000 1
lxc.idmap: g 1001 101001 64535
```

Three ranges per type, user and group: everything below 1000 uses the
normal offset mapping, UID 1000 itself maps straight through to host UID
1000, everything above 1000 goes back to the normal offset. Container UID
1000 and host UID 1000 now refer to the same identity, so a bind-mounted
directory owned by `1000:1000` on the host is readable and writable as the
expected user inside the container, no offset weirdness.

The corresponding ranges also need to actually be present in
`/etc/subuid`/`/etc/subgid` for the mapping user. If the range isn't
allocated there, the container fails to start with a permission error on
boot. That's the most common way people discover this feature exists in
the first place.

## A simpler alternative for Docker-in-LXC setups

Running Docker inside the LXC, a common pattern for self-hosted app
stacks, it's often easier to standardize on one UID/GID for everything.
Set `PUID`/`PGID` environment variables to `1000:1000` on every container
image that supports it (most linuxserver.io images do), create a matching
user on the Proxmox host, and use the idmap trick above once. Every bind
mount, every app, every backup script agrees on the same numbers after
that.

## Checking your work

After adding idmap entries and restarting the container, confirm from
inside the container:

```
stat -c '%u:%g' /path/to/bind/mount
ls -ln /path/to/bind/mount
```

Cross-check against `ls -ln` on the host side for the same directory. If
the numeric UID/GID match on both sides, the mapping is correct, and you
shouldn't need `chown` gymnastics or loosened permissions for the
container's processes to read and write the data.

It's a few minutes of config editing instead of running privileged
containers you don't need, or leaving directories world-writable because
it was easier than fixing the mapping. Do it once for your standard UID
and it's a copy-paste for every unprivileged container after that.
