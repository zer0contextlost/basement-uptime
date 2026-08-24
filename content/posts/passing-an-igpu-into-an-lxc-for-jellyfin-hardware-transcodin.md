---
title: "Passing an iGPU Into an LXC for Jellyfin Hardware Transcoding"
date: 2026-08-24
description: "How to share an Intel iGPU with an unprivileged Proxmox LXC for Jellyfin/Plex, and the render-group GID trap that breaks it."
tags: ["proxmox", "lxc", "jellyfin"]
---

Software transcoding a 4K stream on a low-power homelab box pegs every core and still drops frames. An iGPU sitting idle in the same chassis can do the same job at a fraction of the wattage, if you can get it into the container. LXC makes this far less painful than VM passthrough, because you're not fighting IOMMU groups or VFIO binding. You're just sharing device nodes.

## Why a container instead of a VM

A VM needs the GPU handed over exclusively, usually via PCI passthrough, which means the host loses access to it and only one guest can use it at a time. An LXC container shares the kernel with the host, so the iGPU's device nodes can be bind-mounted straight in. Multiple containers can point at the same `/dev/dri` devices simultaneously, which matters if you're running Jellyfin in one container and, say, Immich's ML service in another and both want hardware acceleration.

## Finding the device nodes

On the Proxmox host, check what the iGPU exposes:

```
ls -l /dev/dri
```

You'll typically see `card0` and a `renderD1xx` node. `card0` is the full display device; `renderD128` (or similar) is the render-only node, which is what you actually want for transcoding. Handing over `card0` isn't necessary and drags in permission requirements you don't need.

## Wiring it into the LXC config

Edit the container's config on the host, at `/etc/pve/lxc/<vmid>.conf`, and add:

```
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
```

The major number `226` is consistent for DRM devices; the minor numbers correspond to card0 and the render node. Run `ls -la /dev/dri` if you want to confirm the minors on your box before copying config blindly from someone else's post, mine included.

Restart the container. Inside it, `ls -l /dev/dri` should now show the same nodes.

## The render group GID trap

Here's where it usually breaks. The render node inside the container is owned by root and a group, typically named `render`, with some numeric GID. Your Jellyfin process, whether it's running as a system user or inside a Docker container nested in the LXC, is not root and needs to be in that group to actually use `renderD128`.

The catch: an unprivileged LXC shifts UIDs and GIDs by an offset from the host, but bind-mounted device files keep the host's raw ownership numbers. So the GID that looks like `render` on the host doesn't line up with any group the container knows about by default, and adding a user to a group called `render` inside the container won't match unless the GIDs actually agree.

The fix is to create a group inside the container with the exact same numeric GID as the host's render group, then add your transcoding user to that group. Check the host's GID with:

```
getent group render
```

Then inside the container:

```
groupadd -g <that-number> render
usermod -aG render <jellyfin-user>
```

If Jellyfin is running in Docker nested inside the LXC, that user is whatever UID the container process runs as, and you'll need to set the render GID in the Docker compose file's `group_add` or `devices` section too, not just in the LXC's own user table. Getting this right at the LXC level and then forgetting Docker has its own group mapping is the single most common reason people give up and conclude passthrough "doesn't work."

## Confirming it's actually working

Don't trust Jellyfin's dashboard alone. From inside the container (or the Docker container), run:

```
vainfo
```

It should list supported profiles without erroring out on device access. During an actual transcode, `intel_gpu_top` on the host will show render engine usage climbing, which is the real tell that the stream is riding the iGPU and not quietly falling back to software decode because a permission check failed silently.

## Sharing across containers

Because the device is just bind-mounted, nothing stops you from repeating this same block of config in a second, third, or fourth LXC. The iGPU has no concept of exclusive ownership the way a passthrough GPU does. The only real constraint is how many concurrent transcodes the hardware can actually push before quality or speed starts to suffer, and that's a hardware ceiling, not a configuration one.
