---
title: "Proxmox LXC Containers Running Out of Inodes While df Shows Plenty of Free Space"
date: 2026-10-29
description: "A container throws 'No space left on device' with gigabytes free. Here is how to confirm inode exhaustion, find the culprit directory, and fix it."
tags: ["proxmox", "lxc", "storage", "troubleshooting"]
---

A container stops writing logs. Apt fails halfway through an upgrade. A database refuses to create a temp file. Every error says `No space left on device`, so you run `df -h` and see 40% used. Gigabytes free. Nothing makes sense.

Check inodes before you start resizing disks.

## Confirm it with df -i

```bash
df -i /
```

The output has the same layout as plain `df`, but the columns count inodes instead of bytes:

```
Filesystem      Inodes  IUsed  IFree IUse% Mounted on
/dev/loop3      524288 524288      0  100% /
```

If `IUse%` is at 100%, that's your answer. Every file, directory and symlink uses one inode, and a filesystem formatted with ext4 gets a fixed number of them at creation time. Run out and you can't create anything new, no matter how many free blocks remain.

For a container on LVM-thin or a directory storage, the rootfs is usually ext4, so this applies. On ZFS-backed subvolumes the story differs, since ZFS allocates dnodes dynamically and you will hit a quota or pool limit first.

## Why containers hit this more than VMs

Two reasons, both boring.

First, Proxmox sizes the ext4 filesystem for a container using the default ratio of one inode per 16 KiB of space, as mkfs.ext4 does. An 8 GB rootfs gets roughly half a million inodes. That's fine for a typical service. It is not fine for something that creates millions of tiny files.

Second, the workloads people put in containers are exactly the ones that do this:

- Mail servers using Maildir, one file per message
- PHP session directories that never get cleaned
- Cache directories from image proxies or thumbnailers
- A `node_modules` tree, or several
- Monitoring agents dumping one small file per check
- Docker overlay layers inside a nested container, which multiply file counts quickly

## Find the directory eating them

Disk usage tools count bytes. You want to count files. This walks one level at a time without crossing filesystem boundaries:

```bash
for d in /*; do
  [ -d "$d" ] && printf '%s %s\n' "$(find "$d" -xdev 2>/dev/null | wc -l)" "$d"
done | sort -rn | head
```

Take the worst offender and repeat inside it. Two or three rounds usually lands on the culprit. Common finds are `/var/lib/php/sessions`, `/var/spool`, `/var/cache`, and an application's own `tmp` folder.

If `find` itself is slow because the filesystem is thrashing, run it from the Proxmox host against the mounted rootfs instead. `pct mount <vmid>` mounts it under `/var/lib/lxc/<vmid>/rootfs`, and you can run the same loop there. Run `pct unmount <vmid>` when you're done.

## Free some inodes right now

Deleting a few huge directories is faster with `find ... -delete` than with `rm -rf` on some trees, and it avoids building a giant argument list:

```bash
find /var/lib/php/sessions -type f -mtime +7 -delete
```

Be careful about what you delete. Session files and caches are usually safe. Anything under a database directory is not. If you don't know what a directory is for, leave it alone and look it up first.

Once the count drops, the container recovers immediately. No restart needed in most cases.

## Fix it for good

Cleaning up buys time. The permanent fix depends on the cause.

**The files are junk that should expire.** Add a cron job or systemd timer that prunes them. PHP distributions ship a session cleaner, but it is sometimes disabled in containers because it depends on a systemd unit that isn't present. Check that something is running it.

**The files are legitimate and there are just a lot of them.** You have to rebuild the filesystem with a better inode ratio, because ext4 can't add inodes in place. Growing the disk with `pct resize` does add inodes proportionally, since the resize extends the filesystem, but the ratio stays the same. Doubling the disk doubles the inode count. That's a blunt but valid fix if you have the space.

For a finer fix, create a new container, mount its rootfs from the host, and copy the data over. Or move the heavy directory to a bind-mounted mount point on a dataset or filesystem formatted with a smaller bytes-per-inode value:

```bash
mkfs.ext4 -i 4096 /dev/mapper/vg-bigfiles
```

The `-i 4096` setting gives one inode per 4 KiB, four times the default density. The cost is a little more metadata overhead, which on a filesystem full of small files is a fair trade. XFS is another option, because it allocates inodes dynamically and rarely hits this limit.

## Add an alert so it doesn't surprise you again

Most monitoring setups watch disk bytes and forget inodes. If you run node_exporter, the filesystem collector already exposes `node_filesystem_files` and `node_filesystem_files_free`. An alert on free inodes below 10% takes one rule. Without Prometheus, a daily cron that runs `df -i` and mails you when any line exceeds 90% works fine.

Put that check on the container, not just the host. The host's own filesystems look healthy while the guest rootfs sits at 100%.

I now run `df -i` as one of the first three commands in any "no space left" ticket I open against myself. It has been the answer more often than I'd like to admit.
