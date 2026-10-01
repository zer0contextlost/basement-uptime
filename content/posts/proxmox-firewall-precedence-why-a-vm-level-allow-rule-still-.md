---
title: "Proxmox Firewall Precedence: Why a VM-Level Allow Rule Still Gets Dropped at the Datacenter Level"
date: 2026-10-01
description: "How Proxmox's Datacenter, Node, and VM firewall layers actually stack, and why an allow rule on a guest can still get silently dropped."
tags: ["proxmox", "networking", "firewall"]
---

I spent an embarrassing amount of time once trying to figure out why a single ACCEPT rule on a VM's firewall tab did nothing. The rule was correct. The source IP was correct. The port was correct. Traffic still got dropped, no log entry, nothing in the VM's own iptables chain to blame. The problem was two levels up, in a place I hadn't even opened in months.

Proxmox's firewall isn't one ruleset. It's three separate layers that get evaluated in sequence, and each one can kill a packet before it ever reaches the layer where you were looking.

## The three layers

- **Datacenter**: cluster-wide defaults, security groups, and the global input/output policy.
- **Node**: rules scoped to a specific hypervisor host, mostly relevant for management traffic to the Proxmox host itself.
- **VM/CT**: per-guest rules, the ones most people edit first because they're the most visible in the UI.

Each layer has its own enable/disable toggle and its own default policy (ACCEPT, DROP, or REJECT). A packet has to survive all three to get through. If Datacenter policy is DROP and nothing in the Datacenter ruleset or an applied security group explicitly permits the traffic, it's gone before your carefully written VM rule even gets consulted.

This is the part that trips people up: the layers aren't "most specific wins." They're closer to independent gates in series. A VM-level ACCEPT does not override a Datacenter-level DROP. It only matters if the packet already made it past Datacenter and Node.

## The checkbox that's easy to miss

Firewall rules on a VM or CT do nothing at all unless two separate switches are flipped:

1. The firewall is enabled on the Datacenter level (global on/off).
2. The firewall is enabled on that specific network interface in the VM/CT hardware config, not just in the Firewall tab of the guest.

That second one is the one people forget. You can write a perfect rule set in the guest's Firewall tab, and if the "Firewall" checkbox on the network device itself (under Hardware) is unticked, none of it is evaluated. The interface just passes traffic as if no firewall exists for it. I've seen people "fix" a firewall problem by disabling the whole thing out of frustration, not realizing it was never actually active in the first place.

## Security groups aren't optional context, they're the default policy

If you've set up a security group at the Datacenter level and applied it broadly (say, to a group of VMs or at the Datacenter ruleset itself), that security group's rules get evaluated as part of the Datacenter layer. A security group with no explicit ACCEPT for your traffic, combined with a DROP default policy, behaves exactly like a missing rule. The VM has no idea any of this happened. Its own ruleset never even gets reached.

Worth checking: `Datacenter > Firewall > Options` for the actual default input/output policy, and `Datacenter > Firewall > Security Groups` for anything applied cluster-wide that might be scoping things down before your VM rules run.

## A sane order of operations when debugging

When traffic isn't reaching a guest and the VM firewall rule looks right:

1. Confirm Datacenter firewall is enabled and check its default policy.
2. Check for security groups applied at the Datacenter or Node level.
3. Check Node-level rules if the traffic is meant for the Proxmox host itself, not the guest.
4. Confirm the network interface on the VM/CT has its own Firewall checkbox enabled.
5. Only then look at the guest's own rule order, since rules are evaluated top to bottom and an early DROP or REJECT short-circuits anything below it.

## Logging is your actual debugging tool here

Each firewall tab has a Log Level setting for its ruleset, separate from the rule itself. Turning on logging at the Datacenter level, even temporarily, will show you in `/var/log/pve-firewall.log` which layer actually dropped the packet. This is far faster than guessing. I don't leave it on permanently since it gets noisy fast, but for tracking down a specific blocked connection it tells you immediately which gate the packet never cleared, instead of you staring at a VM ruleset that was never the problem to begin with.
