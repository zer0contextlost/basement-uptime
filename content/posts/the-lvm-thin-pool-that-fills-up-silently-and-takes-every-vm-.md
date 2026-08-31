---
title: "The LVM-Thin Pool That Fills Up Silently and Takes Every VM With It"
date: 2026-08-31
description: "How Proxmox's default LVM-thin storage overcommits disk space, why fstrim matters more than you think, and how to stop a full pool from corrupting VMs."
tags: ["proxmox", "storage", "lvm"]
---

Proxmox's default local storage on a lot of installs is LVM-thin. It looks like a normal volume group until you check the numbers: you can create four 200GB VM disks on a 500GB pool and Proxmox will let you, no warning. That's thin provisioning working as designed. The problem is what happens when the pool actually fills up, because unlike a full ext4 partition, a full LVM-thin pool doesn't fail gracefully.

## Why the pool fills up faster than you expect

Thin provisioning only allocates blocks as the guest writes to them. A 200GB virtual disk with 20GB of actual data inside only consumes about 20GB from the pool. That's the whole appeal.

The catch is that guests rarely give blocks back. Delete a 10GB file inside a VM and the filesystem inside the guest marks those blocks free, but the hypervisor's thin pool has no idea. As far as LVM is concerned, those blocks are still allocated. Over months of installing packages, rotating logs, and deleting old ISOs inside guests, the pool's actual usage creeps toward 100% even though the guests report plenty of free space internally.

Check real usage with:

```
lvs -a -o+lv_size,data_percent,metadata_percent
```

`data_percent` is the number that matters. Proxmox's summary view can lag behind this, so don't trust the GUI's storage graph as your only signal.

## What happens when it actually fills

This is the part that catches people off guard. When an LVM-thin pool runs out of space, it doesn't reject new writes cleanly. Whichever VM happens to write next gets an I/O error from the block device, and depending on the guest filesystem and what was mid-write, that can mean anything from a service crash to a corrupted filesystem that needs an fsck or a restore from backup. It's not one VM that gets punished either. Every guest sharing that pool is at risk, because the pool doesn't know or care which VM's write pushed it over.

There's also a metadata volume, separate from the data volume, that can fill up independently if you have a lot of snapshots or small extents. `metadata_percent` maxing out causes the same kind of failure and is easy to miss because people only watch the data number.

## Getting space back: discard and fstrim

The fix for the slow creep is making sure guests actually tell the hypervisor when blocks are free. Two things have to line up:

- The VM's disk needs `discard=on` set in its Proxmox hardware config (works with VirtIO SCSI or VirtIO Block controllers).
- The guest OS needs to actually issue TRIM/discard commands, either via a periodic `fstrim` cron/timer or a filesystem mounted with `discard`.

Most modern Linux distros ship an `fstrim.timer` that runs weekly, but it's worth confirming it's enabled rather than assuming:

```
systemctl status fstrim.timer
```

For Windows guests, discard happens through periodic Optimize-Drives runs, which needs the disk to report as non-rotational and support TRIM, both of which depend on `discard=on` being set on the VirtIO controller.

Without this loop in place, thin provisioning is a one-way ratchet. Space gets allocated and never returned until you manually run `fstrim` from inside the guest or move the disk.

## Guardrails worth setting up

Don't rely on remembering to check `lvs` periodically. A few things are worth doing once and forgetting:

- Set a monitoring check (Uptime Kuma, a simple cron script piping to a webhook, whatever you already have) that alerts on `data_percent` crossing something like 80%.
- Leave real headroom in the pool instead of provisioning it to the edge. Overcommitting virtual disk sizes is fine; overcommitting to the point where the physical pool itself is nearly full is not.
- If you're already running ZFS elsewhere on the same host, consider whether you actually need LVM-thin at all. ZFS handles thin provisioning and space accounting more transparently, and `zpool list` won't lie to you the way a stale Proxmox storage summary can.
- Snapshots eat into the same pool and don't get trimmed by guest-level fstrim. Old VM snapshots sitting around are often the real reason a pool that "should" have space doesn't.

A full thin pool is one of the few Proxmox failure modes that can take down several unrelated VMs at once over what looks like unrelated disk pressure in just one of them. Checking `data_percent` takes ten seconds. Finding out the hard way costs a restore.
