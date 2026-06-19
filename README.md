# DevOps Field Notes

> Long-form field notes from the parts of DevOps that bite you in production — AWS, ECS/EKS, Kubernetes, Docker internals, networking, CI/CD, SRE/observability, and a growing stack of AI/LLMOps notes. The reference I wish someone had handed me before I started writing runbooks from memory.

[![Stars](https://img.shields.io/github/stars/sandeepk24/devops-field-notes?style=flat-square)](https://github.com/sandeepk24/devops-field-notes/stargazers)
[![Last Commit](https://img.shields.io/github/last-commit/sandeepk24/devops-field-notes?style=flat-square)](https://github.com/sandeepk24/devops-field-notes/commits/main)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue?style=flat-square)](./LICENSE)
![AWS](https://img.shields.io/badge/AWS-orange?style=flat-square&logo=amazon-aws&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326ce5?style=flat-square&logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)
![Bedrock](https://img.shields.io/badge/Bedrock-232F3E?style=flat-square&logo=amazon-aws&logoColor=white)
![DevOps](https://img.shields.io/badge/DevOps-informational?style=flat-square)

## Why this exists

These are notes I've kept while working through AWS containers, Docker internals, Kubernetes, networking, and — increasingly — the AI and LLMOps work that's landed on every DevOps engineer's desk. I wrote them mostly for myself: the second time I had to re-derive how an ALB listener rule actually resolves to an ECS target, I decided I should write it down properly instead.

Most of them are long-form rather than one-liners. If a topic deserves a mental model, I'd rather spend 800 lines getting it right than paste a cheat sheet I won't trust at 2 a.m.

They're aimed at mid-to-senior DevOps and cloud engineers who are past "hello world" and want the mechanics — the stuff that makes the difference between "the deployment succeeded" and "the application is actually serving traffic."

## What's inside

### 🤖 AI / LLMOps for DevOps
| Note | What's in it |
|---|---|
| [GenAI vs Agentic AI — part 1](./ai-devops/genai-vs-agentic-ai-part1.md) | The foundational distinction: AI that helps you *think* vs AI that *acts* on your behalf — and why getting it wrong costs you in security, compliance, and architecture. |
| [GenAI vs Agentic AI on AWS — part 2](./ai-devops/genai-vs-agentic-ai-part2.md) | The actual AWS services behind the vocabulary, and what you can build with them right now. |
| [AI incident triage agent](./ai-devops/ai-incident-triage-agent.md) | From alert noise to root cause in seconds — designing an AIOps triage agent for real incident response. |
| [RAG knowledge base for DevOps runbooks](./ai-devops/rag-knowledge-base-devops-runbooks.md) | Turning a pile of runbooks into a retrieval-augmented knowledge base — golden datasets, chunking, and grounding. |
| [LLMOps evaluation pipeline for DevOps bots](./ai-devops/llmops-evaluation-pipeline-for-devops-bots.md) | Catching prompt drift, model regression, and silent quality decay before they compound into incidents. |
| [Monitoring AI agents in production](./ai-devops/how-to-monitor-ai-agents-in-production.md) | Why AI monitoring is a different problem, and how to catch false outputs before users do. |
| [SRE for LLM applications](./ai-devops/sre-for-llm--applications.md) | What being on-call for an LLM actually looks like — written from the on-call seat, not the demo. |
| [MCP — the DevOps engineer's field guide](./ai-devops/MCP_DevOps_Guide.md) | Model Context Protocol in plain English: what it is, why it exists, and where it fits. |
| [MCP advanced guide for DevOps engineers & architects](./ai-devops/MCP_Advanced_DevOps_Guide.md) | LLMs vs RAG vs MCP, advanced server patterns, and the architecture trade-offs. |
| [AWS Strands Agents — complete guide](./ai-devops/strands-agents-bedrock-guide.md) | The Strands SDK for cloud & DevOps engineers: what it is and why you'd reach for it. |
| [Amazon Bedrock model availability](./ai-devops/amazon-bedrock-model-availability.md) | Why your model isn't available — and how to actually fix it instead of fighting IAM blind. |

### ☁️ AWS
| Note | What's in it |
|---|---|
| [ALB + ECS deep dive](./aws/aws-alb-ecs-deep-dive.md) | ALB components, listener rules, target groups, and how ECS services actually register. End-to-end. |
| [ECS vs EKS for enterprise applications](./aws/ecs-vs-eks-for-enterprise-applications.md) | A no-BS guide to picking your AWS container platform — from the trenches, for people who just want to ship. |
| [EBS volumes on EKS](./aws/ebs-volumes-on-eks-the-complete-guide-for-devops-engineers.md) | What problem EBS actually solves, the CSI driver, and how persistent storage works on EKS. |
| [AWS Cognito complete guide](./aws/aws-cognito-complete-guide.md) | User authentication without the headaches — user pools, identity pools, and the flows that trip people up. |
| [CPU vs GPU — the missing manual](./aws/cpu-vs-gpu-the-devops-engineers-missing-manual.md) | CPU pipeline internals, the Python GIL + asyncio, Java JIT, Kubernetes throttling, GPU/CUDA, and AWS instance selection — with real benchmarks. |
| [Bedrock 200-level fundamentals](./aws/aws-bedrock-200-level-fundamentals.md) | Building the mental model before the code. Your first Bedrock project without the copy-paste tutorial trap. |
| [Bedrock agentic AI in production](./aws/aws-bedrock-advanced-agentic-ai.md) | AgentCore, Strands SDK, S3 Vectors, A2A, MCP — and what breaks under real concurrency. |
| [What happens when you type www.google.com](./aws/what-happens-when-you-type-www-google-com.md) | The full request lifecycle — DNS, TLS, routing, the lot — as a systems-thinking warm-up. |

### 🌐 Networking
| Note | What's in it |
|---|---|
| [Networking 101 for DevOps & cloud engineers](./networking/01-networking-101-devops-cloud.md) | What actually matters in production — the fundamentals you keep needing and never wrote down. |
| [DNS deep dive](./networking/02-dns-deep-dive.md) | Resolution, failure modes, CoreDNS, and Route 53 — why "it's always DNS" is usually right. |
| [TLS/SSL deep dive](./networking/03-tls-ssl-deep-dive.md) | Certificates, mTLS, and how to actually debug a handshake in production. |
| [VPC networking deep dive](./networking/04-vpc-networking.md) | Subnets, routing, NAT, peering, and Transit Gateway — the AWS network plane explained. |
| [Kubernetes networking deep dive](./networking/05-kubernetes-networking.md) | CNI, kube-proxy, Services, and Ingress — how a packet actually reaches a pod. |
| [Load balancing deep dive](./networking/06-load-balancing.md) | ALB, NLB, health checks, and connection draining — and why a "healthy" target still drops requests. |
| [Network security](./networking/07-network-security.md) | Security groups, NACLs, Zero Trust, and WAF — the layered model and where each control belongs. |
| [Network observability](./networking/08-network-observability.md) | Flow logs, tcpdump, packet analysis, and distributed tracing for when the network is the suspect. |

### 🐳 Docker
| Note | What's in it |
|---|---|
| [Docker internals, part 1 — execution stack, namespaces, cgroups](./docker/docker-advanced-part-1.md) | How Docker actually works at the kernel level. |
| [Docker internals, part 2 — OverlayFS, OCI, BuildKit](./docker/docker-advanced-part-2.md) | Storage engine and build architecture, and why they shape image design. |
| [Docker internals, part 3 — networking, security, signals, debugging](./docker/docker-advanced-part-3.md) | Production failure modes and the security model. |
| [Docker advanced tutorial](./docker/docker-advanced-tutorial.md) | Compose, Swarm, registry ops, checkpointing — the production-grade workflow field guide. |
| [`docker build` vs `docker buildx`](./docker/docker-build-vs-buildx-is-not-just-a-new-command.md) | Not a version upgrade. A different mental model for where your builds live. |
| [Docker Compose → ECS playbook](./docker/docker-compose-local-to-ecs-the-complete-devops-playbook.md) | The local-to-production path: why Compose exists and how to take it all the way to ECS. |

### 📦 ECS
| Note | What's in it |
|---|---|
| [ECS fundamentals for DevOps engineers](./ecs/what-devops-engineers-should-understand-about-ecs.md) | Cluster, service, task, task definition — with the kubectl equivalents for anyone crossing over. |
| [ECS task health is not application health](./ecs/ecs-task-health-is-not-the-same-as-app-health.md) | The four layers of health and why teams stop reading at the wrong one. |
| [ECS task distribution & resource management](./ecs/ecs-task-distribution-guide.md) | How containers are scheduled, distributed, and packed across an ECS cluster. |
| [ECS reliability best practices](./ecs/ecs-reliability-best-practices.md) | Building bulletproof clusters and catching problems before they become incidents. |
| [ECS cost optimization & failure domains](./ecs/ecs-deep-dive-cost-and-failures.md) | When to use Fargate vs EC2, and how ECS actually behaves when things fail. |

### ⚙️ Kubernetes / EKS / CKAD
| Note | What's in it |
|---|---|
| [EKS frontend connectivity — ALB · NLB · Ingress](./eks/aws-alb-nlb-ingress-eks-deep-dive.md) | End-to-end deep dive on getting traffic into EKS, and the mental-model shift from ECS load balancing. |
| [`kubectl rollout status` is underrated](./eks/kubectl-rollout-status-is-underrated.md) | The pipeline gatekeeper command most teams forget to use. |
| [CKAD deployments — part 1](./ckad/ckad-deployments-part-1.md) | Annotated Deployment YAML, every spec field, exam-speed imperative generation. |
| [CKAD deployments — part 2](./ckad/ckad-deployments-part-2.md) | Advanced patterns, rollout strategies, and the 10 rules I'd tell anyone sitting the exam. |
| [Kubernetes Ingress — CKAD exam + EKS production](./ckad/kubernetes-ingress-ckad-exam-and-eks-production-mastery.md) | The Ingress mental model, enough to pass CKAD and to actually ship it on EKS. |

### 🔁 CI/CD
| Note | What's in it |
|---|---|
| [GitHub DevOps fundamentals & best practices](./ci-cd/01_GITHUB_DEVOPS_FUNDAMENTALS.md) | The foundations — repo hygiene, Actions, and the practices that keep a pipeline sane. |
| [GitHub DevOps complete implementation](./ci-cd/github-devops-implementation-02.md) | Building the pipeline for real: workflows, jobs, and a working Python app to hang it on. |
| [GitHub DevOps production deployment](./ci-cd/github-devops-production-03.md) | Taking the pipeline to production deploys against AWS ECS. |
| [GitLab CI/CD 101](./ci-cd/gitlab-cicd-101.md) | End-to-end GitLab CI/CD from first principles — what CI/CD is and why it matters. |
| [Branch strategies](./ci-cd/branch-strategies.md) | A no-nonsense look at the branching models and when each one actually fits. |
| [Trunk-based development](./ci-cd/trunk-based-development.md) | Why long-lived branches hurt, and how trunk-based development avoids the Monday merge marathon. |

### 📈 SRE / Observability
| Note | What's in it |
|---|---|
| [OpenTelemetry 101](./sre/otel_101.md) | The beginner's mental model for modern observability — metrics, logs, and traces in one language. |
| [OpenTelemetry for cloud architects](./sre/otel_intermediate.md) | AWS at scale, multi-cloud from day one, and the OTel mistakes teams keep repeating. |
| [CloudWatch dashboard design for EKS](./sre/cloudwatch-dashboard-design-for-eks.md) | Stitching control plane, data plane, the K8s object model, and app metrics into one pane of glass. |
| [Deployment strategies — canary vs blue/green vs rolling](./sre/canary-vs-blue-green-vs-rolling-deployments.md) | Every deployment strategy, when to use it, and how to pitch it as the business decision it is. |
| [Post-deployment validation checklist](./sre/post-deployment-validation-checklist.md) | Why green CI isn't "done," and what to actually check after the pipeline passes. |

### 🐍 Python
| Note | What's in it |
|---|---|
| [FastAPI for DevOps engineers](./python/what-devops-engineers-should-know-about-fastapi-in-production.md) | Python fundamentals → production-grade FastAPI. For DevOps folks building internal tools, automation APIs, and AI pipelines. |
| [Pydantic — the complete DevOps guide](./python/python_pydantic_devops_guide.md) | Why Pydantic matters in DevOps and how to use it for config, validation, and data contracts. |

## Highlights

A few of the ones I reach for most often:

- **[ECS task health is not application health](./ecs/ecs-task-health-is-not-the-same-as-app-health.md)** — the note I wish every team read before they wire up their first ECS deployment pipeline. Explains the four layers of health and why `RUNNING` tells you almost nothing.
- **[ALB + ECS deep dive](./aws/aws-alb-ecs-deep-dive.md)** — the one I re-read every time I have to debug why a target is unhealthy for reasons that aren't actually about the target.
- **[GenAI vs Agentic AI — part 1](./ai-devops/genai-vs-agentic-ai-part1.md)** — the framing I keep sending people: the difference between AI that helps you think and AI that acts on your behalf, and why that line matters for security and architecture.
- **[SRE for LLM applications](./ai-devops/sre-for-llm--applications.md)** — written from the on-call seat, not the demo. If you're about to put an LLM in front of real users, start here.
- **[`docker build` vs `docker buildx`](./docker/docker-build-vs-buildx-is-not-just-a-new-command.md)** — written after I watched one too many engineers treat buildx as a drop-in upgrade and then lose an image to the cache.
- **[Bedrock agentic AI in production](./aws/aws-bedrock-advanced-agentic-ai.md)** — the Bedrock article I wanted to find and couldn't. Less "how do I call Claude" and more "what's the execution model I'm actually building on."
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
</content>
</invoke>
