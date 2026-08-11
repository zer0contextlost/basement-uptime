---
title: "Why Your Two-Node Proxmox Cluster Keeps Losing Quorum (and What a QDevice Actually Fixes)"
date: 2026-08-11
description: "Two-node Proxmox clusters lose quorum on reboot or network blips because of basic vote math. A QDevice fixes it without buying a third full node."
tags: ["proxmox", "clustering", "homelab"]
---

Two Proxmox nodes, one cluster, and everything is fine until one host reboots for a kernel update. Suddenly the other node's web UI goes read-only, VMs on it keep running but you can't migrate, start, or stop anything, and the cluster status shows "quorum: no". Nothing is actually broken. This is Proxmox's cluster stack doing exactly what it's designed to do, and it catches almost everyone who builds a two-node cluster for the first time.

## Why two nodes is the worst possible cluster size

Proxmox uses Corosync for cluster membership and pmxcfs (the `/etc/pve` cluster filesystem) for shared config. Both rely on quorum: a majority of "votes" in the cluster has to be online and talking to each other before the cluster will allow write operations. The default is one vote per node.

With two nodes, a majority means both. If either one drops offline, even for a routine reboot, the remaining node only has half the votes. Half is not a majority. The surviving node freezes writes to `/etc/pve` to avoid a split-brain scenario where two isolated halves of the cluster both think they're in charge and make conflicting changes.

This is the same reason RAID1 arrays don't magically know which disk is "right" after a split. Quorum exists to prevent two nodes from each independently deciding they're the sole authority and diverging. The problem is that with exactly two members, there's no way to break a tie without outside input.

Three nodes doesn't have this problem: lose one, and two remain, which is still a majority. That's why every Proxmox clustering guide recommends an odd number of nodes, and why "just add a third node" is the textbook answer. But a lot of homelabs only have hardware for two real hypervisors, and buying a third machine just to hold a vote is overkill.

## What a QDevice actually is

A QDevice (Corosync's `votequorum` external quorum device) is a lightweight vote-only participant. It runs `corosync-qnetd` on some separate box, doesn't join the cluster, doesn't run VMs, and doesn't need Proxmox installed at all. It just answers "yes I can see you" to whichever nodes ask, and its vote is what breaks the tie.

With a QDevice added, your two real nodes plus the QDevice add up to three votes. Lose one hypervisor to a reboot, and the surviving hypervisor plus the QDevice still form a majority (2 of 3). The cluster stays quorate and writable.

The QNet daemon is small enough to run on hardware you'd never trust with production VMs:

- A Raspberry Pi
- An old mini PC sitting around doing nothing
- A cheap always-on router box that can run a minimal Debian install
- A VM on a completely separate physical host from your two Proxmox nodes (defeats the purpose if it's a VM running on one of the two nodes itself)

The key requirement is that it needs to stay reachable independently of either Proxmox node's uptime, and it should be on a different failure domain than your two hosts. Putting it on the same UPS and same switch as node1 doesn't help you if that switch dies.

## Setting it up

Install the qnetd package on the separate device (adjust for whatever distro you're running there, this assumes Debian-based):

```bash
apt install corosync-qnetd
```

Then from any node already in the Proxmox cluster:

```bash
apt install corosync-qdevice
pvecm qdevice setup <qdevice-ip>
```

This handles the SSH key exchange and certificate setup between the cluster and the qnetd host automatically. Check it took effect with:

```bash
pvecm status
```

You should see a third vote listed under "Membership information", tied to the QDevice rather than a node name.

## The failure mode this doesn't fix

A QDevice protects you from losing one of two real nodes. It does not help if the QDevice itself goes offline while both real nodes are up, because two out of three votes is still a majority and the cluster keeps running fine on just the two nodes. Where it *does* still bite you: if the QDevice is down and then you lose a node too, you're back to one vote out of a possible three, and quorum is lost again.

So the QDevice needs its own reliability, not perfect reliability, but independence from whatever tends to take your two main nodes down together (same breaker, same switch, same ISP outage). A $10/month VPS reachable over a WireGuard tunnel works as well as a spare Pi sitting in a closet, as long as the tunnel itself doesn't depend on either Proxmox node routing traffic for it.
