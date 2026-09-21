---
title: "Proxmox Backup Server: Why Pruning Old Backups Doesn't Free Any Disk Space"
date: 2026-09-21
description: "Prune jobs on Proxmox Backup Server remove backup references, not the underlying data. Here's why your datastore keeps filling up and how garbage collection actually works."
tags: ["proxmox", "backups", "pbs"]
---

I had a Proxmox Backup Server datastore sitting at 94% full with a prune job running every night and a retention policy that should have kept maybe two weeks of backups per guest. The prune logs showed dozens of old snapshots getting removed on schedule. The disk usage barely moved. Turns out prune was doing exactly what it was supposed to, and the disk was never going to shrink because of it.

## Prune deletes references, not data

PBS stores backup data as content-addressable chunks. When a VM or container backs up, the client splits the data into chunks, hashes each one, and only uploads chunks the server doesn't already have. A backup "snapshot" in the PBS web UI is really just an index file listing which chunks make up that point-in-time backup. Most of those chunks are shared across dozens of snapshots because most of a disk doesn't change day to day.

When prune runs, it deletes the index files for backups that fall outside your retention rules. It does not touch the chunk store. A chunk that's referenced by nine other snapshots obviously can't be deleted just because one snapshot pointing to it got pruned, and PBS has no cheap way to know, at prune time, whether a given chunk is now orphaned. So it doesn't try. Prune's job ends at removing the index.

## Garbage collection is the part that actually frees space

Reclaiming space is a separate job: garbage collection. GC walks every remaining index file in the datastore, builds a list of every chunk still referenced by something, then walks the chunk store itself and deletes anything not on that list. This is a full scan of the datastore, and on spinning disks with a few million chunks it can take hours.

If you never schedule GC, or you disabled it thinking prune already covers cleanup, your chunk store only grows. Datastore → Prune & GC in the UI lets you set a schedule for each independently, and it's easy to configure retention without ever touching the GC schedule because the UI doesn't force you to.

## The atime trap that makes GC unsafe

GC has a built-in safety window. Because a backup job can be uploading new chunks at the same moment GC is scanning, PBS uses each chunk's access time to decide whether it's safe to delete. Chunks touched more recently than a cutoff (currently a day and some change) get skipped even if no index references them, in case they belong to a backup that's mid-upload and hasn't been indexed yet.

This only works if the underlying filesystem actually updates atime on read. If your datastore lives on a filesystem mounted with `noatime`, or on certain network shares, or under some ZFS configurations with atime disabled at the dataset level, GC's safety check silently degrades. In the worst case it either refuses to remove chunks it should (because atime never reflects real access) or, less commonly depending on how you've tuned it, it becomes less safe about deletions during concurrent backup jobs.

Check your datastore's mount options:

```
mount | grep pbs-datastore
```

and the ZFS property if it's on a ZFS dataset:

```
zfs get atime rpool/pbs-datastore
```

If atime is off, either turn it back on for that dataset or use `relatime`, which updates atime often enough for GC's purposes while avoiding the write overhead of full atime tracking.

## What to actually check

- Confirm GC is scheduled, not just prune. They're separate jobs with separate schedules in Datastore → Prune & GC.
- Run GC manually once (`Prune & GC → Show GC Log` or via the datastore's GC button) and watch whether it reports removed chunks and reclaimed bytes at the end.
- If GC runs but reclaims almost nothing on a datastore where you know old data was pruned, check atime on the underlying filesystem before assuming something else is wrong.
- Give GC its own maintenance window. Running it back to back with backup jobs on the same datastore slows both down, and on a large chunk store it can push into your backup window.

None of this shows up as an error anywhere. Prune reports success, backups report success, and the datastore just keeps climbing until you're staring at a disk full warning with no obvious explanation in the logs.
