---
title: "Proxmox Containers Stuck in 'Locked' State: Clearing a Stale Backup or Migration Lock Safely"
date: 2026-10-12
description: "A VM or container shows a lock and refuses to start, stop, or back up. How to find out why the lock exists, and when it is safe to run qm unlock or pct unlock."
tags: ["proxmox", "lxc", "troubleshooting", "backups"]
---

You try to start a container and get this:

```
CT is locked (backup)
```

Or a VM refuses to stop with `VM is locked (snapshot)`. The guest shows a little padlock icon in the web UI, and every action that touches its config fails until the lock goes away.

The fix is one command. The part that matters is knowing whether you should run it yet.

## What the lock is

Proxmox writes a `lock:` line into the guest's config file when a long operation starts. You can see it directly:

```
cat /etc/pve/qemu-server/100.conf
cat /etc/pve/lxc/101.conf
```

Somewhere near the top you will find something like `lock: backup`. The value tells you which operation set it. Common ones:

- `backup`: a vzdump job is running or died mid-run
- `snapshot`: a snapshot is being created
- `rollback`: a snapshot rollback is in progress
- `migrate`: the guest is being moved to another node
- `create`: the guest is still being built or restored
- `disk`: a disk move or resize is underway
- `suspending` / `suspended`: hibernation state

The lock exists so two operations cannot rewrite the same config at once. It is a safety feature, and a stale one is a leftover from a process that never got to clean up.

## Check whether the operation is still running

Never unlock first. Look for the process that owns it.

```
ps aux | grep -E 'vzdump|qmrestore|pct|qm ' | grep -v grep
cat /var/log/pve/tasks/active
```

The task list at `/var/log/pve/tasks/active` shows running tasks. In the web UI, the task log at the bottom of the screen shows the same thing, and a running job will have a spinner and no end status.

If a backup is mid-flight, wait. Unlocking a guest while vzdump is still reading its disk can leave you with a backup that finishes against a config that has changed underneath it, or a snapshot that is never removed.

A migration deserves even more caution. If the lock is `migrate`, check both nodes. The guest may be partly copied to the target. Find out where it is actually running before you touch anything:

```
qm list
pct list
```

Run those on both nodes. A guest listed as running on the target while the source still holds a lock means the migration finished and cleanup was interrupted.

## Why locks go stale

The usual causes are mundane:

- The host rebooted or lost power during a backup.
- The backup target (NFS, PBS) went away and vzdump was killed.
- Someone cancelled a task from the shell with `kill -9`.
- A snapshot operation timed out waiting on the guest agent.

That last one connects to the fs-freeze problem described in an earlier post. A guest that won't freeze can hang vzdump long enough that you kill the job, and the lock stays behind.

## Clearing it

Once you are sure nothing is running:

```
qm unlock 100
pct unlock 101
```

Both commands only remove the `lock:` line. They do not repair whatever the interrupted operation left behind, and that is the part people skip.

## Cleaning up after each lock type

**After a backup lock.** A VM backed up in snapshot mode may still have a leftover backup state. For a VM, check that it is not stuck paused or with a dangling freeze:

```
qm status 100 --verbose
qm agent 100 fsfreeze-status
```

If the status says `frozen`, thaw it with `qm agent 100 fsfreeze-thaw`. A guest left frozen looks hung, with a console that accepts nothing and services that stop answering.

For a container, a failed snapshot-mode backup can leave a mounted snapshot or an ordinary LVM or ZFS snapshot behind. Look at `lvs` or `zfs list -t snapshot` for names you did not create.

**After a snapshot lock.** Run `qm listsnapshot 100`. A half-created snapshot is often listed with a `snapshot` state in the config file under a `[section]`. If it is broken, `qm delsnapshot 100 name --force` removes the entry from the config even when the storage side fails. Then check the storage yourself for orphaned volumes.

**After a rollback lock.** Be careful. A rollback that died halfway may have left the disk in a mixed state. Consider restoring from a known good backup rather than trusting the guest.

**After a migrate lock.** Confirm which node holds the disks, then unlock on the node that owns the config. Leftover disks on the other side show up as `unused` entries or as loose volumes on storage, and they are yours to delete by hand.

## The lock that returns

If the lock keeps coming back after you clear it, something is starting the same operation again. Check the scheduled backup job and run it manually while watching `journalctl -f`. A repeating lock almost always points at storage that is slow or unreachable, so fix that first. Unlocking is only a bandage over a dead NFS mount.

One last habit worth having: before unlocking anything, copy the config somewhere. A `cp /etc/pve/lxc/101.conf /root/101.conf.bak` costs nothing and gives you a reference if the cleanup goes sideways.
