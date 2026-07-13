# grep, awk, and sed: The Log Parsing Trio Nobody Retires

There's a running joke on most infra teams that no matter how sophisticated your observability stack gets — Datadog, Grafana, ELK, whatever your company's paying six figures a year for — the first thing anyone actually does during an incident is SSH into a box and run `grep`. It's not because the fancy tooling doesn't work. It's because when production is on fire, you want raw text on a raw terminal with zero UI lag between you and the answer.

This isn't a "here's what grep does" primer. You know what grep does. This is about the patterns that actually get used when someone pages you at 2 AM because error rates spiked.

## The First Move: Narrowing the Time Window

Nobody greps an entire log file during an incident. The first move is almost always narrowing to a time window, because a service handling meaningful traffic can produce gigabytes of logs per hour, and scrolling through all of it is how you miss the actual signal.

```bash
# Errors in the last 10 minutes, roughly
awk -v since="$(date -d '10 minutes ago' '+%Y-%m-%dT%H:%M:%S')" \
  '$1 >= since' app.log | grep -i error
```

At a large-scale service, this is the kind of one-liner you type without thinking, because the alternative — opening a dashboard, picking a time range with a mouse, waiting for the query to run — costs you thirty seconds you don't have when you're trying to correlate a deploy with a spike.

## Counting Before You Read

Before reading individual log lines, the more useful question is usually "how many, and how is that trending." This is where `awk` earns its keep, because it can aggregate inline without you needing to pull the data into anything else.

```bash
# Count errors per minute, sorted
grep "ERROR" app.log | awk '{print substr($1,1,16)}' | sort | uniq -c
```

That single line turns a wall of text into a shape — you can see at a glance whether errors are climbing, flat, or already recovering. On a team running dozens of services, this is often the difference between "this is getting worse, page more people" and "this already peaked, we're fine to keep investigating calmly."

## sed for Surgical Edits, Not Just Substitution

Most people learn `sed` as "find and replace" and stop there. The more useful daily habit is using it to strip noise so the signal is easier to read. Structured JSON logs are great for machines and genuinely painful for a human eyeballing a terminal during an incident.

```bash
# Pull just the message field out of JSON log lines, ignore the rest
sed -n 's/.*"message":"\([^"]*\)".*/\1/p' app.log | tail -50
```

It's not elegant — a proper JSON parser would be more correct — but it's fast, it needs no dependencies, and when you're on a bastion host with nothing but coreutils installed, fast and dependency-free wins every time.

## A Python Alternative When the Line Gets Blurry

There's a point where chaining five pipes together stops being clever and starts being a liability, especially if someone else needs to read your one-liner during a postmortem. That's usually the signal to drop into a short Python script instead — still fast, but readable six months later.

```python
import re
from collections import Counter
from datetime import datetime, timedelta

cutoff = datetime.utcnow() - timedelta(minutes=10)
error_counts = Counter()

with open("app.log") as f:
    for line in f:
        match = re.match(r"^(\S+) .*ERROR.*service=(\w+)", line)
        if not match:
            continue
        timestamp = datetime.fromisoformat(match.group(1))
        if timestamp >= cutoff:
            error_counts[match.group(2)] += 1

for service, count in error_counts.most_common(5):
    print(f"{service}: {count} errors")
```

This is the kind of script that ends up living in a team's internal toolbox, run so often during incidents that someone eventually wraps it into a Slack bot command. That's usually how internal tooling gets born — not from a roadmap, but from someone getting tired of typing the same grep chain for the third incident in a row.

## Why This Still Matters in a World of Managed Log Platforms

It's tempting to think this is legacy knowledge, made obsolete by centralized logging platforms with powerful query languages. In practice, those platforms still ingest raw text, and understanding how to slice that text by hand is what lets you write a good query in the first place — and what saves you when the platform itself is the thing that's degraded, which does happen, more often than any vendor would like to admit.

The engineers who move fastest during incidents aren't the ones with the fanciest dashboards. They're the ones who can look at a wall of raw text and immediately know which three commands will turn it into an answer.
