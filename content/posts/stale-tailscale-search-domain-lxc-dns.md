---
title: "The Stale Search-Domain Bug That Silently Breaks DNS in Fresh LXC Containers"
date: 2026-08-06
description: "A dead resolver from a Tailscale experiment months ago kept leaking into brand-new containers, breaking DNS resolution in ways that look like an app bug. Here's how it propagates and the two ways to catch it."
tags: ["dns", "proxmox", "lxc", "tailscale"]
---

Symptom: a brand-new container fails an `apt-get install` with a
resolution error, or some app inside it can't reach the outside world,
even though the container clearly has a route out and `ping <ip>` to a raw
address works fine. The instinct is to blame the app. Check
`/etc/resolv.conf` first.

## What was actually happening

Run Tailscale on a Proxmox host or one of its containers, even briefly,
even just to test something, then remove it, and it can leave behind a
stale `search` directive in the resolver config:

```
search taild14820.ts.net
```

Tailscale's MagicDNS appends a search domain like this so short hostnames
resolve within your tailnet. Uninstall Tailscale without cleaning that
config up and the search domain doesn't go away on its own. If it ends up
in the *host's* base resolver config rather than just one container's, it
keeps quietly propagating into every new guest created afterward, because
`pct create` without an explicit `--searchdomain` flag inherits from the
host's own resolver state at creation time.

The practical effect: any bare hostname lookup gets `.taild14820.ts.net`
appended before the real domain is tried, adding a failing lookup against
a tailnet with nothing registered in front of every real one. Some
resolvers handle this gracefully and just take the latency hit. Others
surface it as an outright failure.

## How to catch it

Two places to check whenever DNS looks flaky on a freshly created guest:

```
cat /etc/resolv.conf
```

Look for a `search` line referencing a domain you don't recognize.
`*.ts.net` is the Tailscale MagicDNS tell, but the same class of bug can
come from any VPN or mesh tool that rewrites resolver config and doesn't
clean up on removal.

If it's present on more than one freshly created guest, the leak is at
the host level, not per-container. Check the Proxmox host's own
`/etc/resolv.conf` too. That's very likely where new containers are
inheriting it from.

## Fixes, in order of correctness

1. Best: find and remove the stale entry at the host level, so it stops
   propagating to every future container. This usually means whatever
   config file the VPN tool's uninstall left behind. Check for anything
   still referencing the mesh's DNS.
2. Workaround at creation time: pass an explicit `--searchdomain` when
   running `pct create`, overriding whatever the host would otherwise
   hand the new container.
3. Per-container fix: edit `/etc/resolv.conf` directly inside an
   already-running container. Fine for a one-off. It doesn't stop the
   next container you create from inheriting the same stale config.

Managing more than one or two containers means the host-level source is
worth fixing once it's confirmed. Chasing the same bug container by
container gets old fast.
