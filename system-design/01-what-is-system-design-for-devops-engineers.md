# What is System Design for DevOps Engineers?

Most system design content is written for software engineers designing a service. You're not doing that — you're the one who gets paged when someone else's design decision turns into an outage. That's a different discipline, and it needs its own framework.

This article opens a 75-day series on system design specifically for DevOps, SRE, and cloud engineers. Before we get into CAP theorem, sharding, or consensus algorithms, we need to establish what "system design" even means from this seat — because it's not the same conversation a backend engineer has in an interview loop.

## Fundamentals

### The SWE version vs. the DevOps version

A software engineer's system design interview asks: *"Design Twitter."* The answer is a data model, an API surface, and a handful of services with lines drawn between them. Correctness and feature completeness are the implicit goals.

Your version of that question is different. You're rarely designing the application logic — you're designing the substrate it runs on, and you're accountable for what happens when that substrate is under stress, partially failed, or being attacked. The implicit goal isn't "does it work" — it's "does it keep working, and can I tell why it stopped."

| | SWE System Design | DevOps System Design |
|---|---|---|
| Primary question | How do components talk to each other? | How does the system fail, and how do I know? |
| Success metric | Feature correctness | Availability, recoverability, blast radius |
| Time horizon | Ship it | Operate it for 3 years |
| Interview framing | "Design X" | "Design X to survive Y failure mode" |
| Artifact you own | The service | The platform the service runs on |

### The three questions that actually matter

Every design decision in this series gets run through the same three questions. If you internalize nothing else from Day 1, internalize these:

1. **What happens when this component is slow, not down?** (Down is easy — everything routes around it. Slow is what actually causes cascading failures.)
2. **What's the blast radius if this fails?** (One tenant, one AZ, one region, everyone?)
3. **How do I find out it's failing before a customer tells me?** (If the answer is "I don't," that's the design flaw — not a gap to fill later.)

You'll see these three questions reused in every deep dive in this series, applied to load balancers, databases, message queues, and deployment pipelines alike.

## Operations

### Why "it depends" is the actual skill

Junior engineers want a rule: "always use eventual consistency," "always use a queue," "always shard by user ID." Every one of those rules is wrong in some context that matters. The skill this series is building isn't memorizing patterns — it's building a fast, reliable trade-off evaluation loop, because in a real design review you don't get to look it up.

Concretely, that loop looks like:

```
Requirement stated
      │
      ▼
Is this a hard constraint or a preference?
      │
      ├── Hard (compliance, physics, SLA) ──► Design around it, non-negotiable
      │
      └── Preference (cost, dev velocity) ──► Trade-off table: what do we
                                                give up, what do we gain
      │
      ▼
Does this decision survive a component failure?
      │
      ├── No ──► Redesign or explicitly accept the risk in writing
      │
      └── Yes ──► Ship it, instrument it, revisit at next order-of-magnitude
```

That last branch — "revisit at next order-of-magnitude" — is the part most designs skip. A system designed for 10K requests/day and one designed for 10M requests/day are not the same system with bigger boxes. We'll cover this explicitly in the capacity planning article (Day 44).

### The 3 AM test

Here's a filter to apply to any design before it ships: **could the on-call engineer diagnose and mitigate this failure at 3 AM, half-awake, without paging the original author?**

If the answer is no, the design isn't done — regardless of how elegant the architecture diagram looks. This is the single biggest gap between SWE system design and DevOps system design: the former optimizes for how the system behaves when everything is healthy; the latter has to optimize for the 2% of the time it isn't, because that 2% is when you're actually needed.

## Deep Dive: How this series is structured

Every article from here forward follows the same shape, so you always know where to find what:

- **Fundamentals** — the concept, stripped of implementation detail. This is your interview-prep layer.
- **Operations** — how the concept behaves under real operational conditions: failure, scale, multi-region, cost.
- **Deep Dive** — a concrete build, usually on AWS, with the actual services and trade-offs named.

Do/don't tables and failure-mode ASCII diagrams like the ones above will show up throughout — they're meant to be skimmable during an actual incident, not just read once during study.

### What this series is not

| This series covers | This series does not cover |
|---|---|
| Trade-offs between architectural patterns | Step-by-step Terraform/CDK for any one pattern |
| Why a design fails under specific conditions | Vendor-specific console click-throughs |
| Interview framing for design questions | Certification exam content |

For implementation depth on any pattern discussed here, this series links back to the existing deep-dive guides in this repo — the [Kubernetes networking series](../), [EKS/IRSA guide](../), and [AWS ECS deep dive](../) already cover build-level detail for several patterns we'll reference conceptually.

## What's next

Day 2 starts where every real design conversation starts: scalability, and the very common mistake of reaching for horizontal scaling before you've actually hit the ceiling on vertical.

---

*Part of the [System Design for DevOps & Cloud Engineers](README.md) series — 75 days, one concept at a time. Star the repo to follow along, and open an issue if there's a failure mode you want covered.*
