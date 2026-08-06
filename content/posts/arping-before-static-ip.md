---
title: "ip neigh show Lies: Always arping Before You Assign a Static LAN IP"
date: 2026-08-06
description: "ip neighbor show only reflects hosts that have talked recently. An absent ARP entry does not mean an IP is free — here's why that matters and the one-line check that actually proves it."
tags: ["networking", "homelab", "proxmox"]
---

You're spinning up a new container or VM and you need a free static IP. The
instinctive check is:

```
ip neigh show
```

If the candidate address isn't in the table, it looks free. It isn't
necessarily. The neighbor table (ARP cache) only holds entries for hosts your
machine has actually exchanged traffic with recently — usually within the
last few minutes to a few hours, depending on the kernel's `gc_stale_time`
and related sysctls. A device that's been quietly sitting on the network
without talking to *this particular host* simply won't show up, stale entry
or not.

## Where this bites you

On a home LAN this is common: phones, IoT gear, and other quiet devices can
go a long time without generating traffic that your hypervisor host happens
to see. If you assign a "free" static IP based only on `ip neigh show`, you
can collide with a real device. The symptom is usually intermittent and
confusing — not a hard conflict message, but flaky connectivity, because
you get a race between two different MAC addresses answering ARP requests
for the same IP. Ping works sometimes, fails other times, and the
inconsistency makes it look like anything *but* an IP conflict.

## The actual check

Don't trust the passive cache — force an active probe:

```
arping -c 3 -I vmbr0 192.168.1.50
```

(swap `vmbr0` for whatever interface sits on the LAN you're checking, and
the IP for your candidate address)

Three ARP request broadcasts, and if anything answers, you'll see it
immediately — regardless of whether that device has talked to your host
recently. Zero responses across three probes is a much stronger signal
of "actually free" than an empty neighbor table.

## The habit worth building

Before assigning any new static IP on a shared LAN:

1. `arping -c 3 -I <bridge> <candidate-ip>` — zero replies, good sign
2. If you're scanning a range for the next free slot, arping each candidate
   in turn rather than trusting one bulk `ip neigh show` snapshot
3. Only then commit the address to your VM/container config

It costs about a second per address and it's the difference between "IP
assignment" and "IP assignment, then a confusing debugging session an hour
later when a phone on the couch starts dropping packets."
