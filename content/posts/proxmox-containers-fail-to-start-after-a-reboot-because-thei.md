---
title: "Proxmox Containers Fail to Start After a Reboot Because Their Mount Point Wasn't Ready"
date: 2026-10-15
description: "A Proxmox LXC with a bind mount can start before the underlying disk or share is mounted, then write into the empty directory. Here is how to catch and prevent it."
tags: ["proxmox", "lxc", "storage", "troubleshooting"]
---

After a power outage, one of my containers came back up looking healthy. The service was running, the web UI loaded, and nothing in the logs complained. Two days later I noticed the media library was empty and the root disk of the host was at 94 percent.

The container had started before the host mounted the second disk. The bind mount pointed at an empty directory on the root filesystem, and the app happily wrote its new data there.

## What actually happens

A bind mount in an LXC config is just a path on the host:

```
mp0: /mnt/media,mp=/media
```

Proxmox does not check that `/mnt/media` is a mounted filesystem. If the disk behind it isn't mounted yet, the directory still exists, because you created it once as a mountpoint. The container sees an empty folder and carries on.

Later, when the real disk mounts, it covers whatever was written underneath. The data isn't gone. It's hiding in the directory beneath the mount, eating space on the root filesystem and invisible to `ls` as long as the disk is mounted.

This hits two kinds of setups most often:

- Extra disks mounted through `/etc/fstab` that are slow to appear (USB enclosures, spun-down HDDs, encrypted volumes unlocked after boot).
- Network shares such as NFS or SMB, where the network isn't up when `pve-guests` fires.

## Confirming it happened

Check whether the path is currently a mountpoint:

```
mountpoint /mnt/media
```

If it says it is, the problem may be hiding underneath. Bind-mount the root filesystem somewhere else to look below the overlay:

```
mkdir -p /root/rootview
mount --bind / /root/rootview
ls -la /root/rootview/mnt/media
du -sh /root/rootview/mnt/media
umount /root/rootview
```

A bind of `/` does not carry submounts along, so you see the raw directory. If `du` reports gigabytes where you expected nothing, that's your stray data. Move it onto the real disk (with `rsync -a`) after stopping the container and confirming the target has room, then empty the underlying directory.

## Fix one: make the empty directory unwritable

The cheapest guard is to make the bare mountpoint useless. With the disk unmounted:

```
chattr +i /mnt/media
```

The immutable flag on the underlying directory means writes fail when nothing is mounted over it. Once the real filesystem mounts on top, the flag on the directory beneath no longer matters and the mounted filesystem behaves normally. A container that starts too early gets "permission denied" errors instead of silently filling your root disk. Loud failure beats quiet corruption.

Run `chattr -i` first if you ever need to put data in that directory again.

## Fix two: tell Proxmox to wait

For fstab mounts, add options so systemd treats the mount as a real dependency:

```
UUID=xxxx-xxxx  /mnt/media  ext4  defaults,nofail,x-systemd.device-timeout=30  0  2
```

`nofail` keeps a missing disk from dropping the host into emergency mode. That is good, but it also means boot proceeds without the disk, which is exactly how you got here. So pair it with an ordering rule for the guests.

Proxmox starts guests from `pve-guests.service`. A drop-in can make it wait for the mount:

```
systemctl edit pve-guests.service
```

```
[Unit]
RequiresMountsFor=/mnt/media
```

With `RequiresMountsFor`, systemd pulls in the mount unit for that path and orders guest startup after it. If the mount fails, `pve-guests` fails too, and no container starts against an empty directory. For a host where every guest depends on that storage this is what you want. If only one container depends on it, read on.

## Fix three: gate a single container

Where only some guests need the disk, a hook script in the container config can refuse to start them. Add to `/etc/pve/lxc/101.conf`:

```
hookscript: local:snippets/check-mount.sh
```

And in `/var/lib/vz/snippets/check-mount.sh` (mark it executable):

```bash
#!/bin/bash
vmid="$1"
phase="$2"

if [ "$phase" = "pre-start" ]; then
    if ! mountpoint -q /mnt/media; then
        echo "pre-start: /mnt/media is not mounted, refusing to start $vmid"
        exit 1
    fi
fi
exit 0
```

A non-zero exit from pre-start aborts the start, and the task log shows your message. Pair it with `startup: order=2,up=30` or a retry job so the container isn't just dead until you notice.

## Network shares need extra care

For NFS or SMB, add `_netdev` and `x-systemd.automount` or `x-systemd.requires=network-online.target` so the mount waits for the network. Be aware that a hung NFS mount causes its own problems, so prefer `soft` or a short `x-systemd.mount-timeout` on anything non-critical.

## Add a check that would have caught it

I now run a tiny cron job that alerts when the root filesystem usage jumps, and another that pushes a heartbeat only if `mountpoint -q /mnt/media` succeeds. The first would have flagged the problem within an hour. The second would have told me the disk was missing before any container had a chance to write to it.
