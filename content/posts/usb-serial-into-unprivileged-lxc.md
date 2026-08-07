---
title: "Passing a USB Serial or Webcam Device Into an Unprivileged Proxmox LXC"
date: 2026-08-06
description: "Group membership doesn't save you here — unprivileged LXC UID remapping means the container sees your bind-mounted device as nobody:nogroup. What actually grants access, and the two config lines that do it."
tags: ["proxmox", "lxc", "homelab"]
---

Unprivileged LXC containers are the right default on Proxmox. Root inside
the container isn't root on the host. That same UID remapping breaks the
usual instinct for granting device access: add the service user to
`dialout` or `video` or whatever group. It won't work, and it's worth
knowing why before an hour goes into staring at a permission-denied error.

## Why group membership fails

When an unprivileged container bind-mounts a host device node, the
container's view of that device's ownership goes through the same UID/GID
remapping as everything else. A device owned by `root:dialout` on the
host shows up inside the container as some remapped, meaningless UID:GID
pair, typically rendered as `nobody:nogroup`. Adding your service user to
a group inside the container does nothing. The group the device actually
belongs to, from the container's point of view, isn't a group that exists
in any meaningful sense inside that namespace.

## What actually works: world read-write, at the host level

Make the device world-RW on the host via a udev rule, so permissions don't
depend on group membership surviving the remap at all.

```
# /etc/udev/rules.d/99-mydevice.rules  (on the Proxmox host)
SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", ATTRS{idProduct}=="7523", MODE="0666"
```

Match on vendor/product ID (`lsusb` shows both) rather than a device path
like `/dev/ttyUSB0`. Those paths shift if other USB-serial adapters get
plugged in a different order after a reboot; the vendor/product ID pair
doesn't.

Then, on the container side, two lines in `/etc/pve/lxc/<vmid>.conf`:

```
lxc.cgroup2.devices.allow: c 188:0 rw
lxc.mount.entry: /dev/ttyUSB0 dev/ttyUSB0 none bind,optional,create=file
```

The major:minor pair (`188:0` here, for a USB-serial `ttyUSB0`) has to
match what the device actually enumerates as on the host. Check with
`ls -la /dev/ttyUSB0` before writing the config. A UVC webcam behaves the
same way but typically exposes two nodes, a capture node and a metadata
node, so both need cgroup allow and mount lines.

## Two gotchas up front

New `lxc.mount.entry` lines only take effect at container start, not a
service restart. Adding device passthrough to a running container means a
full `pct reboot <vmid>`, not restarting whatever service inside is trying
to use the device.

Re-verify the major:minor pairing if the device is ever unplugged and
replugged alongside other USB-serial devices. Enumeration order isn't
guaranteed, and a config pinned to the wrong major:minor silently fails to
grant access to the device you actually care about.

Once both pieces are in place, the container sees a world-RW device node
and doesn't need a matching group at all. That's the whole trick for
working around the UID remap.
