---
title: "Proxmox Backup Jobs Silently Skip Guests When You Use Pool or Exclude Selection"
date: 2026-10-19
description: "How vzdump selection modes (all, include, pool, exclude) behave when you add new guests, and how to audit which VMs your backup job really covers."
tags: ["proxmox", "backups", "vzdump", "homelab"]
---

A backup job can run green every night and still never touch the VM you created last month. I found this out the unpleasant way, during a restore I had assumed would be easy. The job was fine. The selection was the problem.

## The four ways a job picks guests

In the Proxmox GUI, a backup job under Datacenter > Backup has a "Selection mode" dropdown. It matters more than it looks.

- **Include selected VMs**: a fixed list of VMIDs. New guests are never added.
- **All**: every guest on the node (or the whole cluster, depending on scope). New guests are picked up automatically.
- **Exclude selected VMs**: everything except a list. New guests are picked up.
- **Pool based**: every guest that is a member of a given resource pool.

"Include selected" is the one that bites. You create a new container, test it for a week, start relying on it, and the job keeps happily backing up the same six guests it knew about when you wrote it. No error. No warning. The task log says OK because, from the job's point of view, it did exactly what it was told.

Pool mode has the same failure with a different trigger. A guest only gets backed up if someone remembered to assign it to the pool at creation time. The create wizard has a pool field, and it is easy to leave blank.

## Node scope in a cluster

There's a second gap in clusters. A job can be limited to a single node. If you migrate a guest to a different node, and the job is scoped to the original one, the guest drops out of coverage. With "All" mode and node scope set, you are backing up everything that currently lives on that node, which is not the same as everything in the cluster.

If you have one job per node, check that each node has one. Adding a third node and forgetting its job is a classic.

## Checking what a job actually covers

Don't trust the job's description in the GUI. Ask the cluster.

```bash
pvesh get /cluster/backup-info/not-backed-up
```

That endpoint lists guests that no enabled backup job includes. On a healthy setup it returns an empty list. In the GUI, the same check appears as a warning banner when you open the Backup panel, but it's easy to dismiss and never look at again.

To see what a specific job resolves to:

```bash
pvesh get /cluster/backup
cat /etc/pve/jobs.cfg
```

`jobs.cfg` shows the `vmid`, `pool`, `all`, or `exclude` fields for each job. Reading it takes thirty seconds and shows exactly which mode you are in.

Then compare against what exists:

```bash
qm list
pct list
```

Or check from the storage side, which is the check that matters. Look at your backup storage and confirm each guest has a recent archive:

```bash
pvesm list backup-store --content backup | awk '{print $1}' | sort
```

Replace `backup-store` with your storage ID. Each volume ID contains the VMID and a timestamp. A VMID missing from this list has no backups, whatever the job says.

## Which mode to pick

For a homelab, I use "All" with an explicit exclude list. The default becomes protection. If I spin up a scratch VM and forget about it, it gets backed up, and the cost is a few gigabytes of dedup'd storage. The opposite default, silently unprotected, costs a lot more when it goes wrong.

Excludes should be deliberate and written down. Typical candidates:

- Throwaway test guests you rebuild from scripts anyway.
- Guests whose data lives on a mounted share that is backed up separately.
- A big media VM where a full image backup would blow up your retention budget.

Note that bind mounts and passthrough disks are not included in container or VM backups unless you set the backup flag on that mount point or disk. That is a related blind spot. A guest can be in the job and still have its important data skipped. For a mount point, look for `backup=1` in the config line, and for a VM disk, make sure "Backup" is not unchecked.

```bash
grep -E 'mp[0-9]+|scsi[0-9]+|virtio[0-9]+' /etc/pve/lxc/101.conf
```

## Alerting on the gap

I put the `not-backed-up` check into a small cron script that sends a notification if the list is non-empty. Something like:

```bash
#!/bin/sh
out=$(pvesh get /cluster/backup-info/not-backed-up --output-format json)
[ "$out" != "[]" ] && echo "$out" | mail -s "Unprotected guests" root
```

Run it daily on one node. Cheap, and it turns a silent gap into a message in your inbox the morning after you create the guest.

Next time you build a new container, open the Backup panel before you close the browser tab. If the banner is there, fix it now, while you still remember why the guest exists.
