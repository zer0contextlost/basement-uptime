---
title: "Proxmox Snapshots Pile Up and Quietly Slow Down Your Guests"
date: 2026-11-05
description: "Old snapshots on LVM-thin, qcow2 and ZFS cost you space and speed. How to find them, what they cost, and a routine for keeping them short-lived."
tags: ["proxmox", "snapshots", "storage", "maintenance"]
---

A snapshot takes two seconds to create and feels free. That is the problem. Nobody schedules a reminder to delete it, so six months later you have a VM with a chain of "before upgrade" snapshots, a disk that has grown for no obvious reason, and a rollback point so old that using it would lose half a year of data.

## What a snapshot costs depends on the storage

The cost is not the same everywhere, so check what your guest actually sits on.

- **ZFS (zvol or dataset):** A snapshot is nearly free at creation. Space is held only for blocks that have changed since. Reads stay fast. The cost is space that cannot be reclaimed until the snapshot is gone, which looks like "I deleted 200 GB and nothing came back."
- **LVM-thin:** Similar copy-on-write behavior at the block level. Every first write to a block after the snapshot may trigger a copy, and the pool's data usage climbs. If you are already watching a thin pool, snapshots are one of the quiet ways it fills.
- **qcow2 on a directory or NFS storage:** Each snapshot adds an internal layer. The more layers, the more metadata lookups per read, and the file keeps growing. Long chains on qcow2 are where I have seen the most noticeable slowdown.
- **LVM (thick) and raw on plain directories:** Snapshots are not supported through the Proxmox UI in the usual way. If the option is greyed out, that is why.

None of these will warn you. Proxmox shows the snapshot list in the guest's Snapshots tab and that is about it.

## Finding every snapshot on the node

Clicking through each guest does not scale. From the shell:

```bash
for id in $(qm list | awk 'NR>1 {print $1}'); do
  echo "== VM $id"; qm listsnapshot $id | grep -v current
done

for id in $(pct list | awk 'NR>1 {print $1}'); do
  echo "== CT $id"; pct listsnapshot $id | grep -v current
done
```

Anything printed besides the header is a snapshot. The listing includes a creation date, which is the number you care about.

On ZFS you can see the space they hold directly:

```bash
zfs list -t snapshot -o name,used,creation -s creation
```

The `used` column is the space that would be freed if that one snapshot were destroyed. A snapshot showing a few hundred megabytes is harmless. One showing tens of gigabytes on a busy database VM is costing you real capacity.

## The rollback you will not want to use

Age matters beyond space. Rolling back to a three-month-old snapshot discards everything written since: database rows, container configs, new files. A snapshot taken before a risky upgrade is valuable for a day or two. After you have confirmed the upgrade worked, it is dead weight that only looks like a safety net.

A snapshot is also not a backup. It lives on the same storage as the guest. If the pool dies, both go together. Keep your vzdump or PBS jobs for that.

## Deleting a snapshot is not always instant

Removing a snapshot on qcow2 merges its layer into the next one, and that can take a while on a large disk and will hit I/O. Do it off-hours. On ZFS the destroy is quick, though freeing a lot of space can briefly load the pool.

```bash
qm delsnapshot 101 before-upgrade
pct delsnapshot 202 before-upgrade
```

If a delete fails and leaves the guest locked, the lock-clearing steps in the earlier locked-containers post apply. Check that no backup is running first.

Remember that deleting a middle snapshot in a chain is fine. The data is merged forward, not lost. Only rollback is destructive.

## A habit that works

I use a naming rule: every manual snapshot gets the date and a reason, such as `2026-11-05-pre-kernel`. The Snapshots tab then shows its own age at a glance, and you do not need to open the description field.

Then a monthly check. A short script that lists snapshots older than a threshold is enough:

```bash
#!/bin/bash
# prints guests with snapshots older than 14 days
cutoff=$(date -d '14 days ago' +%s)
for id in $(qm list | awk 'NR>1 {print $1}'); do
  qm listsnapshot $id | grep -v current | while read -r line; do
    echo "VM $id: $line"
  done
done
```

Extend it to parse the date column and compare with `$cutoff`, then send the output to whatever you already use for alerts. Mine lands in the same place as my backup failure notices, so it gets read.

## When a long-lived snapshot is fine

There are cases where I keep one deliberately: a lab VM I break and restore constantly, or a pristine template-like state for testing installers. For those, the snapshot is the workflow. Put a note in the name, like `golden-base`, so future you knows it is intentional, and exclude it from the cleanup check.

For everything else, the rule is simple. Take the snapshot, do the change, verify, delete it the same week. If you cannot decide whether the change worked, that is a sign to wait a few more days, not to leave it for a year.
