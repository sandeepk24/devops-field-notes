# Latency vs. Throughput

These two get lumped together so often that people start treating them as the same conversation with two names. They're not. You can improve one and make the other worse without touching a single unrelated setting, and if you don't understand why, you'll end up optimizing for the wrong number and wondering why users are still complaining even though your dashboards look great.

Here's the short version, before we get into why it matters: latency is how long one request takes. Throughput is how many requests the system gets through in a given window of time. They sound related because they both involve "speed," but they answer completely different questions, and a system can be excellent at one while being mediocre at the other.

## Fundamentals

### Latency: the wait

Latency is the time between sending a request and getting a response back — start the clock, do the work, stop the clock. That's it. If someone says "the API feels slow," they're almost always talking about latency, because that's the number a human actually experiences. Nobody sitting at a keyboard feels your system's throughput. They feel how long they waited.

It gets more useful once you stop thinking about it as one number, though, because the average is often lying to you. Say your average latency is 100ms — sounds fine, right? But if 95% of requests come back in 20ms and the remaining 5% take two full seconds, your average still looks great while one in twenty users is having a genuinely bad experience. This is why you'll almost never see production systems reported on average latency alone. You'll see p50, p95, p99 — the 50th, 95th, and 99th percentile — because those tell you what the tail actually looks like, and the tail is usually where the real problems live.

### Throughput: the volume

Throughput is about capacity, not experience — requests per second, transactions per minute, whatever unit fits the system. It answers "how much can this thing handle," not "how did any individual request feel." A batch job that processes ten million records overnight cares enormously about throughput and probably doesn't care at all that any single record's processing took 50ms instead of 5ms. Nobody's staring at that record waiting for a response.

### Why they pull against each other

Here's the part that trips people up: pushing throughput up often pushes latency up too, and it's not a bug, it's just physics. Batching is the clearest example. If you process requests one at a time as they arrive, latency per request is as low as it can be, but you're leaving throughput on the table because you're not taking advantage of any economy of scale. Batch twenty requests together and process them as a group, and your throughput jumps — you're doing more total work per unit of overhead — but now the first request in that batch of twenty has to sit and wait for the other nineteen to show up before anything happens. You just traded latency for throughput, on purpose, and that trade is often the right one to make. It's just not free.

```
Low latency, low throughput          High throughput, higher latency

  req ──► [process] ──► resp           req1 ─┐
  req ──► [process] ──► resp           req2 ─┼─► [batch] ──► resp×20
  req ──► [process] ──► resp           ...    │   (after waiting
  (one at a time, fast each time)      req20 ─┘    for the batch)
```

## Operations

### Match the metric to the actual system

The mistake that shows up over and over is optimizing for the wrong one because it's the one that's easier to measure, or the one that sounds more impressive on a slide. A user-facing checkout flow needs to obsess over p99 latency, because that tail is exactly where you lose sales — nobody's forgiving a two-second hang at checkout just because your average is fast. A nightly ETL pipeline moving billions of rows genuinely does not care what any individual row's latency was. It cares whether the whole job finishes inside its window. Optimizing the ETL job for latency would mean spending engineering effort shaving milliseconds off something nobody's watching in real time, while the actual lever — throughput — sits untouched.

So before you optimize anything, it's worth just asking plainly: is a human waiting on this, or is this about total volume over a longer stretch of time? That single question tells you which metric actually deserves your attention, and more importantly, which one you're allowed to sacrifice a little of to improve the other.

### Where this actually bites you in production

Connection pools are a good example of how this plays out for real. A small pool keeps latency low when traffic is light, because requests get a connection immediately and move on. Push more traffic through that same small pool, though, and requests start queuing for a connection to free up — latency climbs, even though nothing about the backend itself got slower. The system's throughput ceiling became a latency problem for anyone unlucky enough to be waiting behind it.

Queues do this too, in a way that's worth sitting with for a second, because it's counterintuitive the first time you see it. A queue can absolutely improve your throughput — it smooths out bursts, lets you process at a sustainable, even pace instead of getting overwhelmed by spikes — while simultaneously making latency worse for individual messages, because now every message sits in line behind whatever arrived before it. Both things are true about the same design decision at the same time. That's not a contradiction; it's just two different measurements of two different concerns, and you have to decide up front which one your system can afford to sacrifice.

## Deep Dive: measuring this on AWS

CloudWatch gives you both, but you have to actually go looking for the right numbers instead of trusting the defaults. ALB's `TargetResponseTime` metric gives you latency, and by default CloudWatch shows you the average — which, as covered above, hides the exact thing you probably need to see. Switch the statistic to p99 and you'll often find a very different, much less comfortable story than the average was telling you.

For throughput, `RequestCountPerTarget` combined with target group health tells you whether you're actually spreading load evenly or whether a handful of targets are quietly absorbing more traffic than the rest, which will show up as latency problems on exactly those overloaded targets — a throughput imbalance disguising itself as a latency issue if you're not looking in the right place.

A quick way to see the trade-off in your own numbers if you're on Python:

```python
import time
import statistics

latencies = []

def timed_request(fn, *args, **kwargs):
    start = time.perf_counter()
    result = fn(*args, **kwargs)
    latencies.append(time.perf_counter() - start)
    return result

# after a batch of calls:
p50 = statistics.median(latencies)
p99 = statistics.quantiles(latencies, n=100)[98]
throughput = len(latencies) / sum(latencies)  # rough requests/sec

print(f"p50: {p50*1000:.1f}ms  p99: {p99*1000:.1f}ms  throughput: {throughput:.1f} req/s")
```

Run something like this against a service under a batching change, and you'll usually see it directly: throughput goes up, p99 goes up right along with it, and now you have real numbers to decide whether that trade was worth making instead of guessing.

| | Optimize for Latency | Optimize for Throughput |
|---|---|---|
| Best for | User-facing, interactive requests | Batch jobs, background processing |
| What you measure | p50 / p95 / p99 | Requests or records per second |
| Common technique | Smaller batches, more parallelism, caching | Batching, queuing, connection pooling |
| Trade-off accepted | Lower total capacity | Slower individual requests |
| Where it breaks if ignored | Checkout flows, live APIs, anything real-time | Nightly jobs missing their window, pipeline backlog |

## What's next

Day 4 is CAP theorem — the one everyone's heard of, half of them can recite, and fewer of them can actually apply correctly to a design decision under pressure.

---

*Part of the [System Design for DevOps & Cloud Engineers](README.md) series — 75 days, one concept at a time.*
