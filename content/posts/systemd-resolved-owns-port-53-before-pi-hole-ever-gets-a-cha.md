---
title: "systemd-resolved Owns Port 53 Before Pi-hole Ever Gets a Chance"
date: 2026-09-03
description: "Why Pi-hole or AdGuard Home fails to bind port 53 on a fresh Debian/Ubuntu host, and the difference between disabling the stub listener and actually freeing the port."
tags: ["dns", "pi-hole", "networking", "self-hosting"]
---

You install Pi-hole or AdGuard Home on a fresh Debian or Ubuntu box, and it fails to bind port 53. Or worse, it starts fine, but every device on your network still resolves through some other DNS path and your Pi-hole query log stays nearly empty. Both symptoms trace back to the same service: `systemd-resolved`.

## What's actually listening on 53

`systemd-resolved` ships enabled by default on most current Debian and Ubuntu installs. It runs a local stub resolver bound to `127.0.0.53:53`, and `/etc/resolv.conf` gets symlinked to point at it. Every process on that machine that resolves a hostname goes through the stub, which then forwards upstream according to whatever `systemd-resolved` was configured with, usually DHCP-provided DNS or nothing useful at all.

Check what's holding the port before you touch anything:

```
sudo ss -tulpn | grep :53
```

If you see `systemd-resolve` in that output, that's your answer. Pi-hole's install script will sometimes detect this and offer to "fix" it for you, but it's worth understanding what that fix actually does, because a half-applied version of it is the most common cause of the second symptom above, where Pi-hole runs but nothing uses it.

## Disabling the stub listener isn't the same as freeing the port

The stub listener is controlled by a line in `/etc/systemd/resolved.conf`:

```ini
[Resolve]
DNSStubListener=no
```

Setting this and restarting the service stops `systemd-resolved` from binding `127.0.0.53:53`. That's necessary, but it's not sufficient. `/etc/resolv.conf` is still a symlink to `/run/systemd/resolve/stub-resolv.conf`, which still points at `127.0.0.53`. Disable the stub without fixing that symlink and every local process on the host, including Pi-hole's own FTL binary when it tries to resolve upstream, is handed a DNS server that no longer exists.

Point `resolv.conf` at something that will actually answer:

```
sudo rm /etc/resolv.conf
sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

That target file (`resolv.conf`, not `stub-resolv.conf`) is the one `systemd-resolved` keeps updated with the real upstream servers it learned from DHCP or your static config, bypassing the stub entirely. Restart both services after:

```
sudo systemctl restart systemd-resolved
sudo systemctl restart pihole-FTL
```

Then re-run the `ss` check. Port 53 should now show Pi-hole's FTL process, not `systemd-resolve`.

## The part people skip: your router's DHCP still hands out the old DNS

Getting Pi-hole bound to port 53 on the host only matters for that host. Every other device on your LAN, phones, laptops, smart TVs, still gets its DNS server from your router's DHCP lease, and that's almost certainly still pointing at the ISP resolver or whatever it defaulted to. Fixing this at the OS level and stopping there is why people report "Pi-hole is running but my query log is empty."

Two ways to actually route your network's DNS through it:

- Change the DNS option in your router's DHCP settings to the Pi-hole host's LAN IP. This is the cleaner option if your router supports setting DNS independently of the gateway.
- If your router insists on handing out its own IP as DNS no matter what you configure, set Pi-hole itself as the DHCP server and disable DHCP on the router. This also gives you hostnames in the Pi-hole log instead of bare IPs, which is worth doing anyway.

Either way, confirm from a client device, not the Pi-hole host itself:

```
nslookup example.com
```

and check that the "Server" line in the response is your Pi-hole's IP, not something else. Testing from the Pi-hole host proves nothing, since local resolution was the thing you just fixed.

## If you'd rather not fight systemd-resolved at all

Running Pi-hole or AdGuard Home in a container or a dedicated LXC sidesteps this whole problem, because the container gets its own network namespace and its own port 53, independent of whatever the Proxmox host or a Docker host is doing with `systemd-resolved`. The catch is the reverse one: make sure nothing on the container host itself needs DNS from that container before it's fully up, and that the container's own `resolv.conf` isn't quietly inheriting the host's broken stub setup through a bind mount or shared network namespace. Check `resolv.conf` inside the container separately; don't assume it matches the host just because the host is "fixed."
