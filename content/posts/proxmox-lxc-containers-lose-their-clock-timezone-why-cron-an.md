---
title: "Proxmox LXC Containers Lose Their Clock Timezone: Why Cron and Logs Run in UTC After a Restore"
date: 2026-11-16
description: "Containers restored or created on Proxmox often log in UTC while the host runs local time. How the timezone setting works and how to fix cron jobs that fire at the wrong hour."
tags: ["proxmox", "lxc", "timezone", "cron"]
---

My nightly backup script inside a container was supposed to run at 02:00. It ran at 21:00 the previous evening, right in the middle of someone streaming a movie. The host clock was correct. The container clock was correct too, as far as `date` was concerned. It just said UTC, and I had never told it otherwise.

## What a container actually inherits

An LXC container shares the host kernel, so it shares the host's clock. Time itself is never wrong inside a container. The timezone is a different matter. It comes from userspace, specifically the `/etc/localtime` symlink and `/etc/timezone` inside the container's root filesystem.

Proxmox has a per-container option for this. In the GUI it is under Options, Time zone. In the config file it looks like this:

```
# /etc/pve/lxc/101.conf
timezone: host
```

The accepted values are `host`, which copies the host's timezone at start, or a name such as `Europe/Berlin`. If the line is missing, the container keeps whatever its template shipped with. Most Debian and Ubuntu templates ship with UTC.

## Check what you have

Run this inside the container:

```
timedatectl
date
readlink -f /etc/localtime
cat /etc/timezone
```

Inside a minimal container `timedatectl` sometimes complains that it can't talk to systemd. That's fine. The `readlink` output is the one that matters. If it points at `/usr/share/zoneinfo/Etc/UTC`, you are in UTC.

On the host, compare with:

```
timedatectl | grep "Time zone"
pct config 101 | grep timezone
```

If `pct config` prints nothing for timezone, the setting was never applied.

## Setting it properly

From the host:

```
pct set 101 --timezone host
```

or for a fixed zone:

```
pct set 101 --timezone America/Chicago
pct reboot 101
```

Proxmox writes the symlink during container startup, so a reboot of the container is needed. Restarting only the services inside it does nothing.

One catch: the `tzdata` package has to be installed inside the container. Some slim images leave it out. Without it, there is nothing in `/usr/share/zoneinfo` for the symlink to point at, and the container quietly falls back to UTC. Fix it with:

```
apt install tzdata
```

Then reboot the container again.

## Why restores and clones bite

A restore from a vzdump archive brings back the config, including the `timezone:` line, so that usually survives. The trouble starts when you build a container from a template with the hand-typed `pct create` command, or when you copy a root filesystem in from another source. Neither of those carries a timezone line, and you get the template default.

Another source of surprise is a container that was migrated from a different host. If it used `timezone: host` and the second host is configured for a different zone, the container follows the new host. That is usually what you want, but only if you remember it.

## Cron is the part that hurts

Cron reads the system timezone when the daemon starts. If you change the timezone while cron is running, existing schedules keep using the old one until you restart it:

```
systemctl restart cron
```

A reboot of the container covers this too, which is another reason to prefer the reboot.

Systemd timers are less fragile, since they take `OnCalendar=` expressions that can name a zone explicitly:

```
OnCalendar=*-*-* 02:00:00 America/Chicago
```

Pinning the zone in the timer makes the schedule independent of whatever the container thinks it is. I now do this for anything that must not move, like backup windows that need to stay clear of the host's own vzdump jobs.

## Logs and the daylight saving problem

Mixed timezones make log correlation miserable. If the host logs in local time and three containers log in UTC, matching a failed backup to the service that stalled becomes arithmetic. For a home setup I set everything to `host` and stop thinking about it.

The alternative is to run everything in UTC deliberately, which is a sound choice for servers. UTC has no daylight saving jump, so a 02:30 job never gets skipped or run twice on the clock change nights. If you go that way, convert your cron expressions once and write the intended local time in a comment next to each one.

Whichever you pick, do it on purpose and apply it to every guest. VMs have the same issue, and there the setting lives inside the guest OS, so `timedatectl set-timezone` is the tool, not anything in the Proxmox config.

## A quick audit

This loop lists the setting for every container on a node:

```
for id in $(pct list | awk 'NR>1 {print $1}'); do
  echo -n "$id: "
  pct config $id | grep -E '^timezone' || echo "not set"
done
```

Anything printing "not set" is running on its template default. Mine had six of them.
