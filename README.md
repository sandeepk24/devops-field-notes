# DevOps Field Notes

> Long-form field notes from the parts of DevOps that bite you in production — AWS, ECS, Kubernetes, Docker internals, and the occasional detour into Python and Bedrock. The reference I wish someone had handed me before I started writing runbooks from memory.

[![Stars](https://img.shields.io/github/stars/sandeepk24/devops-field-notes?style=flat-square)](https://github.com/sandeepk24/devops-field-notes/stargazers)
[![Last Commit](https://img.shields.io/github/last-commit/sandeepk24/devops-field-notes?style=flat-square)](https://github.com/sandeepk24/devops-field-notes/commits/main)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue?style=flat-square)](./LICENSE)
![AWS](https://img.shields.io/badge/AWS-orange?style=flat-square&logo=amazon-aws&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326ce5?style=flat-square&logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)
![DevOps](https://img.shields.io/badge/DevOps-informational?style=flat-square)

## Why this exists

These are notes I've kept while working through AWS containers, Docker internals, Kubernetes, and the bits of Python and Bedrock that kept showing up in my day job. I wrote them mostly for myself — the second time I had to re-derive how an ALB listener rule actually resolves to an ECS target, I decided I should write it down properly instead.

Most of them are long-form rather than one-liners. If a topic deserves a mental model, I'd rather spend 800 lines getting it right than paste a cheat sheet I won't trust at 2 a.m.

They're aimed at mid-to-senior DevOps engineers who are past "hello world" and want the mechanics — the stuff that makes the difference between "the deployment succeeded" and "the application is actually serving traffic."

## What's inside

### ☁️ AWS
| Note | What's in it |
|---|---|
| [ALB + ECS deep dive](./aws/aws-alb-ecs-deep-dive.md) | ALB components, listener rules, target groups, and how ECS services actually register. End-to-end. |
| [Bedrock 200-level fundamentals](./aws/aws-bedrock-200-level-fundamentals.md) | Building the mental model before the code. First Bedrock project without the copy-paste tutorial trap. |
| [Bedrock agentic AI in production](./aws/aws-bedrock-advanced-agentic-ai.md) | AgentCore, Strands SDK, S3 Vectors, A2A, MCP — and what breaks under real concurrency. |

### 🐳 Docker
| Note | What's in it |
|---|---|
| [Docker internals, part 1 — execution stack, namespaces, cgroups](./docker/docker-advanced-part-1.md) | How Docker actually works at the kernel level. |
| [Docker internals, part 2 — OverlayFS, OCI, BuildKit](./docker/docker-advanced-part-2.md) | Storage engine and build architecture, and why they shape image design. |
| [Docker internals, part 3 — networking, security, signals, debugging](./docker/docker-advanced-part-3.md) | Production failure modes and the security model. |
| [Docker advanced tutorial](./docker/docker-advanced-tutorial.md) | Compose, Swarm, registry ops, checkpointing — the production-grade workflow field guide. |
| [`docker build` vs `docker buildx`](./docker/docker-build-vs-buildx-is-not-just-a-new-command.md) | Not a version upgrade. A different mental model for where your builds live. |

### 📦 ECS
| Note | What's in it |
|---|---|
| [ECS fundamentals for DevOps engineers](./ecs/what-devops-engineers-should-understand-about-ecs.md) | Cluster, service, task, task definition — with the kubectl equivalents for anyone crossing over. |
| [ECS task health is not application health](./ecs/ecs-task-health-is-not-the-same-as-app-health.md) | The four layers of health and why teams stop reading at the wrong one. |

### ⚙️ Kubernetes / CKAD
| Note | What's in it |
|---|---|
| [`kubectl rollout status` is underrated](./eks/kubectl-rollout-status-is-underrated.md) | The pipeline gatekeeper command most teams forget to use. |
| [CKAD deployments — part 1](./ckad/ckad-deployments-part-1.md) | Annotated Deployment YAML, every spec field, exam-speed imperative generation. |
| [CKAD deployments — part 2](./ckad/ckad-deployments-part-2.md) | Advanced patterns, rollout strategies, and the 10 rules I'd tell anyone sitting the exam. |

### 🐍 Python
| Note | What's in it |
|---|---|
| [FastAPI for DevOps engineers](./python/what-devops-engineers-should-know-about-fastapi-in-production.md) | Python fundamentals → production-grade FastAPI. For DevOps folks building internal tools, automation APIs, and AI pipelines. |

## Highlights

A few of the ones I reach for most often:

- **[ECS task health is not application health](./ecs/ecs-task-health-is-not-the-same-as-app-health.md)** — the note I wish every team read before they wire up their first ECS deployment pipeline. Explains the four layers of health and why `RUNNING` tells you almost nothing.
- **[ALB + ECS deep dive](./aws/aws-alb-ecs-deep-dive.md)** — the one I re-read every time I have to debug why a target is unhealthy for reasons that aren't actually about the target.
- **[`docker build` vs `docker buildx`](./docker/docker-build-vs-buildx-is-not-just-a-new-command.md)** — written after I watched one too many engineers treat buildx as a drop-in upgrade and then lose an image to the cache.
- **[Bedrock agentic AI in production](./aws/aws-bedrock-advanced-agentic-ai.md)** — the Bedrock article I wanted to find and couldn't. Less "how do I call Claude" and more "what's the execution model I'm actually building on."
- **[`kubectl rollout status` is underrated](./eks/kubectl-rollout-status-is-underrated.md)** — short, but this is the command I lean on to close the gap between "kubectl apply succeeded" and "the rollout actually finished."
- **[FastAPI for DevOps engineers](./python/what-devops-engineers-should-know-about-fastapi-in-production.md)** — the longest note in the repo. Starts at Python fundamentals so DevOps folks picking it up for the first time don't have to stitch together five other tutorials.

## How I use this repo

- **Browse on GitHub** — the folder structure is flat on purpose. Skim until a title catches your eye.
- **Clone and grep locally** — `git clone` and `grep -r "some error" .` is how I actually find things when I'm mid-incident. The notes are plain markdown, which is the point.
- **Fork and adapt** — if the structure works for you, rip it out and make it your own team's reference. That's what it's for.

## Contributing

If I got something wrong, open an issue or a PR — I'd rather be right than look right. Corrections, additions, and "actually, here's a sharper way to put it" are all welcome. These notes improve every time someone pushes back on them.

## Star history

<!--
Once this repo has a handful of stars, enable star-history:

[![Star History Chart](https://api.star-history.com/svg?repos=sandeepk24/devops-field-notes&type=Date)](https://star-history.com/#sandeepk24/devops-field-notes&Date)
-->

_Will enable the star-history embed once the repo has enough stars to make the chart worth looking at._

## License

MIT. Use it, fork it, lift bits for your own team's runbooks.
