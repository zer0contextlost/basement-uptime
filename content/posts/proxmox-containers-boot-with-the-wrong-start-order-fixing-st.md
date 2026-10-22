---
title: "Proxmox Containers Boot With the Wrong Start Order: Fixing Startup Delays and Dependencies"
date: 2026-10-22
description: "Proxmox starts guests in parallel unless you tell it otherwise. Here is how order, up, and down values keep DNS, databases, and apps from racing each other at boot."
tags: ["proxmox", "lxc", "boot-order", "homelab"]
---

After a power outage, my homelab came back up in a state I can only describe as polite chaos. Everything was running. Nothing worked. The app containers had started before the database, the database had started before its storage was mounted, and the DNS container was still initializing when everything else went looking for it.

Proxmox does not know about your dependencies. It only knows what you tell it about order, and by default it tells it nothing.

## What the default actually does

Guests with "Start at boot" enabled all get started when the node comes up. Without any order set, they launch in no particular sequence you should rely on. In practice that is often by VMID, but if you have ever depended on that, you have been lucky.

The setting lives in each guest's Options tab under "Start/Shutdown order". From the shell, it looks like this:

```
pct set 101 --onboot 1 --startup order=1,up=30,down=60
qm set 201 --onboot 1 --startup order=3,up=60
```

Three values matter:

- `order`: a number. Lower starts first. Shutdown runs in reverse.
- `up`: seconds to wait after this guest starts before moving to the next order.
- `down`: seconds to wait for this guest to shut down before Proxmox gives up and stops it hard.

Guests with no order set start last, after all the ordered ones, and shut down first.

## Order is a gate, not a health check

This is the part that bites people. The `up` value is a plain sleep. Proxmox starts guest A, waits `up` seconds, then starts the next tier. It does not check that A's service is answering. For an LXC, "started" means the container's init has launched. For a VM with the guest agent enabled, Proxmox can wait for the agent to report in, which is a little better, but it still says nothing about whether PostgreSQL has finished crash recovery.

So you pick delays by measuring. Reboot the node, watch how long your slowest dependency takes to become usable, and add margin. A database that replays a long journal after an unclean shutdown takes far longer than one that was stopped cleanly. I size `up` for the unclean case, because that is the case where ordering matters most.

## A tier layout that works

My layout is boring, and boring is the point:

| Order | What | Why first |
|-------|------|-----------|
| 1 | DNS (Pi-hole or similar) | Everything resolves names at startup |
| 2 | Storage-dependent guests, NFS servers | Others mount from these |
| 3 | Databases | Apps connect on launch |
| 4 | Reverse proxy and apps | Need all of the above |
| 5 | Monitoring | Should observe a settled system |

Give the DNS tier a short `up`, perhaps 15 to 30 seconds. Give databases longer. Give the last tier nothing at all.

One caution on DNS. If your Proxmox node itself uses that DNS container as its resolver, the node will have no name resolution until the container is up. Keep a second resolver in the node's config so that boot-time tasks do not stall.

## Startup delay for the whole node

There is also a node-level knob. Under the node's System > Options, "Start/Shutdown order" lets you inspect the defaults, and the config in `/etc/pve/datacenter.cfg` and node settings can hold a global delay. If your network gear takes a minute to hand out link after a power cut, a node that boots faster than the switch will start guests onto a dead network. Adding a delay at the front of the sequence, or a first-tier guest with a generous `up`, covers that.

## Shutdown matters as much as startup

The `down` value is a timeout, and the default is 60 seconds. A database container with a large buffer to flush can need more. If the timeout expires, Proxmox kills the guest, and you get the unclean shutdown you were trying to avoid, plus the long recovery on the next boot.

Because shutdown runs in reverse order, apps stop first, then databases, then DNS last. That is what you want. Check it by running a `shutdown -r` during a quiet window and reading the task log in the GUI. Each guest shows its stop time, and any that hit the timeout are obvious.

## Checking what you configured

To see every guest's startup settings at once:

```
grep -H -E '^(onboot|startup)' /etc/pve/lxc/*.conf /etc/pve/qemu-server/*.conf
```

Any guest with `onboot: 1` and no `startup:` line is in the unordered pile. I found two of mine there, one of them a container that mounted a share and failed every cold boot until I noticed.

Order also does nothing for guests on other cluster nodes, since the settings are per node at boot time. In a cluster, a guest managed by HA follows the HA manager's rules instead, and the start order you set by hand will not govern it.

The first test worth doing is the ugly one: pull the power plug on a lab node, let it come back untouched, and see which service you have to restart by hand. That list is your missing `order` values.
