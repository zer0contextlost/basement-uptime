---
title: "Proxmox LXC Containers Can't Resolve Hostnames After Enabling the Datacenter Firewall: The DHCP and DNS Rules You Forgot"
date: 2026-11-09
description: "Turning on the Proxmox firewall can break DHCP and DNS for guests with no obvious error. Here is how to find the dropped packets and write the rules."
tags: ["proxmox", "firewall", "dns", "dhcp", "lxc"]
---

You flip the firewall switch at the datacenter level, everything you already had running keeps working for a few minutes, and then a container renews its lease and goes quiet. Or you reboot a VM and it comes up with no address. Nothing in the guest log says "blocked." It just can't talk.

The cause is the default policy. Once the datacenter firewall is enabled, input to the host is dropped unless a rule or a built-in exception allows it. Guest traffic is a separate matter, handled by the per-guest firewall, and that's where the trouble usually starts.

## What the defaults actually do

With the datacenter firewall on, the host itself still allows a small set of things: the web UI on 8006 and SSH from the local network, plus cluster traffic. Everything else inbound to the host is dropped.

Guests behave differently. If you enable the firewall on a VM or container's network device (the checkbox on the NIC itself, easy to miss), the guest gets its own input and output policies. The default input policy is DROP and the default output policy is ACCEPT. So a guest with the firewall checkbox ticked and no rules will reach the internet outbound just fine, but nothing will reach it, including replies that don't match connection tracking.

Two things make this confusing:

- The datacenter firewall switch and the per-NIC firewall checkbox are independent. You can have one without the other.
- Existing connections survive because of conntrack. The failure only shows up when something new starts, like a DHCP renewal or a fresh boot.

## Why DHCP breaks specifically

A DHCP client broadcasts from 0.0.0.0 port 68 to 255.255.255.255 port 67. The server replies to port 68. If the DHCP server is your router or a separate box, the reply arrives as inbound traffic to the guest, and it doesn't match an established connection in the way a normal TCP reply would. With input set to DROP, the offer never arrives.

Proxmox ships a macro for this. In the guest firewall, add an inbound rule:

```
IN DHCPfwd(ACCEPT) -i net0
```

In the GUI, that is Direction `in`, Action `ACCEPT`, Macro `DHCPfwd`. There is also an option on the NIC firewall settings called "DHCP" that you can enable, which handles client-side DHCP for that guest. Check that it is on. If your guests use static addresses, you can skip all of this, but then you will hit the next problem.

## DNS replies and the "pinhole" you need

Outbound DNS is allowed by the default output policy, and replies are matched by conntrack, so plain DNS from a guest to your resolver usually works. It breaks in these cases:

- You tightened the guest output policy to DROP and forgot to allow DNS out.
- Your resolver is another guest on the same bridge, and that guest has its own input DROP policy with no rule for port 53.
- Large responses or DNSSEC fall back to TCP on 53, and you only allowed UDP.

If you run Pi-hole or another resolver in an LXC, that container needs an inbound rule from your LAN:

```
IN ACCEPT -source 192.168.1.0/24 -p udp -dport 53
IN ACCEPT -source 192.168.1.0/24 -p tcp -dport 53
```

Allow both protocols. Skipping TCP produces the maddening symptom where small lookups work and a few odd ones time out.

## Finding what is being dropped

Guessing is slow. Turn on logging instead. In the guest firewall options, set the log level for input to `info`. Then watch the host:

```
journalctl -f -k | grep -i drop
```

Or read the firewall log file, which on current versions lives under `/var/log/pve-firewall.log`:

```
tail -f /var/log/pve-firewall.log
```

Each line names the chain (like `PVEFW-DROP` or a guest-specific one), the source, the destination, the protocol and the port. A line with `SPT=67 DPT=68` is your DHCP reply getting eaten. A line with `DPT=53` is a resolver with no inbound rule. Trigger the failure on purpose, run `dhclient -r && dhclient` or `dig example.com @192.168.1.53` from the guest, and read the matching entry.

Remember to turn log levels back to `nolog` afterward. Dropped broadcast noise on a busy LAN fills the log faster than you'd expect.

## A safer way to turn it on

Do this in order so you do not lock yourself out or break a running service:

1. Before enabling anything, write the datacenter-level rules you need for management access (8006 and 22 from your admin subnet). Do not rely on the built-in exceptions alone.
2. Build an IP set or a security group for the common allowances: DHCP, DNS, ICMP. Attach that group to each guest rather than retyping rules.
3. Enable the firewall on one low-stakes guest first. Reboot it. Confirm it gets a lease and resolves names.
4. Only then enable the datacenter switch and roll through the rest.

The reboot in step 3 matters. A guest that has been up for weeks tells you nothing, because conntrack entries and long leases mask the missing rules.

## ICMP is worth the same attention

While you are in there, allow ping inbound from your LAN. Without it, your monitoring thinks the guest is down and neighbor checks can get strange. The `Ping` macro covers it, and one rule in a shared security group fixes it everywhere.

The next time a container loses its lease right after a firewall change, check the NIC checkbox and `pve-firewall.log` before you touch the guest's network config. The answer is almost always sitting in that log as a single line with port 67 or 53 in it.
