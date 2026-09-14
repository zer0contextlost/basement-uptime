---
title: "pmxcfs and the Backup Blind Spot Hiding in /etc/pve"
date: 2026-09-14
description: "Proxmox stores its config as a SQLite database wearing a filesystem costume, and vzdump doesn't touch it. Here's what that means for disaster recovery."
tags: ["proxmox", "pmxcfs", "backups"]
---

Try this on any Proxmox host, clustered or not: `ln /etc/pve/qemu-server/100.conf /etc/pve/qemu-server/100-backup.conf`. It fails. Not because of permissions, because `/etc/pve` doesn't support hard links at all. Try a device node, a named pipe, or a file north of a few megabytes and you'll hit the same wall. That directory looks like ext4 or XFS when you `ls` it, but it's not a filesystem in the normal sense. It's a SQLite database wearing a filesystem costume.

## What's actually mounted there

`/etc/pve` is a FUSE mount served by a process called `pmxcfs`. Everything under it, VM configs, container configs, storage.cfg, cluster membership, the works, lives in a SQLite database at `/var/lib/pve-cluster/config.db`. The FUSE layer translates normal file reads and writes into database rows, and on a clustered setup, corosync replicates every write to every other node in near real time. That's the whole trick behind editing a VM config on one node and seeing it appear instantly on another: you're not looking at two copies of a file, you're looking at two views of the same replicated record.

This runs even on a single standalone node with no cluster configured. There's no cluster to sync to, but pmxcfs is still the thing serving `/etc/pve`, still backed by that same SQLite file, still subject to the same constraints.

## Where this bites you

**Backup jobs that assume `/etc/pve` is covered.** vzdump backs up VM and container disks and their config snapshots at backup time, but it does not back up the pmxcfs database itself. If you're rsyncing `/etc` as part of a general host backup strategy expecting to capture Proxmox's cluster config, you're capturing a FUSE mount, not the underlying data. Restoring from that rsync gets you nothing useful, because there's no `pmxcfs` process on the receiving end to reconstitute it into.

**Tools that write via rename.** A common atomic-write pattern is: write to a temp file, then rename it over the target. Plenty of editors and config-management tools do this by default. pmxcfs handles renames within `/etc/pve` inconsistently for some tools depending on how they invoke it, and scripts built around that pattern can fail silently or throw errors that don't make sense until you remember you're not on a real filesystem.

**Quorum loss makes it read-only.** If a clustered node can't see a majority of the cluster, pmxcfs on that node drops to read-only. You'll see this most often as "unable to write file" errors when you try to create a container, edit a VM's hardware, or do anything else that touches `/etc/pve`, even though the node itself is up and VMs already running on it keep running fine. The filesystem-looking thing under `/etc/pve` is really a proxy for cluster consensus, and it stops accepting writes the moment consensus is in doubt.

## What to actually back up

The thing worth protecting is `/var/lib/pve-cluster/config.db`. pve-cluster keeps its own rolling set of timestamped SQL dump backups in `/var/lib/pve-cluster/` automatically, which is worth knowing about before you go build your own cron job to do the same thing. For your own backup strategy, a filesystem-level snapshot or a straight copy of `/var/lib/pve-cluster/config.db` (with the service stopped, or via a tool that handles SQLite consistently during writes) is what actually lets you rebuild `/etc/pve` from scratch.

For disaster recovery on a fresh install, the general shape is: stop `pve-cluster`, restore `config.db` into place, start `pve-cluster` again, and let pmxcfs rebuild the FUSE view from the database. The exact steps depend on whether you're restoring into a standalone node or rejoining a cluster, but the database file is the artifact that matters, not anything you'd find by looking inside `/etc/pve` with `find` or `tar`.

## A quick sanity check

Run `mount | grep pve` on any Proxmox host and you'll see the FUSE entry spelled out:

```
/dev/fuse on /etc/pve type fuse.pve (rw,nosuid,nodev,...)
```

That line is worth remembering the next time a backup script silently produces an empty archive for `/etc/pve`, or a config edit throws a permission error on a node that looks perfectly healthy otherwise. The directory isn't lying to you about being different. It's just not advertising it.
