# grep, awk, and sed: The Log Parsing Trio Nobody Retires

There's a running joke on most infra teams that no matter how sophisticated your observability stack gets — Datadog, Grafana, ELK, whatever your company's paying six figures a year for — the first thing anyone actually does during an incident is SSH into a box and run `grep`. It's not because the fancy tooling doesn't work. It's because when production is on fire, you want raw text on a raw terminal with zero UI lag between you and the answer. Dashboards are great for confirming a hunch. They're terrible for forming one in the first ninety seconds of an incident, when you don't yet know what question to ask.

This isn't a "here's what grep does" primer. You already know what grep does. This is about the patterns that actually get used when someone pages you at 2 AM because error rates spiked, and about the habits that separate someone who finds the cause in five minutes from someone who's still scrolling twenty minutes later.

## The First Move: Narrowing the Time Window

Nobody greps an entire log file during an incident. A service handling meaningful traffic can produce gigabytes of logs per hour, and scrolling through all of it is how you miss the signal entirely — your eyes glaze over somewhere around line ten thousand. The first move is almost always narrowing to a time window that brackets the incident.

```bash
# Errors in the last 10 minutes, roughly
awk -v since="$(date -d '10 minutes ago' '+%Y-%m-%dT%H:%M:%S')" \
  '$1 >= since' app.log | grep -i error
```

At a large-scale service, this is the kind of one-liner you type without thinking, because the alternative — opening a dashboard, picking a time range with a mouse, waiting for the query to run — costs you thirty seconds you don't have when you're trying to correlate a deploy with a spike. If you're on a box without GNU date's `-d` flag (macOS ships BSD date, which behaves differently), the same idea works with a hardcoded timestamp once you know roughly when things went sideways:

```bash
awk '$1 >= "2026-07-16T14:20:00" && $1 <= "2026-07-16T14:35:00"' app.log
```

It's less elegant, but during an actual incident, elegant loses to correct-right-now every time.

## Counting Before You Read

Before reading individual log lines, the more useful question is usually "how many, and how is that trending." This is where `awk` earns its keep, because it can aggregate inline without you needing to pull the data into anything else.

```bash
# Count errors per minute, sorted chronologically
grep "ERROR" app.log | awk '{print substr($1,1,16)}' | sort | uniq -c
```

That single line turns a wall of text into a shape — you can see at a glance whether errors are climbing, flat, or already recovering. On a team running dozens of services, this is often the difference between "this is getting worse, page more people" and "this already peaked, we're fine to keep investigating calmly." A shape is worth more than a thousand individual log lines when you're trying to decide how urgently to escalate.

You can take this one step further and break errors down by type, not just by time, which usually points you straight at which subsystem is misbehaving:

```bash
grep "ERROR" app.log | awk -F'error_type=' '{print $2}' | awk '{print $1}' | sort | uniq -c | sort -rn
```

Sorting numerically in reverse at the end means the most frequent error type lands at the top of your terminal, which is usually exactly where you want your eyes to go first.

## sed for Surgical Edits, Not Just Substitution

Most people learn `sed` as "find and replace" and stop there. The more useful daily habit is using it to strip noise so the signal is easier to read. Structured JSON logs are great for machines and genuinely painful for a human eyeballing a terminal during an incident — every line is a wall of quotation marks and curly braces, and the one field you care about is buried in the middle.

```bash
# Pull just the message field out of JSON log lines, ignore the rest
sed -n 's/.*"message":"\([^"]*\)".*/\1/p' app.log | tail -50
```

It's not elegant — a proper JSON parser would be more correct, and would handle escaped quotes inside the message field, which this regex will choke on — but it's fast, it needs no dependencies, and when you're on a bastion host with nothing but coreutils installed and no internet access to `pip install` anything, fast and dependency-free wins every time.

`sed` is also genuinely useful for redacting sensitive data before you paste a log snippet into a Slack thread or an incident ticket — something that matters more than people think, because it's very easy to accidentally paste a customer's email address or an API key into a channel with hundreds of people in it.

```bash
# Redact anything that looks like an email address before sharing
sed -E 's/[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/[REDACTED]/g' app.log
```

## Piecing Together a Timeline Across Multiple Log Sources

Incidents rarely live in a single log file. A request comes in through a load balancer, hits an API gateway, gets routed to a service, which calls a database and a downstream service, and any one of those five hops could be where things actually broke. The habit that separates fast diagnosis from slow diagnosis is pulling a single request ID or trace ID and following it across every log source you have access to.

```bash
# Same request, five different log files, one chronological view
grep "req_id=8f3ac21" gateway.log service-a.log service-b.log db-proxy.log \
  | sort -t: -k1,1
```

This only works if your services are actually propagating a request ID end to end, which is worth checking *before* an incident, not during one — discovering mid-fire that your services don't correlate trace IDs across hops is a painful lesson to learn live.

## A Python Alternative When the Line Gets Blurry

There's a point where chaining five pipes together stops being clever and starts being a liability, especially if someone else needs to read your one-liner during a postmortem, or if the logic needs a conditional that shell just isn't pleasant for. That's usually the signal to drop into a short Python script instead — still fast to write, but readable six months later by someone who wasn't in the incident.

```python
import re
from collections import Counter
from datetime import datetime, timedelta

cutoff = datetime.utcnow() - timedelta(minutes=10)
error_counts = Counter()
sample_lines = {}

with open("app.log") as f:
    for line in f:
        match = re.match(r"^(\S+) .*ERROR.*service=(\w+)", line)
        if not match:
            continue
        timestamp = datetime.fromisoformat(match.group(1))
        if timestamp >= cutoff:
            service = match.group(2)
            error_counts[service] += 1
            sample_lines.setdefault(service, line.strip())

print("Top offending services in the last 10 minutes:\n")
for service, count in error_counts.most_common(5):
    print(f"{service}: {count} errors")
    print(f"  example: {sample_lines[service][:120]}")
```

Keeping a sample line per service alongside the count is a small addition, but it saves a second pass — you get the shape of the problem and a concrete example to click into, in one run.

## Why This Still Matters in a World of Managed Log Platforms

It's tempting to think this is legacy knowledge, made obsolete by centralized logging platforms with powerful query languages. In practice, those platforms still ingest raw text under the hood, and understanding how to slice that text by hand is what lets you write a good query in the first place. It's also what saves you when the platform itself is the thing that's degraded — and that does happen, more often than any vendor's status page would like to admit. Being able to `ssh` onto a box and grep local logs when your entire logging pipeline is the thing on fire is not a hypothetical skill.

The engineers who move fastest during incidents aren't the ones with the fanciest dashboards. They're the ones who can look at a wall of raw text and immediately know which two or three commands will turn it into an answer — and who've built enough muscle memory that they're not reasoning about syntax while the clock is running.
