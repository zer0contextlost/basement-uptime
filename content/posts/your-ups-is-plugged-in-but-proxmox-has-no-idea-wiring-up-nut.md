---
title: "Your UPS Is Plugged In, But Proxmox Has No Idea: Wiring Up NUT for Real Shutdowns"
date: 2026-08-13
description: "A UPS that isn't talking to your hypervisor just delays the crash. How to set up NUT on Proxmox so hosts and guests actually shut down before the battery dies."
tags: ["proxmox", "monitoring"]
---

Most homelab UPS setups amount to a box under the rack that beeps during an outage and keeps the server alive for another twenty minutes. That's not protection, it's a countdown. If nothing on the network is watching the battery level, the outage either ends before the UPS dies or it doesn't, and when it doesn't, every VM and LXC on that host goes down as hard as if you'd pulled the cord yourself. ZFS pools survive that most of the time. Application databases and container filesystems don't always.

Network UPS Tools (NUT) closes that gap. One machine talks to the UPS over USB, and every other host on the network asks that machine for status over the LAN. When the battery crosses a threshold, NUT can trigger an orderly shutdown of every dependent host before the power actually runs out.

## The two roles

NUT splits into a server and however many clients you need:

- **nut-server** runs on whichever machine has the physical USB connection to the UPS. This is usually your Proxmox host or a small always-on box like a NAS.
- **nut-client** (or `upsmon` in client mode) runs on every other machine that should shut down when the battery gets low, even though none of them can see the UPS directly.

On the server side, install `nut` and figure out which driver matches your UPS. Most consumer and prosumer units talk `usbhid-ups`. Check `/lib/nut/` for available drivers, then set up `/etc/nut/ups.conf`:

```
[serverups]
    driver = usbhid-ups
    port = auto
    desc = "Rack UPS"
```

`/etc/nut/upsd.conf` needs a `LISTEN` line for the LAN interface, not just localhost:

```
LISTEN 0.0.0.0 3493
```

And `/etc/nut/upsd.users` needs at least one account with `upsmon` privileges for the monitoring role, plus one for remote clients to authenticate as:

```
[monuser]
    password = somepassword
    upsmon master

[remoteuser]
    password = anotherpassword
    upsmon secondary
```

Set `MODE=netserver` in `/etc/nut/nut.conf`, restart the `nut-server` and `nut-monitor` services, and confirm locally with `upsc serverups@localhost`. You should see battery charge, load, and status flags.

## Getting other hosts to listen

On each client, install `nut-client`, set `MODE=netclient`, and point `/etc/nut/upsmon.conf` at the server:

```
MONITOR serverups@192.168.1.10 1 remoteuser anotherpassword secondary
```

The `1` is the power value, how many "UPS units" this counts as for load calculation purposes; leave it at 1 unless you're doing something unusual with dual-fed redundant supplies. Restart `nut-monitor` and check `upsc serverups@192.168.1.10` from the client to confirm it can reach the server across the network.

## Making the shutdown actually happen

The default `upsmon.conf` on the server already defines `NOTIFYCMD` and shutdown thresholds, but the piece people skip is `SHUTDOWNCMD`. On the Proxmox host itself, this should gracefully stop guests before powering off, not just call `shutdown -h now` and hope for the best:

```
SHUTDOWNCMD "/usr/bin/systemctl poweroff"
```

Proxmox's own shutdown sequence already stops VMs and containers cleanly when systemd asks it to power off, so you generally don't need extra scripting there. What you do need is the `FSD` (forced shutdown) flag propagating correctly to secondary clients, and that only works if `MINSUPPLIES` and the timing values in `upsmon.conf` are consistent across server and clients. If a secondary client never receives the low-battery notification because the server itself already powered off, you've built a race condition instead of a safety net.

Test this before you need it. Pull the UPS's input power (not just unplug from the wall if your model has other quirks), watch `upsc` update the charge percentage in real time, and confirm the shutdown actually fires at your configured threshold rather than at zero percent. Runtime estimates on most consumer UPS units drift badly as batteries age, and the last thing you want to discover during a real outage is that "20% remaining" actually meant four minutes, not twenty.

## Sizing the threshold correctly

Set the shutdown trigger well above the point where the UPS reports critical. A host with several VMs can take a couple of minutes to shut down cleanly, especially if any guest is running its own database with a slow checkpoint on stop. If your UPS starts screaming at 10% battery, don't wait until 5% to start the shutdown chain, trigger it at 20-25% and give the whole stack room to land softly. The battery you save by cutting it close isn't worth the filesystem you might not get back.
