---
title: "Proxmox Ate All My RAM: Understanding ZFS ARC Before You Blame a VM"
date: 2026-08-10
description: "Why `free -h` on a ZFS-based Proxmox host looks alarming, and how to find and cap the ARC before you start killing VMs to 'free up memory.'"
tags: ["proxmox", "zfs", "memory"]
---

The first time I saw a Proxmox host sitting at 94% memory usage with three lightly-loaded VMs and a couple of idle LXCs, I assumed something had a leak. I went hunting through `htop`, sorted by memory, and found nothing that added up. The VMs were configured with modest RAM allocations, the containers were practically empty, and yet the host reported almost nothing free. Turns out nothing was wrong. ZFS was just doing exactly what it's designed to do.

## The ARC is not a leak

ZFS uses an Adaptive Replacement Cache, ARC, to cache recently and frequently read blocks in RAM. By default on most distros, ZFS will grow the ARC to use up to half of system memory, sometimes more depending on version and tuning defaults. It does this because unused RAM caching disk reads is strictly better than unused RAM sitting idle. The problem is that tools like `free` and `htop` often report this cached memory as "used" rather than "available," which makes a perfectly healthy host look like it's about to fall over.

If your Proxmox host has ZFS pools (rpool, a data pool, whatever), and you're seeing memory usage climb steadily after boot and plateau somewhere uncomfortably high, check the ARC before you start shutting down guests to "free up" RAM.

```bash
arc_summary | head -n 30
```

Look for the `ARC size` line versus `Target size`. If ARC size is sitting right at your installed RAM minus a small buffer, that's the cache doing its job, not a leak.

You can also watch it live:

```bash
watch -n1 arcstat
```

This shows hit rate, miss rate, and current ARC size in a rolling view. A high hit rate (90%+) on a host doing a lot of disk-backed reads means the cache is earning its keep.

## Why this actually matters

In most cases the ARC will shrink automatically under memory pressure, since it's designed to yield to applications that need RAM. But on a hypervisor, that "application" competing for memory is often a VM's balloon driver or a fresh container start, and the reclaim isn't always fast enough to avoid an OOM event if you're running the host close to the edge. I hit this on a box with 32GB of RAM, a handful of VMs with static allocations, and ZFS defaulting to using roughly half of that for ARC. Everything worked fine for weeks until I spun up one more VM during a migration and the host started killing processes instead of the ARC yielding room in time.

## Setting a hard ceiling

The fix is to cap ARC size explicitly rather than trust it to always negotiate cleanly with the rest of the system. On Linux ZFS this is controlled by `zfs_arc_max`, set in bytes.

Create or edit `/etc/modprobe.d/zfs.conf`:

```
options zfs zfs_arc_max=8589934592
```

That example caps ARC at 8GB. Pick a number based on what you actually need free for VMs, containers, and normal host overhead, not just whatever's left over. A common rule of thumb is capping ARC somewhere between 25-50% of total RAM on a hypervisor that's also running guest workloads, but the right number depends entirely on your pool size and read patterns.

After editing, rebuild initramfs so the parameter takes effect on boot:

```bash
update-initramfs -u
reboot
```

You can also set it live without a reboot for testing, though it won't persist:

```bash
echo 8589934592 > /sys/module/zfs/parameters/zfs_arc_max
```

Just be aware that setting it live only lets you shrink the current ARC size down to the new max, it won't retroactively fix a value that was already too generous until the cache naturally cycles.

## Don't overcorrect

It's tempting to cap ARC aggressively once you understand what's happening, but going too low hurts read performance for anything hitting the pool repeatedly, backup jobs, container image pulls, database-heavy LXCs. Watch `arcstat` hit rates after changing the limit. If your hit rate drops noticeably after capping, you took away cache that was actually being used, not just idle padding.

Check this before you scale up guest RAM allocations on a ZFS host, not after you've already spent an afternoon debugging phantom OOM kills.
