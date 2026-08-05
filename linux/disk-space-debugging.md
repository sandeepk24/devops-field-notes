# "No Space Left on Device": Chasing Down Disk Usage Before It Pages You

There's a specific kind of dread to a disk-full alert, mostly because the fix is usually simple — delete something — but finding *what* to delete on a box you don't know intimately can turn into twenty minutes of increasingly frustrated `du` commands. This is about doing that faster, and about the handful of disk space traps that look like a mystery until you know they exist.

## df Tells You There's a Problem, du Tells You Where

`df` and `du` get confused for doing the same job, but they answer different questions. `df` reports space at the filesystem level — how full is this mount point. `du` reports space at the directory level — what's actually taking up that room. You need both, in that order: `df` to confirm and locate the problem, `du` to find the culprit.

```bash
# Which filesystem is actually full?
df -h

# Now drill into it — top-level directories on that mount, sorted by size
du -sh /var/* 2>/dev/null | sort -rh | head -10
```

That `2>/dev/null` matters more than it looks — without it, any directory you don't have permission to read spits a permission-denied error into your output and clutters what should be a clean list. On a box with a lot of directories you can't fully traverse, that noise adds up fast and makes the real signal harder to spot.

Once you've found the fat top-level directory, the same pattern recurses one level deeper:

```bash
du -sh /var/log/* 2>/dev/null | sort -rh | head -10
```

Repeating that a few levels down usually gets you straight to the actual offender — often a single log file or a directory of them that never got rotated.

## The Classic Trap: df and du Disagreeing

Every so often you'll hit a situation where `df` says a filesystem is 95% full, but `du` on the whole mount only adds up to a fraction of that. This isn't a bug, and it isn't `du` lying to you — it's almost always a deleted-but-still-open file. On Linux, deleting a file just unlinks its name from the directory; if a running process still has that file open, the disk space it occupies isn't actually freed until the process closes it or exits.

```bash
# Find processes holding open file handles to files that have been deleted
lsof +L1
```

`+L1` filters specifically for files with a link count under 1 — meaning they've been unlinked but something still has them open. This is an extremely common cause of "the disk is full but I can't find what's using it," especially with long-running services that log to a file, get their logs rotated by an external tool, but never actually close and reopen the file handle, so they keep writing into what is now, from the filesystem's perspective, invisible space.

```bash
# Once you've found the PID, you generally have two options:
# 1. Restart the process cleanly, which releases the handle
# 2. If you can't restart it yet, truncate the still-open file in place
: > /proc/12345/fd/7
```

That second option is a genuine emergency-only move — it doesn't delete anything new, it just zeroes out the content of a file descriptor that's already been unlinked, buying you space back immediately without touching the running process. It's not something to reach for casually, but it's saved more than a few production hosts from filling up completely while someone figures out the real fix.

## Log Rotation Gone Wrong

A huge share of disk-full incidents trace back to log rotation either not running, or running against the wrong path after a config change. It's worth knowing how to sanity check this quickly rather than assuming it's configured correctly just because it was at some point.

```bash
# Is logrotate actually running, and when did it last succeed?
cat /var/lib/logrotate/status

# Dry run against the config to see what it would do, without touching anything
logrotate -d /etc/logrotate.d/my-app
```

The `-d` dry-run flag is worth defaulting to whenever you're debugging a rotation config, rather than just running it live and hoping — it prints exactly what it would rotate and why, or why it's skipping a file, which is usually enough to spot a size or age threshold that's set wrong.

## inodes: The Disk Space Problem That Isn't About Space

Every so often `df -h` shows plenty of free space, and you're still getting "no space left on device" errors when trying to create new files. This is almost always an inode exhaustion problem, not a disk space one — every file, no matter how small, consumes one inode, and a filesystem has a fixed number of them set at creation time. A directory full of millions of tiny files, which is exactly what you get from a misbehaving cache or a session-file directory that never gets cleaned up, can exhaust inodes while barely denting actual disk usage.

```bash
df -i
```

If `IUse%` is near 100% while your regular `df -h` shows plenty of headroom, that's the tell. The fix isn't freeing disk space, it's finding and cleaning up whatever's generating an enormous number of small files — and it's a genuinely confusing failure mode the first time you hit it, because every instinct says "check the disk," and the disk looks fine.

## A Quick Habit for Preventing the Page Entirely

Most of this is reactive — figuring out what's wrong after an alert fires. The cheaper version is catching the trend before it becomes an incident, which is usually a five-line cron job away.

```python
import shutil
import subprocess

def check_disk_usage(threshold_pct=85):
    usage = shutil.disk_usage("/")
    used_pct = (usage.used / usage.total) * 100
    if used_pct >= threshold_pct:
        top_dirs = subprocess.run(
            ["du", "-sh", "/var/log", "/var/lib", "/tmp"],
            capture_output=True, text=True
        ).stdout
        print(f"WARNING: disk at {used_pct:.1f}% used\n{top_dirs}")
        # send to Slack / PagerDuty / wherever your alerts go

if __name__ == "__main__":
    check_disk_usage()
```

Nothing sophisticated, but running this on a schedule and alerting at 85% instead of finding out at 100% is the difference between a calm cleanup task and an actual incident where services are failing to write and someone's paged at an inconvenient hour.

## The Underlying Lesson

Disk space problems have a reputation for being simple, and mostly they are — except for the two traps that aren't: a deleted file still held open by a running process, and inode exhaustion masquerading as a space problem when there's plenty of space left. Knowing those two exist, and knowing `lsof +L1` and `df -i` as the specific commands that reveal them, turns a confusing half hour into a two-minute diagnosis the next time either one shows up.
