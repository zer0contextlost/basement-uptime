---
title: "Split-Horizon DNS: Why Your Reverse Proxy Setup Needs Two Different Answers for the Same Hostname"
date: 2026-09-10
description: "How to make service.example.com resolve to a different address inside your LAN than it does on the public internet, and why skipping this causes broken NAT hairpinning and cert mismatches."
tags: ["dns", "networking", "reverse-proxy", "homelab"]
---

You set up a reverse proxy, pointed a real domain at it, got a valid Let's Encrypt cert, and everything works great from your phone on cellular data. Then you try the same URL from a laptop sitting three feet from the server, on the same LAN, and it either times out or takes a slow detour out to the internet and back through your router. That's a NAT hairpinning problem, and the fix is split-horizon DNS.

## What's actually happening

When you request `service.example.com` from inside your network, the query resolves to your router's public IP, because that's what the A record says. Your traffic then leaves your LAN, hits your router's WAN interface, and your router has to NAT that connection back to the internal server. Plenty of consumer routers don't support this reflection at all, so the connection just dies. Even when it works, you're routing local traffic through your ISP's edge and back for no reason.

The fix isn't changing the public DNS record. It's giving your internal resolver a different, more specific answer for the same name.

## Setting it up with a local resolver

If you're already running Pi-hole, Unbound, or dnsmasq for LAN DNS, you add a local override for each hostname you reverse-proxy:

```
# Unbound local-data example
local-data: "service.example.com A 192.168.1.10"
```

In Pi-hole this is just a Local DNS Record entry. In dnsmasq, an `address=/service.example.com/192.168.1.10` line in a conf file does it.

Point it at your reverse proxy's internal IP, not the backend container's IP. The proxy still needs to receive the request with the correct hostname so it can route based on SNI or the Host header, and so it can present the right cert.

Once that's in place, internal DNS queries for `service.example.com` return the LAN IP directly. Public queries (from your registrar or DNS provider) still return your public IP. Same name, two different answers depending on which side of the fence you're asking from. That's the "split horizon" part.

## Certificates still have to match

A common mistake once this is working: someone notices the internal traffic never actually leaves the LAN, so they think the public cert doesn't matter anymore and switch the internal path to plain HTTP or a self-signed cert. Don't do this. Your reverse proxy is still terminating TLS for `service.example.com`, and browsers are still checking that cert against the hostname in the URL bar, regardless of whether the IP behind it is public or private. Keep using the same Let's Encrypt (or internal CA) cert for both paths. The DNS split changes routing, not identity.

If you're issuing certs via DNS-01 challenge, this whole setup barely changes your cert renewal process. HTTP-01 challenges require your ACME client to be reachable from the public internet at the time of renewal, so make sure that path still works even if all your day-to-day traffic goes over the internal route.

## Watch for double-NAT and CGNAT edges

If your ISP puts you behind CGNAT, hairpinning was probably never going to work anyway, since your "public" router IP isn't actually the internet-facing one. Split-horizon DNS sidesteps that entirely, since internal clients never touch NAT at all for these requests. It's arguably the more correct fix even if hairpinning does happen to work for you, because it removes a router feature dependency from your day-to-day traffic path.

## Don't forget the container/VM's own resolver

If your reverse proxy or backend services run in LXC containers or VMs that use their own resolv.conf instead of querying your LAN resolver, they can end up resolving hostnames differently than your client machines do. Health checks or internal service-to-service calls that use the public hostname will hit the same hairpin problem from the inside. Point every container and VM at the internal resolver that has the split-horizon records, not at a public resolver like 1.1.1.1 or 8.8.8.8, or you'll be debugging this same symptom one layer deeper.
