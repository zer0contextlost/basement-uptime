---
title: "The NFS Mount That Hangs Your Whole Proxmox Host When the NAS Blinks"
date: 2026-10-05
description: "Why a single NFS mount with default options can freeze pct commands and container management across your whole Proxmox node when the NAS drops offline."
tags: ["nfs", "proxmox", "lxc", "storage"]
---

## The symptom

Your NAS reboots for a firmware update, or a switch port flaps, or you just unplug the wrong cable for ten seconds. Normally that's a non-event. Instead, your Proxmox host locks up. `pct enter` on an unrelated container hangs. `pct list` hangs. Even `ls` on an unrelated path that has nothing to do with the NAS stalls for a while before coming back, if it comes back at all. The web UI shows the node as unreachable even though the host is clearly still up and pingable.

The cause is almost always an NFS mount set up with the default options, used to feed media, backups, or shared storage into one or more LXC containers.

## Why NFS hangs instead of failing

NFS mounts default to `hard`. A hard mount means the kernel NFS client will retry a request forever if the server stops responding, rather than giving up and returning an error to whatever process asked for the file. Any process that touches a file or even just `stat`s a path under that mount gets parked in an uninterruptible sleep until the server answers again.

That's a reasonable default for an NFS server you fully trust and expect to recover. The problem is the blocking. A process in uninterruptible sleep waiting on NFS can't be killed with a normal signal, and sometimes not even with `SIGKILL`, until the mount either reconnects or the kernel's retry logic gives up. If that stuck process happens to be something `pvestatd` or the container management stack touches while enumerating mounts or checking container status, the stall propagates outward to tools that have nothing to do with the NAS.

## Why it's worse with LXC specifically

Unprivileged LXC containers generally can't mount NFS shares themselves, since that needs capabilities the container doesn't have in its restricted namespace. So the common pattern is to mount the NFS share on the Proxmox host and bind it into the container with `lxc.mount.entry` or a matching line in the container config.

That means the mount lives at the host level, shared infrastructure for every container that references it. One flaky NAS connection doesn't just hang the container using it. It hangs the host path that container's mount is rooted in, and anything else on the host that walks that same directory tree, including backup jobs and monitoring checks that happen to stat storage paths in a loop.

## The options that actually matter

- **`soft` vs `hard`** — `soft` gives up after a configured number of retries and returns an I/O error to the calling process instead of blocking forever. It avoids the hang, but it comes with a real risk: if a write was in flight when the timeout hit, the app has no reliable way to know whether that write landed. For read-mostly mounts (media libraries, static file shares) `soft` is usually the better trade. For anything with a database or appendonly file living on it, don't.
- **`timeo` and `retrans`** — these control how long the client waits and how many times it retries before giving up (on `soft`) or before each retry cycle repeats (on `hard`). The out-of-the-box values are tuned for links with real latency, not a NAS two feet away on the same switch. Tightening them means a blip resolves in seconds instead of minutes.
- **`bg`** — if the mount is in `/etc/fstab` and the NAS isn't up yet at boot, `bg` backgrounds the mount attempt instead of blocking the boot sequence until it succeeds or times out.
- **`_netdev`** — tells the system this is a network filesystem so it isn't attempted before networking is actually up, which matters more on a host that boots the NAS and the hypervisor from the same power event.

A reasonable middle ground for a homelab media mount looks something like `soft,timeo=100,retrans=3,_netdev` rather than the bare defaults.

## Diagnosing one after the fact

If you're already stuck, check `ps aux` for processes in state `D`. `cat /proc/<pid>/stack` on one of them will usually show it parked somewhere in the NFS client waiting on a request. A lazy unmount, `umount -l`, can sometimes free up new access to the path without killing the stuck processes, but it won't unstick the ones already blocked. Those clear only when the server comes back or the kernel's retry window finally lapses. In the worst cases, a full host reboot is the fastest way out, which is itself a strong argument for not using bare defaults in the first place.

Before trusting any NFS mount feeding an LXC container, pull the NAS's network cable for thirty seconds while nothing else is happening and watch what the host does. That test tells you more about your mount options than reading the man page ever will.
