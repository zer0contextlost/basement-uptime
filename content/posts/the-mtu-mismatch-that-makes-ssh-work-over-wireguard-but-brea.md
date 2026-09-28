---
title: "The MTU Mismatch That Makes SSH Work Over WireGuard But Breaks Everything Else"
date: 2026-09-28
description: "Why a WireGuard tunnel can pass SSH and ping fine while web pages and file transfers hang, and how to fix it by getting the MTU right."
tags: ["wireguard", "networking", "vpn", "homelab"]
---

You bring up a WireGuard tunnel between your home network and a VPS or a second site. SSH works. Ping works. Then you try to load a web page through it, or `rsync` a directory, and the connection just stalls. Small packets sail through. Anything bigger sits there until the client gives up. This is almost always an MTU problem, and it's one of the more confusing ones to diagnose because the symptoms look like a routing issue instead of a packet-size issue.

## Why small packets work and big ones don't

SSH handshakes, DNS queries, and pings fit comfortably in a single small packet. TLS handshakes for HTTPS, and the bulk data in a file transfer, don't. Those get sent as full-size packets, typically 1500 bytes on a standard Ethernet link, and that's where things break.

WireGuard wraps every packet it sends in its own header: a UDP header, an IP header, and the WireGuard protocol overhead on top of that. For IPv4 this adds up to roughly 60 bytes per packet, more if you're running over IPv6 or nesting the tunnel inside another one. If your physical interface has an MTU of 1500 and your WireGuard interface also claims an MTU of 1500, every full-size packet WireGuard tries to send becomes larger than what the underlying link can actually carry. That packet either gets fragmented or dropped, depending on the DF (don't fragment) flag and whether anything downstream permits fragmentation at all.

In a well-behaved network, path MTU discovery is supposed to catch this: a router along the path sends back an ICMP "fragmentation needed" message, and the sending host adjusts. In practice, plenty of networks and cloud provider firewalls drop that ICMP message outright, either by policy or by accident. When that happens, PMTUD is broken silently, and you just get packets vanishing into a black hole. No error, no obvious log line pointing at the cause.

## Where this actually bites you

A few situations make this worse than the textbook case:

- **PPPoE or DSL links** often cap the real MTU at 1492 instead of 1500, so even a "correctly" configured WireGuard interface built for a standard 1500 link is already too big.
- **Double encapsulation** — running WireGuard over Tailscale, or a site-to-site tunnel that itself rides over another VPN — stacks overhead on overhead, and each layer needs headroom.
- **Docker's default bridge network** assumes an MTU of 1500 for containers regardless of what the host's actual outbound path supports. If a containerized app is the thing initiating traffic over your WireGuard tunnel, the container doesn't know the tunnel's real ceiling.
- **Consumer routers doing their own tunneling or QoS shaping** sometimes silently reduce effective MTU without exposing that fact anywhere in their UI.

## Diagnosing it

The reliable way to find the actual usable size is to stop guessing and binary-search it with ping, using the flag that disables fragmentation on your end so you get an honest failure:

```
ping -M do -s 1472 <remote-tunnel-ip>
```

Start near 1472 (1500 minus the 28 bytes of IP/ICMP headers) and work down until packets stop failing. Once you've found the largest packet that gets through cleanly, subtract the WireGuard overhead you're dealing with to land on a safe interface MTU. 1420 is a common safe default for a single layer of WireGuard over a standard 1500-byte link, but don't just copy that number if you're on PPPoE or stacking tunnels. Measure it.

`tcpdump` on either endpoint is useful for confirming what's happening: you'll see the large packets go out and nothing come back, or you'll catch ICMP "fragmentation needed" replies that never make it to the sending host because something in between is filtering ICMP.

## Fixing it

Set the MTU explicitly in your WireGuard config instead of relying on the default:

```
[Interface]
Address = 10.10.10.2/24
MTU = 1420
```

If you're running WireGuard as a router or gateway for other devices rather than just terminating it on a single host, MSS clamping is the more robust fix, since it corrects TCP's segment size at the point traffic enters the tunnel regardless of what MTU each client thinks it has:

```
iptables -t mangle -A FORWARD -o wg0 -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

Tailscale sidesteps a lot of this by defaulting its interface MTU to 1280, deliberately leaving headroom for extra encapsulation layers, which is part of why it tends to "just work" where a hand-rolled WireGuard config doesn't. If you've moved from Tailscale to raw `wg-quick` and hit this exact failure mode, that default is the thing you lost.
