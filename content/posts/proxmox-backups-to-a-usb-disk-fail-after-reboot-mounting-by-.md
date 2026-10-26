---
title: "Proxmox Backups to a USB Disk Fail After Reboot: Mounting by UUID and Guarding the Mountpoint"
date: 2026-10-26
description: "A USB backup disk that comes back as a different device name, or doesn't mount at all, can quietly fill your root filesystem. Here is how to prevent it."
tags: ["proxmox", "backups", "usb", "fstab", "storage"]
---

A cheap and common backup setup: an external USB drive plugged into the Proxmox host, added as a directory storage, and vzdump pointed at it every night. It works for months. Then one day the root filesystem is 100% full, the web UI is crawling, and the "USB backup" turns out to be a pile of archives written to the host's own system disk.

This is a mount problem, and it has a clean fix.

## What actually happens

A directory storage in Proxmox is just a path, say `/mnt/usbbackup`. Proxmox does not check that something is mounted there. If the disk is absent, that path is an ordinary empty folder on the root filesystem, and vzdump will happily write into it.

Two things make the disk go missing:

- The drive enumerates under a different name after a reboot or replug. `/dev/sdb1` becomes `/dev/sdc1` because something else grabbed the earlier letter first.
- The drive is slow to spin up, and the mount attempt at boot happens before the kernel has seen it.

If your fstab line says `/dev/sdb1`, either case leaves you with an unmounted directory, or worse, the wrong partition mounted over it.

## Step 1: Mount by UUID

Never reference USB disks by device letter. Get the filesystem UUID:

```
blkid /dev/sdb1
```

Then use it in `/etc/fstab`:

```
UUID=<your-uuid-here>  /mnt/usbbackup  ext4  defaults,nofail,x-systemd.device-timeout=30  0  2
```

Two options matter here. `nofail` lets the host boot even if the disk is absent. Without it, a missing drive can drop you into emergency mode on a headless box, which is a miserable way to find out. `x-systemd.device-timeout=30` caps how long boot waits for the device instead of the default, which can be long enough to look like a hang.

After editing, run `systemctl daemon-reload` and `mount -a` to confirm the line parses. A typo in fstab is much cheaper to find now than at the next reboot.

## Step 2: Make the empty mountpoint unwritable

This is the trick that stops the root-disk-fill failure entirely. With the disk unmounted, set the immutable flag on the bare directory:

```
umount /mnt/usbbackup
chattr +i /mnt/usbbackup
mount -a
```

The immutable attribute belongs to the directory on the root filesystem underneath. Once the USB disk is mounted on top, you see the disk's own directory, which is writable as normal. When the disk is missing, the underlying directory is immutable, and vzdump fails with a permission error instead of silently filling `/`.

A failed backup that shows up red in the task log is a far better outcome than a successful one that ate your boot disk.

## Step 3: Tell Proxmox the storage needs a mount

Proxmox has a flag for exactly this. On the directory storage, set `is_mountpoint`:

```
pvesm set usbbackup --is_mountpoint yes
```

With it enabled, Proxmox checks that the path is a real mountpoint before using it and treats the storage as offline otherwise. You can also set it in `/etc/pve/storage.cfg` as `is_mountpoint 1` under the storage definition. Combined with the immutable flag you have two independent guards, which is reasonable for something that fails this quietly.

## Step 4: Check the storage state, don't assume

After a reboot, look at it before the first scheduled job:

```
findmnt /mnt/usbbackup
pvesm status
```

`findmnt` printing nothing means nothing is mounted. In `pvesm status`, an offline directory storage shows as inactive. If you run monitoring, alert on the mount being absent rather than on disk usage, since by the time root usage trips an alarm the damage is done.

A simple cron or systemd timer check works fine:

```
mountpoint -q /mnt/usbbackup || echo "usbbackup not mounted" | logger -t backupcheck
```

Wire that into whatever already pages you.

## Cleaning up after the fact

If you suspect it has already happened, unmount the disk and look at the bare directory:

```
umount /mnt/usbbackup
du -sh /mnt/usbbackup
ls -la /mnt/usbbackup
```

Archives sitting there are the ones that landed on the root filesystem. Delete them, then apply the immutable flag from step 2 before remounting. Remember that anything mounted over a non-empty directory hides its contents but doesn't free the space, so `df` on `/` will stay high until you remove them.

## A note on USB disks themselves

USB enclosures sometimes go to sleep or drop off the bus under load, and a disk that disappears mid-backup leaves a truncated archive. Mounting correctly will not solve that. Check the job logs for I/O errors now and then, and test a restore periodically. A USB drive is a fine second copy, but it should not be the only one.
