# Scalability: Vertical vs. Horizontal

Every design conversation eventually runs into the same wall: the thing works fine right now, but what happens when it doesn't anymore? That's scalability, and it's usually the very first real decision you make in any system — long before sharding or consensus or any of the fancier topics later in this series. Get it wrong here, and you spend the next two years fighting the consequences.

There are two ways to answer "we need more capacity," and most engineers know the textbook definitions. Add a bigger box, or add more boxes. What doesn't get talked about enough is when each one actually makes sense, and how expensive it is to guess wrong.

## Fundamentals

### The bigger box

Vertical scaling means giving the same machine more resources — more CPU, more RAM, a faster disk. Nothing about your architecture changes. Your database is still one database, your app is still one process, and as far as the code is concerned, nothing happened except everything got faster.

This is tempting for a reason: it's almost free in terms of engineering effort. You resize an instance, maybe schedule a short restart, and you're done. No new failure modes, no new code paths, no distributed systems problems to think about. For a lot of early-stage systems, this is genuinely the right call, and I'd argue it's underused — people reach for horizontal scaling because it sounds more sophisticated, when a bigger instance would've solved the problem for six months at a fraction of the complexity.

But it has a ceiling, and the ceiling is physical. There's a biggest instance AWS sells. There's a limit to how much you can put in one machine before the cost curve stops being linear and starts being absurd. And there's a much quieter problem: a single machine is a single point of failure no matter how big it is. Scale vertically forever and you've built a system that's fast right up until the moment it isn't there at all.

### More boxes

Horizontal scaling means adding more machines and spreading the load across them instead of making any one of them bigger. This is where load balancers, service discovery, and a good chunk of everything else in this series comes from — because the moment you have more than one instance of something, you've inherited a whole category of problems that simply don't exist when there's only one.

```
Vertical scaling                    Horizontal scaling

   ┌─────────┐                      ┌───────┐  ┌───────┐  ┌───────┐
   │  Bigger │                      │Instance│  │Instance│  │Instance│
   │   Box   │        vs.           │   1   │  │   2   │  │   3   │
   │         │                      └───────┘  └───────┘  └───────┘
   └─────────┘                           │          │          │
                                          └────┬─────┴────┬─────┘
                                          Load Balancer / Router
```

The upside is real: no hard ceiling, and no single point of failure, since losing one instance out of ten is a rounding error instead of an outage. The cost is that your application now has to be built to run this way. State that used to live comfortably in one process — sessions, in-memory caches, anything assuming "there's only one of me" — has to move somewhere shared, or get replicated, or just get redesigned out of existence. That's not a small ask. It's the reason "just make it stateless" shows up so often in this kind of writing; it's not a platitude, it's the actual precondition for horizontal scaling to work at all.

## Operations

### Where people actually get this wrong

The mistake I see most isn't picking the wrong one — it's picking horizontal too early. A team stands up a Kubernetes cluster with autoscaling and five replicas for a service doing a few hundred requests a minute, when a single reasonably-sized instance would've handled that load without breaking a sweat, and without anyone needing to think about session affinity or distributed rate limiting or any of the rest of it.

There's a reasonable order of operations here, and it's almost always:

1. Scale vertically first, because it buys you time cheaply and doesn't add complexity.
2. Scale vertically again if there's room, because the second upgrade is usually still cheaper than a redesign.
3. Only go horizontal once you've actually hit a ceiling — a real one, not a hypothetical future one — or once availability requirements mean you can't tolerate a single instance being a single point of failure regardless of size.

That third point matters as much as the ceiling does. Sometimes you go horizontal not because one big box can't handle the load, but because you can't accept the downtime when that one box needs a restart, or dies. Availability and raw capacity are different reasons to make the same architectural move, and it's worth being honest with yourself about which one is actually driving the decision, because they lead to different designs. Capacity-driven horizontal scaling can tolerate a lot of latency in bringing new instances online. Availability-driven horizontal scaling usually can't.

### The stateless requirement isn't optional

If you take one operational lesson from this article, it's this: horizontal scaling doesn't work — not "works poorly," but doesn't work — unless your application can genuinely be stateless, or unless the state is handled somewhere all instances can see it equally. This shows up constantly in incident reviews. Someone scales an app horizontally, forgets that user sessions are stored in local memory, and now half of their users are randomly getting logged out depending on which instance the load balancer happens to route them to that request. The fix isn't a scaling fix, it's an architecture fix — move sessions to Redis, or to a database, or sign a token the client can carry instead.

The same applies to anything you might be tempted to keep local: file uploads waiting for processing, in-memory caches you built to save a database round trip, background job queues that only one process knows about. Every one of these needs a shared home before horizontal scaling is actually safe to turn on, not after.

## Deep Dive: what this looks like on AWS

In practice, on AWS, vertical scaling usually means resizing an EC2 instance type, bumping an RDS instance class, or increasing the CPU/memory limits on an ECS task definition. It's a config change and a restart, and for RDS specifically it's worth knowing you can do this with minimal downtime using Multi-AZ failover — the standby gets resized, promoted, and traffic cuts over in a handful of seconds rather than the minutes a plain restart would cost you.

Horizontal scaling on AWS usually means an Auto Scaling Group behind an Application Load Balancer for EC2, or a service on ECS/EKS with its replica count driven by target tracking — CPU, memory, or a custom CloudWatch metric like queue depth. The part people skip is testing the scale-in behavior, not just scale-out. Everyone tests "can we handle a traffic spike." Fewer people test "what happens when load drops and we terminate half our instances — do in-flight requests get dropped, does connection draining actually work, does anything break during the shrink." That's usually where the real incidents hide.

A good middle path worth knowing about, because it doesn't get mentioned enough: you don't have to pick one and commit forever. Plenty of production systems scale vertically within a horizontal fleet — run three or four instances instead of thirty, and size each one generously. You get some of horizontal's fault tolerance without needing every last piece of infrastructure to support hundreds of ephemeral nodes coming and going.

| | Vertical Scaling | Horizontal Scaling |
|---|---|---|
| Complexity added | Minimal | Significant — statelessness, discovery, load balancing |
| Ceiling | Hard limit (biggest instance) | Effectively unbounded |
| Single point of failure | Yes, always | No, if done correctly |
| Cost curve | Linear, then steep | Linear, roughly |
| Good first move for | Early-stage systems, unclear future load | Systems with proven, sustained scale needs |
| AWS mechanism | Instance resize, RDS class change | ASG + ALB, ECS/EKS with target tracking |

## What's next

Day 3 picks up latency vs. throughput — two numbers that sound like the same thing, get optimized as if they were the same thing, and are actually often in direct tension with each other.

---

*Part of the [System Design for DevOps & Cloud Engineers](README.md) series — 75 days, one concept at a time.*
