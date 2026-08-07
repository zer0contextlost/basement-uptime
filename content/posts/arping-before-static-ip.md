---
title: "ip neigh show Lies: Always arping Before You Assign a Static LAN IP"
date: 2026-08-06
description: "ip neighbor show only reflects hosts that have talked recently. An absent ARP entry does not mean an IP is free — here's why that matters and the one-line check that actually proves it."
tags: ["networking", "homelab", "proxmox"]
---

You're spinning up a new container or VM and need a free static IP. The
instinctive check:

```
ip neigh show
```

If the candidate address isn't in the table, it looks free. It isn't
necessarily. The neighbor table only holds entries for hosts your machine
has actually exchanged traffic with recently, usually within the last few
minutes to a few hours depending on the kernel's `gc_stale_time` and
related sysctls. A device that's been quietly sitting on the network
without talking to *this particular host* won't show up. Stale entry or
not, it just isn't there.

## Where this bites you

Phones, IoT gear, and other quiet devices can go a long time without
generating traffic your hypervisor host happens to see. Assign a "free"
static IP based only on `ip neigh show` and you can collide with a real
device. The symptom is usually intermittent, not a hard conflict message.
Two MAC addresses answer ARP requests for the same IP and you get a race.
Ping works sometimes, fails other times. The inconsistency makes it look
like anything but an IP conflict.

## The actual check

Don't trust the passive cache. Force an active probe:

```
arping -c 3 -I vmbr0 192.168.1.50
```

Swap `vmbr0` for whatever interface sits on the LAN you're checking, and
the IP for your candidate address.

Three ARP request broadcasts. If anything answers, you'll see it
immediately, regardless of whether that device has talked to your host
recently. Zero responses across three probes is a much stronger signal of
"actually free" than an empty neighbor table.

## The habit worth building

Before assigning any new static IP on a shared LAN:

1. `arping -c 3 -I <bridge> <candidate-ip>`. Zero replies is a good sign.
2. Scanning a range for the next free slot means arping each candidate in
   turn, not trusting one bulk `ip neigh show` snapshot.
3. Only then commit the address to your VM/container config.

It costs about a second per address. The alternative is a confusing
debugging session an hour later when a phone on the couch starts dropping
packets.
