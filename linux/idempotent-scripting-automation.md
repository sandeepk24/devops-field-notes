# Writing Scripts That Don't Bite You Back: Idempotency in Everyday Automation

Every infrastructure team has a story about a script that ran twice by accident and caused real damage — a cleanup job that deleted more than it should have because it ran concurrently with itself, a database migration that half-applied because a retry fired mid-execution. The common thread in almost all of these stories isn't bad intent or carelessness. It's a script that assumed it would only ever run once, cleanly, in isolation. Production doesn't work that way, and the sooner that assumption gets dropped, the fewer 3 AM pages you get.

## What Idempotency Actually Buys You

The formal definition — running an operation multiple times produces the same result as running it once — sounds abstract until you see it in a cron job. Cron doesn't know or care if the previous run of a job finished. If a nightly cleanup script takes eleven minutes and the schedule fires it every ten, you now have two copies running against the same data simultaneously, and if the script wasn't written to handle that, you find out the hard way.

## Locking: The First Layer of Defense

The simplest fix is preventing overlapping runs in the first place, which is a smaller problem than making every operation fully idempotent, and worth doing even when you also do the harder work.

```python
import fcntl
import sys
import os

LOCK_FILE = "/var/run/cleanup_job.lock"

def acquire_lock():
    lock_fd = open(LOCK_FILE, "w")
    try:
        fcntl.flock(lock_fd, fcntl.LOCK_EX | fcntl.LOCK_NB)
    except BlockingIOError:
        print("Another instance is already running. Exiting.")
        sys.exit(1)
    lock_fd.write(str(os.getpid()))
    lock_fd.flush()
    return lock_fd

if __name__ == "__main__":
    lock = acquire_lock()
    try:
        run_cleanup()
    finally:
        os.remove(LOCK_FILE)
```

This is a small amount of code that quietly prevents an entire category of incident. At companies running thousands of scheduled jobs across a fleet, this pattern — or an equivalent using a distributed lock in something like Redis or a database row — is close to mandatory, not optional, because the cost of a double-run scales with the size of the fleet running it.

## Designing Operations to Be Naturally Idempotent

Locking prevents concurrent runs, but it doesn't help if the same job runs twice sequentially — once tonight, once again tomorrow because yesterday's run silently failed halfway through and nobody caught it. The more durable fix is designing the operation itself so re-running it is safe.

Compare these two approaches to a resource cleanup task:

```python
# Fragile: assumes a clean starting state every time
def cleanup_old_snapshots(snapshots):
    for snap in snapshots:
        delete_snapshot(snap.id)  # fails hard if already deleted

# Idempotent: tolerates partial completion from a previous run
def cleanup_old_snapshots(snapshots):
    for snap in snapshots:
        try:
            delete_snapshot(snap.id)
        except SnapshotNotFoundError:
            # Already gone — from a previous partial run, and that's fine
            continue
```

The second version isn't more clever, it's just more honest about the environment it actually runs in — one where network blips, timeouts, and partial failures are normal, not exceptional. That mindset shift, from "this will complete cleanly" to "this needs to survive being interrupted anywhere," is most of what separates automation that's trusted from automation that gets someone paged.

## State Checks Before Action, Every Time

A pattern worth making into a habit: before performing any mutating action, check whether the desired end state already exists.

```python
def ensure_iam_role_exists(role_name, policy_document):
    existing = iam_client.list_roles()
    if any(r["RoleName"] == role_name for r in existing["Roles"]):
        print(f"{role_name} already exists, skipping creation.")
        return
    iam_client.create_role(RoleName=role_name, AssumeRolePolicyDocument=policy_document)
    print(f"Created {role_name}.")
```

This looks almost too simple to mention, but it's exactly the pattern that infrastructure-as-code tools like Terraform and CloudFormation are built around at a much larger scale — reconcile actual state against desired state, and only act on the difference. Writing your own scripts with the same instinct, even the small ones, tends to make them a lot more forgiving of the messy conditions they actually run under.

## Logging Enough to Reconstruct What Happened

The last piece, and the one that's easiest to skip when you're writing something "quick," is logging enough that a future incident review doesn't turn into guesswork.

```python
import logging
import json
from datetime import datetime

logging.basicConfig(
    filename="/var/log/cleanup_job.log",
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(message)s"
)

def run_cleanup():
    started = datetime.utcnow()
    logging.info(json.dumps({"event": "job_start", "started_at": started.isoformat()}))
    # ... do the work ...
    logging.info(json.dumps({"event": "job_complete", "duration_s": (datetime.utcnow() - started).total_seconds()}))
```

Structured logging like this costs almost nothing to add and pays off enormously the one time someone needs to answer "did this job actually run last Tuesday, and what did it do" during a postmortem, months after anyone remembers writing it.

## The Underlying Habit

None of these patterns are individually complicated. What matters is applying them by default, before there's a reason to, rather than retrofitting them onto a script after it's already caused an incident. The scripts that quietly run for years without anyone thinking about them are almost never the cleverest ones — they're the ones written with the assumption that they'll eventually run twice, run late, or run into a half-finished version of themselves, and handle it without complaint.
