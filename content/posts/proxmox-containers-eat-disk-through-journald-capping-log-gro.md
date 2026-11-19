---
title: "Proxmox Containers Eat Disk Through journald: Capping Log Growth Inside LXC"
date: 2026-11-19
description: "A small LXC rootfs can fill up from systemd-journald alone. Here is how to find the culprit, cap journal size, and stop it from happening again."
tags: ["proxmox", "lxc", "journald", "logging", "storage"]
---

A container with an 8 GB root disk was fine for months, then started throwing "No space left on device" during an apt upgrade. Nothing in it was big. A database that barely grew, a reverse proxy, a couple of small services. The disk was 100% full anyway.

The cause was `/var/log/journal`. It had quietly grown to several gigabytes.

## Why journald does this

When `/var/log/journal` exists, systemd-journald stores logs persistently. If the directory doesn't exist, it logs to `/run/log/journal` in RAM instead, and the data disappears on reboot. Many Debian-based LXC templates ship with the persistent directory present, or something creates it later, and from that moment the journal writes to your rootfs.

The default limits are percentage-based. Journald allows itself up to 10% of the filesystem for persistent logs, and it also tries to leave 15% free. On a big disk that is generous. On a small container rootfs it sounds fine, but the percentages are computed against the filesystem the journal lives on. If your rootfs is a thin-provisioned volume or a ZFS subvolume with a refquota, the numbers journald sees may not match the quota you set. On a ZFS-backed container with a large pool, some versions of this logic see far more space than the container can use. The cap then never kicks in before your quota does.

A chatty service makes it worse. One app logging a stack trace in a retry loop can write hundreds of megabytes in a day.

## Find out what is using the space

Start from inside the container:

```
journalctl --disk-usage
du -xh /var --max-depth=2 | sort -h | tail
```

`journalctl --disk-usage` reports archived and active journal files together. If that number is a large fraction of your rootfs, you have found it.

Then find who is producing it:

```
journalctl --since "1 hour ago" -o json-pretty --output-fields=_SYSTEMD_UNIT \
  | grep _SYSTEMD_UNIT | sort | uniq -c | sort -rn | head
```

One unit usually dominates. Fixing that service's log level is the real cure. The caps below are the safety net.

## Cap the journal

Don't edit `/etc/systemd/journald.conf` directly. Use a drop-in so package upgrades don't fight you:

```
mkdir -p /etc/systemd/journald.conf.d
cat > /etc/systemd/journald.conf.d/size.conf <<'EOF'
[Journal]
SystemMaxUse=200M
SystemKeepFree=500M
SystemMaxFileSize=25M
MaxRetentionSec=2week
EOF
systemctl restart systemd-journald
```

What each line does:

- `SystemMaxUse` is the hard ceiling for all persistent journal files combined.
- `SystemKeepFree` makes journald stop growing before the disk gets tight. Set it relative to your rootfs size, not copied blindly. On a 4 GB container, 500M is a big bite.
- `SystemMaxFileSize` controls the size of each file before rotation. Smaller files mean vacuuming can remove old data in finer steps.
- `MaxRetentionSec` drops entries older than the limit regardless of size.

Then reclaim space right now:

```
journalctl --vacuum-size=200M
```

Vacuuming only removes archived files, not the active one. If the number barely moves, rotate first with `journalctl --rotate` and vacuum again.

## Or skip persistence entirely

Do you ever read a container's logs after a reboot? For many small service containers the answer is no, because anything important is shipped elsewhere or the container is easy to rebuild. In that case, keep the journal in RAM:

```
[Journal]
Storage=volatile
RuntimeMaxUse=64M
```

Put that in the same drop-in, remove `/var/log/journal`, and restart journald. Logs live in tmpfs, capped at 64 MB. Memory used counts against the container's RAM limit, so keep the number small.

I use persistent storage on anything I might need to debug after a crash, and volatile on the throwaway ones. After a kernel panic on the host, you want the container's last words on disk.

## Check the other usual suspects

If the journal is clean and the disk is still full, look at these:

- `/var/log` with classic text logs from nginx, apache, or a database. Logrotate may be missing or may not be running, since cron and timers can be absent in minimal templates. Check `systemctl list-timers | grep logrotate`.
- Docker's JSON logs under `/var/lib/docker/containers/*/*-json.log` if you run Docker inside the container. Set `max-size` and `max-file` in `/etc/docker/daemon.json`.
- Deleted-but-open files. `lsof +L1` lists files that were removed while a process still holds them. The space stays used until the process restarts.

## Resizing as a stopgap

If the container is already wedged and apt can't even run, grow the disk from the host while it is running:

```
pct resize 101 rootfs +4G
```

That buys you room to do the cleanup above. Don't treat it as the fix. A runaway logger will eat the extra 4 GB too, just a few weeks later.

Pick the container with the smallest rootfs you own, run `journalctl --disk-usage` in it, and see how close it is to the edge.
