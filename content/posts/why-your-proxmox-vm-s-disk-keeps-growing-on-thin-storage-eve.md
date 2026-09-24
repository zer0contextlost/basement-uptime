---
title: "Why Your Proxmox VM's Disk Keeps Growing on Thin Storage Even After You Delete Files Inside It"
date: 2026-09-24
description: "Deleting files inside a VM doesn't shrink its thin-provisioned image on the host. Here's why, and how to wire up discard/TRIM so it actually does."
tags: ["proxmox", "storage"]
---

You free up 40GB inside a VM, check the host, and the `.qcow2` or LVM-thin volume backing it hasn't moved an inch. This trips up almost everyone who moves from thick-provisioned disks to thin storage, because the mental model of "delete file, get space back" doesn't hold anymore once there's a virtualization layer in between.

## The disconnect between guest and host

A thin-provisioned disk only allocates blocks on the host as the guest writes to them. That part works as expected: install an OS, write 8GB of data, and the underlying image grows to roughly 8GB even though you told Proxmox to create a 100GB disk.

The problem is the reverse direction. When you delete a file inside the guest filesystem, the guest just marks those blocks as free in its own filesystem metadata. It doesn't tell the host "hey, blocks 40000 through 50000 are free now, you can reclaim them." As far as QEMU is concerned, those blocks were written to once, so they stay allocated in the image file forever, or at least until something explicitly punches a hole in them.

This is the same problem SSDs solved with TRIM, and the fix here is the same mechanism, just relayed through an extra layer.

## What has to line up

Three things need to be true simultaneously for reclaimed guest space to actually shrink the host-side image:

- The virtual disk needs `discard=on` set in its Proxmox config.
- The disk controller needs to support discard passthrough. VirtIO Block does not; VirtIO SCSI (or SCSI with the VirtIO SCSI controller) does.
- The guest OS needs to issue TRIM/discard commands, either continuously via a mount option or periodically via a scheduled job.

Miss any one of these and you get the growing-forever behavior. The most common mistake is having discard enabled on the disk but still using the old `virtio` bus type from a VM that was created years ago and never migrated to `virtio-scsi`.

Check the VM's hardware config in the Proxmox UI, or from the shell:

```
qm config 101 | grep -E 'scsi|virtio'
```

You want to see something like:

```
scsi0: local-lvm:vm-101-disk-0,discard=on,size=32G
scsihw: virtio-scsi-single
```

If the disk line shows `virtio0` instead of `scsiN`, discard isn't going to propagate no matter what you do inside the guest. You'll need to add a new SCSI disk, migrate the data over, and remove the old one, since you can't just flip the bus type on an existing attached disk.

## Getting the guest to actually ask for trim

Once the controller and discard flag are right, the guest still has to issue the commands. For Linux guests, the two options are:

- Mount with the `discard` option in `/etc/fstab`. This trims continuously as files are deleted, at the cost of a small write penalty on every delete.
- Enable `fstrim.timer`, which runs a batch trim on a schedule (weekly by default on most distros). This is the one I'd recommend for almost every case, since continuous discard on a virtual disk adds latency to operations that don't need it.

```
systemctl enable --now fstrim.timer
systemctl status fstrim.timer
```

You can also trigger it manually to test immediately after freeing up space, rather than waiting for the timer:

```
fstrim -v /
```

The `-v` flag prints how many bytes were trimmed, which is the fastest way to confirm the whole chain is actually working end to end.

For Windows guests, this is usually already handled: modern Windows runs a scheduled optimization pass that issues TRIM to any disk that reports itself as thin-provisioned, provided the VirtIO SCSI driver is installed and discard is enabled on the Proxmox side.

## Where the reclaimed space goes

Even after `fstrim` runs cleanly inside the guest, don't expect the storage backend to instantly reflect it, and don't expect it to reflect it the same way everywhere.

On LVM-thin, freed blocks return to the thin pool's free space, but the pool's `Data%` usage as reported by `lvs` can lag slightly depending on when the discard actually gets flushed through.

On ZFS-backed storage (zvols), trimmed blocks become free space in the pool, visible via `zpool list`, but the individual zvol's reported allocation can still look larger than expected because ZFS itself has its own block accounting quirks around sparse files and compression.

On plain directory storage using `qcow2` files, a successful trim can actually shrink the file's apparent size on disk (check with `du --apparent-size` vs `du` on the file), but only for blocks at the end of the file in some filesystem/qcow2 version combinations. A `qemu-img` sparsify pass after a big cleanup will do a more thorough job than relying on in-guest fstrim alone.

If none of this reclaims space and you're on LVM-thin, check that the pool itself wasn't created with an older `chunk_size` that predates discard support, since pools built years ago before this was default-on sometimes need to be recreated to get proper reclaim behavior.
