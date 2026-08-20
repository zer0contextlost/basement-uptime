---
title: "Docker Inside an Unprivileged LXC: The Nesting Flags and Storage Driver Gotchas"
date: 2026-08-20
description: "Running Docker inside a Proxmox LXC instead of a VM saves RAM, but the nesting flags and overlay storage driver will bite you if you skip the details."
tags: ["proxmox", "lxc", "docker"]
---

Running Docker inside an LXC container instead of giving it a full VM is one of those things that works great until it doesn't, and when it doesn't, the error messages point everywhere except the actual cause. The container starts, `docker run hello-world` might even work, and then three weeks later a container that writes a lot of small files starts throwing filesystem errors that make no sense.

## Why bother with this at all

A VM running Docker needs its own kernel, its own memory reservation, and its own boot overhead. An LXC container shares the host kernel, starts in about a second, and lets you overcommit memory far more aggressively because Proxmox's LXC containers use the host's page cache directly instead of each VM caching independently. For a homelab running a dozen small services, that difference adds up fast. The tradeoff is that you're now running Docker-in-container instead of Docker-on-bare-metal, and LXC wasn't originally built with "run another container runtime inside me" as the primary use case.

## The two flags that actually matter

In the Proxmox container options, under Features, you need:

- **Nesting**: `nesting=1`
- **Keyctl**: `keyctl=1`

Nesting allows the container to mount its own cgroups and use the namespacing primitives Docker needs to create its own containers. Without it, `dockerd` fails to start with cgroup-related errors that don't mention nesting anywhere in the text, so if Docker won't start at all, this is the first thing to check.

Keyctl relates to the kernel keyring. Some Docker functionality and several images that use tools like GPG or certain credential helpers will fail silently or hang without it. It's a smaller footgun than nesting, but cheap to enable and annoying to debug when missing.

You can set both from the CLI on the Proxmox host:

```bash
pct set 105 --features nesting=1,keyctl=1
```

Or edit `/etc/pve/lxc/105.conf` directly:

```
features: nesting=1,keyctl=1
```

The container needs a restart, not just a reload, for these to take effect.

## The storage driver problem nobody warns you about

This is the part that actually causes production headaches. Docker wants to use the `overlay2` storage driver by default, which itself relies on overlayfs, a filesystem that stacks layers. The trouble is that overlayfs doesn't like being run on top of certain filesystems, and ZFS is one of them. If your Proxmox host stores the LXC's rootfs on a ZFS dataset (which is the default for a lot of ZFS-based Proxmox setups), Docker inside that container may refuse to use overlay2, or worse, appear to use it and then produce corrupted layers under load.

Check what driver Docker actually picked:

```bash
docker info | grep "Storage Driver"
```

If you see `overlay2` and your LXC's backing storage is ZFS, don't assume it's fine just because containers start. Test it under actual write load, not just a hello-world pull.

Your options, roughly in order of how much hassle they are:

- **Give the LXC a separate mount point backed by ext4** (a raw disk image or a dedicated zvol formatted ext4, not the ZFS dataset directly) and point Docker's data-root at that mount. This gets you real overlay2 support with none of the ZFS-on-overlay weirdness.
- **Use `vfs` as the storage driver.** It works everywhere but copies entire layers instead of stacking them, so image pulls are slower and disk usage is higher. Fine for a couple of lightweight services, painful once you're running anything with large images.
- **Switch that specific workload to a VM.** If a service is disk-I/O heavy (databases, anything doing lots of small writes), the storage driver headache alone is often a sign it belongs in a VM with a real block device, not nested inside a container's container.

The ext4-mountpoint approach is the one I'd default to for anything beyond a couple of throwaway containers. Add a mount point in the LXC config pointing at a raw-formatted volume, set `/etc/docker/daemon.json` inside the container to point `data-root` at that mount, and restart the Docker daemon. It's a few extra minutes of setup and it removes an entire category of "why did my container's filesystem just go read-only" incidents.

## AppArmor and unprivileged containers

Unprivileged LXC containers run under an AppArmor profile that restricts a chunk of what would normally be allowed inside, and Docker's own container creation trips over parts of it. Proxmox ships a profile (`lxc-container-default-cgns` or similar depending on version) that's usually sufficient once nesting is enabled, but if you see permission-denied errors that don't correspond to any file permission you can find, check `dmesg` on the host for AppArmor denials before you start chasing the problem inside the container. The host log will tell you what got blocked; the container's own logs usually won't.

None of this means you should give up and run everything privileged. It means budget an extra twenty minutes for the storage driver check before you trust a Docker-in-LXC setup with anything you'd be annoyed to lose.
