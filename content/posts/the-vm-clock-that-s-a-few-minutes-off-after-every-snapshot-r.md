---
title: "The VM Clock That's a Few Minutes Off After Every Snapshot Rollback"
date: 2026-08-30
description: "Why restoring a Proxmox VM snapshot leaves the guest clock drifted, and why that quietly breaks TLS and TOTP auth until NTP catches up."
tags: ["proxmox", "networking"]
---

You roll back a VM to a snapshot from last week to test something, and five minutes later a service inside it starts throwing TLS handshake errors, or a TOTP-based login rejects a code you know is correct. Nothing in the VM looks broken. The disk is fine, the service is running, the network is up. Check the clock and it's off by minutes, sometimes more, depending on how long ago the snapshot was taken.

This is a clock drift problem, and it's specific to how virtualization handles time, not a Proxmox bug.

## Why the Clock Freezes

A snapshot captures RAM and disk state at a point in time. The guest's internal clock is part of that captured state. When you roll back, the VM resumes with whatever timestamp it had when the snapshot was taken, then the guest kernel keeps ticking forward from there using its own clock source, unaware that real wall-clock time has moved on by hours, days, or weeks.

Modern guest kernels have some tolerance for this. QEMU exposes a paravirtualized clock (`kvm-clock` on Linux) that's supposed to resync against the host after events like this. In practice, resync after a snapshot rollback is inconsistent. Suspend/resume and live migration usually trigger a clean catch-up. Snapshot restore doesn't always. The guest boots back into a stale timeline and just keeps counting from there until something forces a correction.

## Why This Breaks TLS and TOTP Specifically

Both are unusually clock-sensitive:

- TLS certificate validation checks `notBefore`/`notAfter` against the client or server's local time. A clock that thinks it's five minutes in the past can reject a cert as "not yet valid" if the cert was issued after the drifted timestamp, or reject a peer's cert for the same reason in reverse.
- TOTP (the six-digit codes from an authenticator app) is time-windowed, typically a 30-second step with a small tolerance on either side. A few minutes of drift blows straight through that tolerance and every code fails until the clock is corrected.

Log timestamps get quietly wrong too, which is its own problem when you're trying to correlate an incident across multiple hosts later.

## Why NTP Doesn't Fix It Instantly

Most guests run `chrony` or `systemd-timesyncd` and assume it "just handles" clock sync. It does, but not instantly and not always by stepping the clock. NTP daemons are built around the idea that clocks drift by milliseconds, not minutes, so their default behavior is to *slew* the clock, nudging it gradually to avoid time jumping backward and confusing anything watching the clock (cron, logs, database timestamps). Slewing a multi-minute offset back into alignment can take a long time at a fraction of a second per adjustment.

`chrony` has a `makestep` directive that allows a hard step under specific conditions (usually: only within the first few clock updates after startup, or only if the offset exceeds some threshold). If your `chrony.conf` doesn't have a `makestep` line, or has one scoped too narrowly, a post-rollback drift can sit there for a long time instead of getting corrected in one shot.

Check what's actually configured:

```bash
grep makestep /etc/chrony/chrony.conf
```

A reasonable default for guest VMs that get snapshotted or migrated is something like:

```
makestep 1.0 3
```

This steps the clock immediately if it's off by more than a second, for up to the first three time updates after the service starts. That covers boot-time drift but not necessarily a mid-uptime rollback, since the "first 3 updates" window may have already passed by the time you restore an old snapshot into a VM that's been running a while.

## The More Reliable Fix: a QEMU Guest Agent Hook

If the QEMU guest agent is installed and running in the VM, Proxmox can request a clock sync through it after specific lifecycle events. It's not automatic for snapshot restore in every version, so the more dependable approach is a small systemd unit or cron job inside the guest that forces an NTP step on boot and periodically after, rather than trusting slew-only correction:

```bash
chronyc makestep
```

Run manually or triggered right after a rollback, this forces an immediate step regardless of the `makestep` config window. For VMs you snapshot and restore often (test environments especially), it's worth wiring this into a startup script rather than remembering to run it by hand.

## LXC Doesn't Have This Problem

Containers share the host's kernel and therefore the host's clock. There's no independent guest clock to drift, so this entire class of bug doesn't apply to LXC snapshot rollbacks. If you've only ever tested snapshot restores on containers, this failure mode will look completely new the first time you hit it on a VM.

If you rely on snapshots for testing, add clock verification to whatever you check right after a restore. `timedatectl` or `chronyc tracking` takes five seconds to check and saves you from chasing a TLS or auth failure that has nothing to do with the change you were actually testing.
