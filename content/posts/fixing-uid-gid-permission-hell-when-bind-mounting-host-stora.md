---
title: "Fixing UID/GID Permission Hell When Bind-Mounting Host Storage Into Unprivileged LXC"
date: 2026-08-06
description: "Why files bind-mounted into an unprivileged Proxmox LXC show up as nobody:nogroup, and how to fix it properly with subuid/subgid mapping instead of going privileged."
tags: ["proxmox", "lxc"]
---

If you've ever bind-mounted a host directory into an unprivileged LXC container and watched every file inside turn into `nobody:nogroup` (or worse, `65534:65534`), you've run into the UID mapping problem. It's one of those things that seems like a bug the first time you hit it, but it's actually the container isolation working exactly as designed — just not the way most people expect.

## Why this happens

Unprivileged LXC containers remap UIDs/GIDs so that root inside the container (UID 0) is *not* root on the host. Instead, container UID 0 maps to some high UID on the host — by default something like 100000. The whole UID range inside the container (0-65535) gets shifted up by that offset on the host side.

This is great for security: if something breaks out of the container, it doesn't have host root. But it means that when you bind-mount a host directory into the container, the ownership the container sees is whatever the *mapped* UID/GID resolves to, not the UID/GID you're used to. A file owned by `1000:1000` on the host shows up as some huge, meaningless UID inside the container, and Proxmox just renders that as `nobody:nogroup` because there's no matching entry in the container's passwd file.

The two bad "fixes" people reach for are making the container privileged (defeats the point of using unprivileged containers) or `chmod -R 777` on the mount (defeats the point of having permissions at all). Neither is necessary.

## The real fix: idmap entries

Proxmox lets you add explicit UID/GID mapping entries per container so that specific host UID ranges pass through unshifted (or shifted to a value you choose) instead of using the default offset for everything.

First, check what subuid/subgid ranges are allocated to the container's user (usually `root` on the Proxmox host) in `/etc/subuid` and `/etc/subgid`. You'll see something like:

```
root:100000:65536
```

That's the default 65536-UID block starting at 100000. To carve out an exception for a specific UID — say your media stack runs as UID 1000 on both host and container — you add `lxc.idmap` lines to the container config (`/etc/pve/lxc/<vmid>.conf`):

```
lxc.idmap: u 0 100000 1000
lxc.idmap: u 1000 1000 1
lxc.idmap: u 1001 101001 64535
lxc.idmap: g 0 100000 1000
lxc.idmap: g 1000 1000 1
lxc.idmap: g 1001 101001 64535
```

Read this as three ranges per type (user/group): everything below 1000 uses the normal offset mapping, UID 1000 itself maps straight through to host UID 1000, and everything above 1000 goes back to the normal offset. This lets container UID 1000 and host UID 1000 refer to the same identity, so a bind-mounted directory owned by `1000:1000` on the host is readable and writable as the expected user inside the container — no offset weirdness.

For this to work, the corresponding ranges also need to actually be present in `/etc/subuid`/`/etc/subgid` for the mapping user. If the range isn't allocated there, the container will fail to start with a permission error on boot, which is the most common way people discover this feature exists in the first place.

## A simpler alternative for Docker-in-LXC setups

If you're running Docker inside the LXC (a common pattern for self-hosted app stacks), it's often easier to just standardize on one UID/GID for everything — set `PUID`/`PGID` environment variables to `1000:1000` on every container image that supports it (most linuxserver.io images do), create a matching user on the Proxmox host, and use the idmap trick above once. Then every bind mount, every app, every backup script all agree on the same numbers, and you stop having to think about it per-service.

## Checking your work

After adding idmap entries and restarting the container, confirm from inside the container:

```
stat -c '%u:%g' /path/to/bind/mount
ls -ln /path/to/bind/mount
```

and cross-check against `ls -ln` on the host side for the same directory. If the numeric UID/GID match on both sides, the mapping is correct — you shouldn't need `chown` gymnastics or loosened permissions to make the container's processes able to read and write the data.

This is a few minutes of config editing that saves you from either running privileged containers you don't need or leaving directories world-writable because it was easier than fixing the mapping. Once you've done it once for your standard UID, it's a copy-paste for every unprivileged container after that.
