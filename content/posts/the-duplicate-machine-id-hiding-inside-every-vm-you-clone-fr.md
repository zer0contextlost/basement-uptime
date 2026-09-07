---
title: "The Duplicate Machine ID Hiding Inside Every VM You Clone From a Template"
date: 2026-09-07
description: "Cloning a Proxmox VM template can leave every child VM sharing the same /etc/machine-id and SSH host keys. Here's why that happens and how to fix it."
tags: ["proxmox", "cloud-init", "networking"]
---

Clone ten VMs from the same Proxmox template, boot them all, and check `cat /etc/machine-id` on each one. If you built that template by hand, hardened it, then converted it, there's a good chance every single VM reports the exact same ID. Same for the SSH host keys in `/etc/ssh/`. Nothing in the boot process complains about this. Everything appears to work. The problems show up later, and they're the kind that take a while to trace back to the template.

## Where machine-id actually gets used

`/etc/machine-id` is supposed to be unique per installation. It gets read by dbus, by journald (some log correlation tooling keys off it), and by systemd-networkd when it generates a DHCP client identifier for DHCPv6 or the DUID used for lease requests. That last one is the one that bites people on home networks.

When several VMs on the same subnet present the same DUID, your DHCP server can get confused about which lease belongs to which client. Symptoms are things like a VM losing its IP after another VM reboots, or a lease that keeps flipping between two MACs, or a router's client list showing fewer devices than you actually have running. None of this points obviously at machine-id. You end up staring at DHCP server logs wondering why two completely different machines are contesting the same lease.

## Why templates end up like this

A Proxmox template usually starts life as a normal VM. You install the OS, configure it, maybe run cloud-init once to test it, then convert the VM to a template and clone from there. The problem is that `/etc/machine-id` gets written the first time the OS boots, and once it exists, nothing regenerates it automatically. Cloning the disk just copies that file along with everything else. Every child VM inherits the exact same ID because, as far as the OS is concerned, it never had a reason to make a new one.

SSH host keys have the same problem for the same reason. They're generated once on first boot by default and then just sit there. Clone the disk, and every VM you spin up trusts a stranger's identity as its own, meaning any of them can silently answer for any of the others' host key fingerprint if something on your network ever gets confused about which host is which.

Cloud-init is supposed to handle this. Its whole job on first boot is to notice "this is a new instance" and regenerate machine-specific state. But it only does that reliably if it thinks it's actually looking at a new instance. If your template already ran cloud-init to completion before you converted it, the instance metadata in `/var/lib/cloud/instance` says "already initialized," and cloud-init happily skips all of that work on every clone, forever.

## Fixing VMs that already have this problem

For any VM currently running with a duplicated ID:

```
rm -f /etc/machine-id
systemd-machine-id-setup
reboot
```

Regenerate SSH host keys separately, since they don't follow machine-id automatically:

```
rm -f /etc/ssh/ssh_host_*
ssh-keygen -A
systemctl restart sshd
```

Do this on every VM that came from the same tainted template, not just the ones currently misbehaving. If the DHCP symptom hasn't shown up yet, that just means you haven't cloned enough VMs from that template yet to collide.

## Building a template that doesn't do this again

Before converting a VM to a template, clean out the state that's supposed to be instance-specific:

```
cloud-init clean --logs
rm -f /etc/machine-id
rm -f /etc/ssh/ssh_host_*
truncate -s 0 /etc/machine-id
```

Truncating to an empty file (rather than deleting it outright) matters on some distros, since systemd treats an empty `/etc/machine-id` as "please generate one on next boot" but treats a missing file differently depending on version. Check `man machine-id` for your distro's exact behavior before assuming either approach is safe.

Then convert to template and clone. Boot the first clone and confirm `/etc/machine-id` differs from the template's original value before you build a dozen more VMs on top of the same assumption.
