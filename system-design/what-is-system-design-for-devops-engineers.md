# What is System Design for DevOps Engineers?

If you've read any system design content before this, chances are it was written for a software engineer sitting in an interview loop, being asked to "design Twitter." They sketch a data model, draw some boxes for services, connect them with arrows, and the interviewer nods along as long as the pieces fit together and the feature works. That's a real skill. It's just not your skill, and it's not really your job.

You're the one who gets paged when someone else's clever design decision falls apart at 2 AM under real traffic. You're the one who has to explain, six months after launch, why a system that worked fine in staging is now falling over every Friday at 5 PM. So this series isn't going to pretend that DevOps and SRE system design is the same conversation as a coding interview with extra infrastructure vocabulary. It isn't. It deserves its own framework, and that's what we're building here — 75 days, one piece at a time, starting today with the question underneath all the others: what does "system design" even mean from where you sit?

## Fundamentals

### Two different jobs wearing the same name

Ask a backend engineer to design something, and the implicit success criteria is usually "does it work, does it have the right features, is the API clean." Ask you to design something, and none of that is really the point — or rather, it's assumed you'll get that part right. What actually matters is what happens when things go sideways. Does the system degrade gracefully or fall off a cliff? Can someone half-asleep figure out what's wrong from the dashboards, or do they need to wake up the person who built it?

That difference shows up everywhere once you start looking for it:

| | SWE System Design | DevOps System Design |
|---|---|---|
| Primary question | How do components talk to each other? | How does the system fail, and how do I know? |
| Success metric | Feature correctness | Availability, recoverability, blast radius |
| Time horizon | Ship it | Operate it for three years |
| Interview framing | "Design X" | "Design X to survive Y failure mode" |
| Artifact you own | The service | The platform the service runs on |

Neither column is "the real one." They're just answering different questions, and most of the confusion people run into when studying system design comes from reading material written for the left column while trying to prepare for the right one.

### Three questions worth memorizing

There's a lot of theory ahead of us in this series — CAP theorem, consensus algorithms, sharding strategies, the whole catalog. But almost everything in that catalog gets evaluated the same way in practice, and it comes down to three questions. If you take nothing else from today, take these:

1. **What happens when this component is slow, not down?** A dead service is almost the easy case — everything routes around it, alarms fire, it's obvious. Slow is the dangerous one. Slow requests pile up, threads block, retries multiply the load, and the failure spreads to healthy components before anyone notices what started it.
2. **What's the blast radius if this fails?** One customer, one availability zone, one region, or everybody? You'd be surprised how often this question doesn't get asked until after an incident, when it's suddenly very relevant.
3. **How do I find out this is failing before a customer tells me?** If you can't answer that one, you haven't actually finished the design — you've just finished the part that's fun to draw on a whiteboard.

Every deep dive in this series comes back to these three, whether we're talking about a load balancer, a database, a message queue, or a deployment pipeline. They don't change. Only the answers do.

## Operations

### "It depends" isn't a cop-out, it's the actual job

Early in a career, people want rules. Always use eventual consistency. Always put a queue in front of it. Always shard by user ID. The problem is that every one of those rules is wrong in some situation that will eventually land on your desk, and memorizing patterns doesn't help you when the pattern doesn't fit. What you're actually building — what this whole series is trying to build in you — is a fast, reliable way to weigh trade-offs on the spot, because in a real design review nobody's going to wait while you look it up.

In practice, that reasoning tends to run something like this:

```
Requirement stated
      │
      ▼
Is this a hard constraint or a preference?
      │
      ├── Hard (compliance, physics, SLA) ──► Design around it, non-negotiable
      │
      └── Preference (cost, dev velocity) ──► Weigh it: what do we
                                                give up, what do we gain
      │
      ▼
Does this decision survive a component failure?
      │
      ├── No ──► Redesign it, or write down the risk you're accepting
      │
      └── Yes ──► Ship it, instrument it, and revisit once you're an
                    order of magnitude bigger
```

That last step gets skipped more than any other part of this loop. A system built to handle ten thousand requests a day and one built for ten million aren't the same design scaled up — they're different systems that happen to solve the same problem. We'll get into exactly why when we hit capacity planning later in the series.

### The 3 AM test

Here's a filter I'd encourage you to run every design through, even ones that look finished: could the on-call engineer figure out what's wrong and do something about it at 3 AM, half-asleep, without calling the person who built it?

If the honest answer is no, the design isn't actually done, no matter how clean the architecture diagram looks in the doc. This, more than anything else, is the gap between the two columns in that table above. Software engineering system design tends to optimize for how things behave when everything's healthy. Yours has to hold up during the small percentage of the time when it isn't — because that's the only time anyone actually needs you.

## Deep Dive: how this series is going to work

Every article from here on follows the same rough shape, so you'll always know where to look for what you need:

- **Fundamentals** — the concept on its own, before we complicate it with implementation. Good for interview prep, good for building intuition.
- **Operations** — how that concept actually behaves once you throw scale, failure, multi-region traffic, and cost pressure at it.
- **Deep Dive** — a real build, usually on AWS, naming actual services and the trade-offs that come with them.

You'll see more tables like the one above, and more diagrams like the decision flow, scattered through the series. They're meant to be things you can scan mid-incident, not just read once and forget.

Worth being clear about what this series isn't going to try to be, too:

| This series covers | This series doesn't cover |
|---|---|
| Trade-offs between architectural patterns | Step-by-step Terraform or CDK for any one pattern |
| Why a design breaks under specific conditions | Vendor console click-throughs |
| Interview framing for design questions | Certification exam content |

For anything that needs implementation-level depth, this series will point back to the build guides already in this repo — the Kubernetes networking series, the EKS/IRSA guide, and the AWS ECS deep dive all cover several of these patterns at that level already.

## What's next

Day 2 picks up where most real design conversations start: scalability, and the very common mistake of reaching for horizontal scaling before you've actually confirmed vertical scaling has hit its ceiling.

---

*Part of the [System Design for DevOps & Cloud Engineers](README.md) series — 75 days, one concept at a time. Star the repo to follow along, and open an issue if there's a failure mode you'd like to see covered.*

