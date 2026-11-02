---
title: "Proxmox LXC Containers Lose Their IPv6 Address or Get the Wrong One: SLAAC, Privacy Extensions, and Static Leases"
date: 2026-11-02
description: "Why LXC containers on Proxmox get changing IPv6 addresses, how SLAAC and privacy extensions cause it, and how to pin addresses so firewall rules and DNS stay valid."
tags: ["proxmox", "lxc", "ipv6", "networking", "dns"]
---

You set a container to DHCP, give it a static IPv4 reservation, point a DNS record at it, and move on. A few weeks later something works over IPv4 and fails over IPv6, or an AAAA record points at an address the container no longer uses. Nothing is broken in the usual sense. IPv6 just doesn't behave like IPv4, and the defaults in a Proxmox LXC make it easy to get surprised.

## Why the address keeps changing

If you select "SLAAC" for IPv6 in the container's network settings (or `ip6=auto` in the config), the container builds its own address from the prefix your router advertises. It doesn't ask anyone. There is no lease to reserve.

Two things then move underneath you:

- **The prefix.** Most residential ISPs hand out a delegated prefix that can change when the router reboots, the WAN link renegotiates, or the ISP decides it's Tuesday. Every container address changes with it.
- **The interface identifier.** Depending on the distro and sysctl settings, the lower 64 bits are either derived from the MAC, a stable hash, or a random value that rotates.

Run this inside the container and look at the flags:

```
ip -6 addr show dev eth0
```

An address marked `temporary` is a privacy-extension address. It's meant for outbound connections and it expires. An address marked `mngtmpaddr` is the stable one that generates the temporaries. A container can hold several global addresses at once, and which one it uses as a source address depends on selection rules most people never read.

## The symptom that sends you debugging the wrong thing

The classic version: you allow `2001:db8:1234:1::abcd` through a firewall or app allowlist, and the container's traffic gets blocked anyway. The container is connecting from its temporary address, not the one you listed. Check what it actually uses:

```
ip -6 route get 2001:4860:4860::8888
```

The `src` field in the output tells you the source address the kernel picked. If that's a temporary address, your allowlist entry is aimed at the wrong one.

Servers should not be using rotating source addresses. Turn privacy extensions off inside the container:

```
sysctl -w net.ipv6.conf.eth0.use_tempaddr=0
sysctl -w net.ipv6.conf.default.use_tempaddr=0
```

Put the same keys in a file under `/etc/sysctl.d/` so they survive a reboot. Existing temporary addresses linger until they expire, so either wait or bounce the interface.

## Pick a strategy: three workable options

**Stateful DHCPv6 with reservations.** If your router or DHCP server supports DHCPv6 with reservations by DUID, you get the IPv4-style experience. The catch is that DUIDs are not MACs. Many containers generate one from a machine ID, so a container rebuilt from a template gets a new identity. Clone-related machine ID problems apply here too. Also, Android clients ignore DHCPv6 entirely, so you usually still run SLAAC alongside it.

**Static addresses inside the prefix.** Set `ip6=2001:db8:1234:1::50/64,gw6=fe80::1` on the container, or use the GUI fields. Simple and predictable, but it hardcodes the prefix. If your ISP prefix is not stable, this breaks the day the prefix rotates. The gateway can be the router's link-local address, which doesn't change when the prefix does.

**Unique local addresses (ULA) for internal traffic.** Give your LAN a `fd00::/8` range (pick a random 40-bit global ID, don't use `fd00::/64` verbatim) and assign containers static ULA addresses. Internal DNS and firewall rules reference those, and they never change regardless of what the ISP does. The container can still have a SLAAC global address for outbound internet access. This is what I use at home, because internal records stay valid through any ISP prefix change.

## Don't forget the Proxmox firewall

If the Proxmox firewall is enabled on the container's NIC, there is a per-interface option for IP filter and an IPSet that restricts which source addresses the guest may use. A set that only lists the IPv4 address will drop every IPv6 packet the guest sends, and the failure looks like "IPv6 is just slow to come up." Check the container's Firewall tab for IPSet entries, and look at the firewall log for drops on the tap/veth interface.

## DNS that doesn't lie to you

If a container has both an A and an AAAA record, clients prefer IPv6 when it's reachable. A stale AAAA record causes long timeouts before fallback to IPv4, which people misdiagnose as a slow app. Two rules help:

- Only publish AAAA records for addresses that are static (ULA or a truly fixed assignment).
- When debugging, test each family explicitly: `curl -4 https://host.example.com` and `curl -6 https://host.example.com`. If one hangs and the other doesn't, you've narrowed it to a family problem within seconds.

## Quick checklist for a new container

1. Decide whether it needs IPv6 at all. If nothing consumes it, set `ip6=manual` and disable it with `net.ipv6.conf.all.disable_ipv6=1`.
2. If it does, give it a ULA static address for internal use.
3. Disable temporary addresses on anything that serves traffic.
4. Add the IPv6 address to any IPSet and allowlist that mentions the IPv4 one.
5. Publish AAAA records only for the stable address.

The next time a service mysteriously works from one machine and not another, run `ip -6 route get` on the client side before touching anything else. The source address it prints is usually the whole story.
