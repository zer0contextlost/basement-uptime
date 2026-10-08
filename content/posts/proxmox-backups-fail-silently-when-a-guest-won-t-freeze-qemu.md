---
title: "Proxmox Backups Fail Silently When a Guest Won't Freeze: QEMU Guest Agent and fs-freeze Timeouts"
date: 2026-10-08
description: "A VM backup that hangs or logs a guest-agent freeze error usually points at a stuck fsfreeze. Here is how to find the cause and what to change."
tags: ["proxmox", "backups", "qemu-guest-agent", "vm"]
---

The first sign is usually boring. A nightly vzdump job that used to take four minutes now takes forty, or it sits at 0% on one VM while the rest finish. Open the task log and you find something like this:

```
INFO: issuing guest-agent 'fs-freeze' command
ERROR: VM 101 qmp command 'guest-fsfreeze-freeze' failed - got timeout
INFO: issuing guest-agent 'fs-thaw' command
```

Sometimes the job continues without a consistent snapshot. Sometimes the VM itself goes unresponsive for the length of the timeout. Both outcomes are bad, and the second one is the kind that makes your monitoring page you at 2 a.m.

## What the freeze actually does

With the QEMU guest agent enabled, a snapshot-mode backup asks the agent inside the guest to freeze its filesystems. The guest flushes dirty pages, blocks new writes, and reports back. Proxmox then starts the backup point-in-time copy and sends a thaw command.

Without the agent, vzdump still works, but the backup is only crash-consistent. That's the same state the disk would be in after pulling the power cord. Most journaled filesystems recover from that. Databases usually do too, but "usually" is carrying a lot of weight.

The freeze can fail or stall for a few boring reasons:

- The agent service isn't running, or isn't installed, but the VM option says it is.
- A filesystem inside the guest can't be frozen, such as a FUSE mount or a network share that has gone away.
- Heavy write load means the flush takes longer than the timeout.
- Something inside the guest is holding a lock the freeze has to wait for.

## Check the basics first

On the host, confirm the agent answers:

```
qm agent 101 ping
```

No output and exit code 0 means it responded. An error like "QEMU guest agent is not running" tells you the option is enabled in the VM config but nothing is listening in the guest. Inside the guest:

```
systemctl status qemu-guest-agent
```

On Debian and Ubuntu guests the package is `qemu-guest-agent`. The VM needs the agent option ticked under Options, and a full stop and start of the VM afterward. A reboot from inside the guest won't always apply the hardware change.

## Find the filesystem that won't freeze

This one cost me an evening. The agent was running, ping worked, and the freeze still timed out on exactly one VM. That VM had an NFS mount for media, and the NAS had been rebooted earlier that week. The mount was stale.

The guest agent walks every mounted filesystem. A hung network mount can block the whole freeze call, because the agent tries to touch it and the kernel waits on the server.

Inside the guest, check what is mounted and whether it responds:

```
findmnt -t nfs,nfs4,cifs,fuse
timeout 5 ls /mnt/media || echo "mount is stuck"
```

If a mount is hung, fix or lazily unmount it. Longer term, mount network shares with options that fail instead of hanging forever. For NFS, `soft` with a sane `timeo` helps, though soft mounts carry their own risk of silent write errors, so think about what you store there. I only use them for read-mostly media.

You can also tell the agent to skip filesystems. On Linux guests, the agent supports a freeze hook directory and a block list of filesystem types or mount points through its config file at `/etc/default/qemu-guest-agent` or `/etc/qemu/qemu-ga.conf`, depending on the distribution. Check your distro's packaging, as the exact path and option names vary.

## Freeze hooks for databases

The freeze is also the right moment to get a database into a clean state. The agent runs scripts from a hook directory (commonly `/etc/qemu/fsfreeze-hook.d/`) with the argument `freeze` before the freeze and `thaw` after. A hook that runs a quick flush or pauses a writer makes the backup much more trustworthy.

Keep hooks short. A hook that takes longer than the agent timeout produces exactly the error you were trying to get rid of. If the script has to wait on a big checkpoint, you've moved the problem, not solved it.

## Raising the timeout is a last resort

The agent call has a timeout on the Proxmox side. If the guest is simply busy, such as a big database under constant writes, the freeze can legitimately take a while. Before touching timeouts, schedule the backup for a quieter window. A 3 a.m. job that overlaps with the guest's own nightly batch work is a common culprit, and moving one of them by an hour fixes it for free.

If a VM can't be frozen safely at all, two options remain. You can disable the agent option for that VM and accept crash-consistent backups, or use stop mode for the backup so the guest is shut down cleanly during the copy. For a small service that tolerates a brief outage, stop mode is the most honest option, and it is trivially restorable.

## Verify after the fix

Run the job manually against the one VM and read the log top to bottom. You want to see the freeze command, a short gap, and the thaw, all within a second or two on an idle guest. Then do a test restore to a spare VMID and boot it. A clean log tells you the freeze worked. Only the restore tells you the backup is any good.

If you inherit a setup with a dozen VMs, run `qm agent <vmid> ping` in a loop across all of them once. Every one that errors out has been quietly taking crash-consistent backups, and you will want to know which ones before you need them.
