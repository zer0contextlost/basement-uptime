---
title: "Your Backup Job Says 'Success.' That Doesn't Mean the Data Is Restorable."
date: 2026-08-17
description: "A backup that reports success can still be silently useless. Here's how to build a restore test you actually run, using restic as the example."
tags: ["backups", "restic", "homelab"]
---

Every backup tool you'll run at home reports success the same way: exit code 0, a log line that says something like "snapshot complete," maybe a summary of files and bytes. None of that tells you the data can be put back. It tells you the write phase finished without an I/O error. Those are different claims, and conflating them is how people find out their three years of photos are gone at the exact moment they need them.

## What actually goes wrong

A few failure modes that produce a green checkmark and a useless archive:

- **Partial snapshots that still exit clean.** A source directory gets unmounted mid-run, or a container stops halfway through, and the backup tool happily archives whatever's left, reports success, and never flags that half the tree is missing.
- **Repository corruption that isn't caught until read time.** Bit rot on the backup target, a botched disk replacement, an interrupted prune, a repository that was fine last month and silently isn't now. Write-time checks don't catch this because the corruption happens after the write.
- **Permissions or ownership that don't survive the round trip.** You restore a directory and it comes back owned by the backup user instead of the original service account, and whatever consumed that data refuses to start.
- **Encryption keys or passwords that only live in one place.** The backup itself is intact, but the thing needed to open it was stored on the same host that just died.
- **Excludes that quietly grew too aggressive.** Someone adds a `.dockerignore`-style exclude pattern to speed up a snapshot, and six months later half your compose stacks aren't in the archive at all.

None of these show up in a job log unless you go looking specifically for them.

## `restic check` is necessary, not sufficient

If you're using restic, `restic check` verifies repository structure and, with `--read-data`, actually reads back the pack files and checks them against their hashes. Run it. It's the single best automated defense against silent repository corruption, and most people never enable it because it's slow on large repos.

But `restic check` proves the repository is internally consistent. It doesn't prove that what's inside it is what you think is inside it, that it restores with the right permissions, or that the thing you'd actually need in a disaster (a database dump, a config directory, a container's persistent volume) unpacks into something usable. A repository can pass `check` completely and still be missing the one file you needed because an exclude pattern silently swallowed it eight months ago.

## Build a restore test, not just a backup test

The only way to know a backup works is to restore it somewhere and look. That doesn't need to be a full disaster recovery drill every week. It needs to be small, automated, and boring enough that it actually keeps running.

A minimal version:

1. Spin up a scratch LXC or VM with no persistent state, or reuse a throwaway container that gets wiped after each run.
2. Pull the latest snapshot for one specific, meaningful path, not the whole repository. Pick something that would hurt if it silently rotted: a database volume, a config directory, whatever your household actually depends on.
3. Restore it into the scratch environment.
4. Check something concrete about the restored data, not just that files exist. For a database, that means starting it and running a query. For a config tree, that means diffing file counts and checksums against a known-good manifest, or actually starting the service against the restored config.
5. Tear the scratch environment down and log the result somewhere you'll notice if it stops updating.

The point of tearing it down each time is that a scratch environment that accumulates state stops being a clean test. If it still has last week's restored data sitting around, a broken restore this week can look identical to a working one.

Wire this into a systemd timer or cron job separate from the backup job itself, so a broken restore test doesn't silently depend on the same script that might also be broken. Have it write a timestamp to a file, or ping a dead-man's-switch style monitoring check, so a restore test that stops running is itself an alert, not silence.

## What to actually verify

Don't just check that a file exists after restore. Check that it's the right size, that a checksum matches a manifest you keep separately from the backup repo, and for anything database-backed, that the service starts against the restored data and answers a query. A config file that restores as zero bytes still "exists."

Keep the manifest and the restore test script outside the backup target itself. If the only copy of your verification process lives in the thing you're trying to verify, you've built a single point of failure into your safety net.

The failure mode worth designing against isn't the backup that fails loudly. It's the one that's been quietly wrong for four months and nobody noticed because nobody looked.
