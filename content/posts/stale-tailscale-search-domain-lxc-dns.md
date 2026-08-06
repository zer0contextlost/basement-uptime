---
title: "The Stale Search-Domain Bug That Silently Breaks DNS in Fresh LXC Containers"
date: 2026-08-06
description: "A dead resolver from a Tailscale experiment months ago kept leaking into brand-new containers, breaking DNS resolution in ways that look like an app bug. Here's how it propagates and the two ways to catch it."
tags: ["dns", "proxmox", "lxc", "tailscale"]
---

Symptom: a brand-new container fails a `apt-get install` with a resolution
error, or some app inside it can't reach the outside world, even though the
container clearly has a route out and `ping <ip>` to a raw address works
fine. The instinct is to blame the app. Check `/etc/resolv.conf` first.

## What was actually happening

If you've ever run Tailscale on a Proxmox host or one of its containers —
even briefly, even just to test something — and then removed it, it can
leave behind a stale `search` directive in the resolver config:

```
search taild14820.ts.net
```

Tailscale's MagicDNS appends a search domain like this so short hostnames
resolve within your tailnet. If Tailscale itself gets uninstalled without
that config being cleaned up, the search domain doesn't go away on its own
— and if it ends up in the *host's* base resolver config (rather than just
one container's), it will keep quietly propagating into every new guest
created afterward, because `pct create` without an explicit
`--searchdomain` flag inherits from the host's own resolver state at
creation time.

The practical effect: any bare hostname lookup gets `.taild14820.ts.net`
appended before the real domain is tried, adding a failing lookup (against
a tailnet that no longer has anything registered) in front of every real
one. Some resolvers/tools handle this gracefully and just take the latency
hit; others surface it as an outright failure.

## How to catch it

Two places to check whenever DNS looks flaky on a freshly created guest:

```
cat /etc/resolv.conf
```

Look for a `search` line referencing a domain you don't recognize —
`*.ts.net` is the Tailscale MagicDNS tell, but the same class of bug can
come from any VPN/mesh tool that rewrites resolver config and doesn't clean
up on removal.

If it's present on more than one freshly created guest, the leak is at the
host level, not per-container — check the Proxmox host's own
`/etc/resolv.conf` too, since that's very likely where new containers are
inheriting it from.

## Fixes, in order of correctness

1. **Best**: find and remove the stale entry at the host level, so it stops
   propagating to every future container. This usually means whatever
   config file the VPN tool's uninstall left behind — check for anything
   still referencing the mesh's DNS.
2. **Workaround at creation time**: pass an explicit `--searchdomain` when
   running `pct create`, overriding whatever the host would otherwise hand
   the new container.
3. **Per-container fix**: edit `/etc/resolv.conf` directly inside an
   already-running container. Fine for a one-off, but it doesn't stop the
   next container you create from inheriting the same stale config.

If you manage more than one or two containers, it's worth actually fixing
the host-level source once you've confirmed that's where it lives — chasing
this same bug container-by-container gets old fast.
