# AWS ALB + ECS Deep Dive: End-to-End Load Balancer Architecture
> **Audience:** DevOps Engineers · Senior DevOps Engineers · Principal DevOps Engineers  
> **Scope:** Application Load Balancer (ALB) concepts, routing mechanisms, and full ECS integration  
> **Last Updated:** 2026

---

## Table of Contents

1. [Load Balancer Types on AWS](#1-load-balancer-types-on-aws)
2. [ALB Architecture — Every Component Explained](#2-alb-architecture--every-component-explained)
   - [The ALB Resource](#the-alb-resource-itself)
   - [Listeners](#listeners)
   - [Rules and Routing](#rules-and-routing--the-core-of-alb-intelligence)
3. [Target Groups — The Bridge to ECS](#3-target-groups--the-bridge-to-ecs)
   - [Target Types](#target-types)
   - [Health Checks](#health-checks--the-most-critical-configuration)
   - [Stickiness](#stickiness-session-affinity)
4. [ECS Integration — Deep Dive](#4-ecs-integration--deep-dive)
   - [Network Modes](#ecs-network-modes-and-their-impact-on-alb-integration)
   - [Service Load Balancer Config](#ecs-service-load-balancer-configuration)
   - [The Full Request Path](#the-full-request-path)
   - [Security Group Chain](#security-group-chain--the-most-misunderstood-part)
   - [Deployments and the ALB](#ecs-service-deployment-and-the-alb)
5. [Advanced ALB Patterns for ECS](#5-advanced-alb-patterns-for-ecs-architectures)
6. [Terraform IaC Reference](#6-iac--terraform-reference-patterns)
7. [Observability — What to Monitor](#7-observability--what-to-monitor)
8. [AWS Reference Links](#8-aws-reference-links)

---

## 1. Load Balancer Types on AWS

Before touching any AWS console, you need to internalize *where* each load balancer type operates in the OSI stack — this determines everything about what it can and cannot do.

| Load Balancer | OSI Layer | Protocol | Key Use Case | ECS Relevance |
|---|---|---|---|---|
| **ALB** | Layer 7 | HTTP / HTTPS / WebSocket / gRPC | Content-based routing, microservices | ✅ Primary choice |
| **NLB** | Layer 4 | TCP / UDP / TLS | Ultra-low latency, static IPs, gRPC at scale | ✅ High-throughput / IP whitelisting |
| **GWLB** | Layer 3 | GENEVE | Inline network appliances (firewall, IDS/IPS) | ❌ Not for app workloads |
| **CLB** | Layer 4/7 | HTTP / TCP (legacy) | Legacy only | ❌ Retired — never use for new builds |

> 📖 **AWS Docs:** [Elastic Load Balancing product overview](https://aws.amazon.com/elasticloadbalancing/)  
> 📖 **AWS Docs:** [Comparison of load balancer types](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/load-balancer-types.html)  
> 📖 **AWS Docs:** [ALB vs NLB vs CLB — choosing the right type](https://aws.amazon.com/elasticloadbalancing/features/)

---

## 2. ALB Architecture — Every Component Explained

### The ALB Object Model

An ALB is composed of a strict hierarchy of objects:

```
ALB (the load balancer resource itself)
 └── Listeners (port + protocol bindings)
      └── Rules (ordered, evaluated top-to-bottom)
           └── Actions (what to do when a rule matches)
                └── Target Groups (where traffic is sent)
                     └── Targets (ECS tasks, EC2 instances, IPs, Lambda)
```

---

### The ALB Resource Itself

When you create an ALB, AWS provisions a fleet of load balancer nodes — **one per Availability Zone** you select. This is critical:

- An ALB node lives in a specific AZ and only receives traffic for subnets in that AZ
- If only one AZ is enabled and ECS tasks are spread across AZs, cross-zone traffic will occur (AWS charges for it unless cross-zone load balancing is enabled)
- **Always enable at least two AZs** on your ALB for high availability

The ALB is assigned a **DNS name** (not a static IP). This resolves to the ALB node IPs across your enabled AZs. Register the ALB DNS name in Route 53 — never hardcode IPs.

**ALB Scheme** — first critical configuration decision:

| Scheme | Node IPs | Reachable From | Typical Subnet |
|---|---|---|---|
| `internet-facing` | Public IPs | Public internet | Public subnets |
| `internal` | Private IPs only | VPC / VPN / Direct Connect | Private subnets |

> The canonical production pattern: internet-facing ALB in **public subnets**, ECS tasks in **private subnets**. The ALB proxies from public → private.

> 📖 **AWS Docs:** [Create an Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-application-load-balancer.html)  
> 📖 **AWS Docs:** [ALB components overview](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html)  
> 📖 **AWS Docs:** [Cross-zone load balancing](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/disable-cross-zone.html)

---

### Listeners

A **Listener** is a process that checks for connection requests using a protocol and port you configure. Every ALB must have at least one listener (up to 50 per ALB).

**Listener Protocol/Port combinations:**

| Protocol | Port | Common Use |
|---|---|---|
| HTTP | 80 | Redirect to HTTPS only |
| HTTPS | 443 | Primary production listener |
| HTTP | 8080 | Internal/API traffic |
| HTTPS | 8443 | Alternate HTTPS for internal |

**Key Listener Concepts:**

**SSL/TLS Termination** — when using HTTPS, the ALB terminates the TLS connection. The certificate lives on the ALB managed via ACM (AWS Certificate Manager). Traffic between ALB and ECS can be HTTP (most common — acceptable because traffic is within VPC) or HTTPS (for end-to-end encryption compliance requirements).

**SNI (Server Name Indication)** — a single ALB listener can host **multiple SSL certificates**. When a client connects, the ALB reads the SNI field in the TLS handshake to determine which certificate to serve. One ALB can serve `api.app.com`, `app.app.com`, and `admin.app.com` each with their own certificate. Each listener supports up to **25 certificates** plus the default.

**ALPN (Application-Layer Protocol Negotiation)** — critical for gRPC. When ALPN is set to `h2` on the listener, it negotiates HTTP/2 which gRPC requires. Without this configuration, gRPC over ALB will fail silently.

**TLS Security Policy** — always use the latest recommended policy. As of 2026, use `ELBSecurityPolicy-TLS13-1-2-2021-06` which enforces TLS 1.2+ and prefers TLS 1.3.

> 📖 **AWS Docs:** [ALB listeners](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html)  
> 📖 **AWS Docs:** [HTTPS listener with ACM certificates](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-https-listener.html)  
> 📖 **AWS Docs:** [ALB SNI — multiple certificates per listener](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/https-listener-certificates.html)  
> 📖 **AWS Docs:** [ALB ALPN policies for gRPC / HTTP2](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html#alpn-policies)  
> 📖 **AWS Docs:** [TLS security policies for ALB](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/describe-ssl-policies.html)

---

### Rules and Routing — The Core of ALB Intelligence

Every listener has an ordered list of **rules**. Rules are evaluated from **priority 1** (highest) to **priority 50,000**, with the default rule evaluated last (catch-all).

> ⚠️ **Critical:** The first rule that matches wins. This is NOT a "best match" system. **Order matters.**

**Rule anatomy:**
```
IF [condition(s)]  →  THEN [action]
```

Each rule supports up to **5 conditions** (AND logic — all must match) and exactly **one action**.

---

#### Rule Conditions

**Path-based routing** — the most common condition:

```
/api/*          → API target group
/static/*       → CDN proxy target group
/health         → health check endpoint
/ws/*           → WebSocket target group
```

Path matching supports wildcards:
- `?` — matches any single character
- `*` — matches zero or more characters
- Matching is **case-sensitive** by default

**Host-based routing** — uses the HTTP `Host` header:

```
api.myapp.com      → API service target group
admin.myapp.com    → Admin service target group
*.myapp.com        → Wildcard — catches all subdomains
```

> This is the most powerful routing mechanism for microservices. A single ALB serves as the entry point for an entire platform.

**HTTP header conditions** — inspect arbitrary request headers:

```
X-App-Version: v2              → canary target group
X-Internal-Request: true       → internal processing target group
```

**HTTP method conditions:**

```
GET                            → read-only replicas
POST, PUT, PATCH, DELETE       → write-capable services
```

**Query string conditions:**

```
?version=2                     → v2 service target group
?debug=true                    → debug-enabled target group
```

**Source IP conditions:**
- CIDR block matching on the client's source IP
- Useful for restricting admin routes to corporate IP ranges: `10.0.0.0/8`

**Combining conditions (AND logic):**
```
Host = api.myapp.com
  AND Path starts with /v2/
  AND Header X-Region = us-east
→ Route to: US East v2 API target group
```

---

#### Rule Actions

| Action | Description | Key Use Case |
|---|---|---|
| **Forward** | Send to a target group | Standard routing |
| **Weighted Forward** | Distribute % across multiple TGs | Canary / blue-green |
| **Redirect** | Return HTTP redirect from ALB | HTTP→HTTPS, URL normalization |
| **Fixed Response** | Return custom HTTP response directly | Maintenance pages, CORS preflight |
| **Authenticate** | OAuth2/OIDC via Cognito or OIDC IdP | Offload auth from application code |

**Weighted Forward** for canary deployments:
```
TG: api-v1-tasks   → weight: 90
TG: api-v2-tasks   → weight: 10
```
Update weights progressively (80/20 → 50/50 → 0/100) without redeploying anything.

**Fixed Response** example — maintenance window:
```
HTTP 503 | Content-Type: text/html | "Service temporarily unavailable"
```

**Authenticate action** — the ALB handles the full OAuth2/OIDC flow. Unauthenticated requests are redirected to the IdP login page. Only authenticated requests reach your ECS tasks. Authentication state is stored in an encrypted ALB-managed session cookie.

> 📖 **AWS Docs:** [ALB listener rules](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html#listener-rules)  
> 📖 **AWS Docs:** [Path-based routing](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/tutorial-load-balancer-routing.html)  
> 📖 **AWS Docs:** [Host-based routing](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html#host-conditions)  
> 📖 **AWS Docs:** [HTTP header and method conditions](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html#http-header-conditions)  
> 📖 **AWS Docs:** [Weighted target groups — traffic shifting](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html#target-group-routing-configuration)  
> 📖 **AWS Docs:** [ALB authentication with Cognito / OIDC](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-authenticate-users.html)

---

## 3. Target Groups — The Bridge to ECS

A **Target Group** is a logical grouping of targets that the ALB routes traffic to. It defines HOW to communicate with targets and WHAT to send traffic to. Target Groups are **decoupled from listeners and rules** — one target group can be referenced by multiple rules across multiple listeners, even on multiple load balancers.

---

### Target Types

| Target Type | Registration | Network Mode Required | Best For |
|---|---|---|---|
| `instance` | EC2 Instance ID | `bridge` or `host` | ECS on EC2 with dynamic port mapping |
| `ip` | IP address (ENI) | `awsvpc` | ECS Fargate, ECS on EC2 with `awsvpc` ✅ Recommended |
| `lambda` | Lambda function ARN | N/A | Mixed ALB+Lambda architectures |
| `alb` | Nested ALB | N/A | Multi-tier NLB→ALB architectures |

> ✅ **Best Practice:** Always use `target_type = ip` with `awsvpc` network mode. Every task gets its own ENI, its own private IP, and its own security group — full task-level isolation.

**Target Group Protocol Version** — must match your application:

| Protocol Version | Use Case |
|---|---|
| HTTP/1.1 | Standard REST APIs |
| HTTP/2 | HTTP/2 streaming, some microservices |
| gRPC | gRPC microservices — requires ALPN on listener |

> 📖 **AWS Docs:** [Target groups for ALB](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html)  
> 📖 **AWS Docs:** [Target type: IP](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html#target-type)  
> 📖 **AWS Docs:** [gRPC target groups](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html#target-group-protocol-version)

---

### Health Checks — The Most Critical Configuration

Health checks determine if a target is healthy and eligible to receive traffic. **Misconfigured health checks are one of the most common sources of production incidents with ECS.**

| Parameter | Default | Recommended for ECS | Notes |
|---|---|---|---|
| Protocol | HTTP | HTTP | Use HTTPS for end-to-end encryption |
| Path | `/` | `/health` or `/healthz` | Never use `/` — use a dedicated lightweight endpoint |
| Port | `traffic-port` | `traffic-port` | Use a separate port for management if needed |
| Healthy threshold | 3 | **2** | Speeds up new task registration during deployments |
| Unhealthy threshold | 3 | 3 | Don't lower to 1 — causes false positives |
| Timeout | 5s | 5s | Health endpoint must respond within this window |
| Interval | 30s | **10s** | Lower for faster failure detection |
| Success codes | 200 | `200` or `200-299` | Match your app's health endpoint response code |
| Deregistration delay | **300s** | **30s** | ⚠️ Tune this down! 300s makes deployments take 5+ extra minutes |

> ⚠️ **Critical:** The `deregistration_delay` default is **300 seconds**. For most ECS workloads this should be **30–60 seconds**. At the default, every ECS deployment waits an extra 5 minutes for old tasks to drain, even if they're idle.

**Health check path best practices:**
```python
# Example: lightweight FastAPI health endpoint
@app.get("/health")
async def health():
    return {"status": "healthy"}
# Returns 200 immediately — no DB calls, no Bedrock calls, no external deps
```

> 📖 **AWS Docs:** [ALB health checks for target groups](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html)  
> 📖 **AWS Docs:** [Deregistration delay (connection draining)](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html#deregistration-delay)

---

### Stickiness (Session Affinity)

| Stickiness Type | Cookie Name | Managed By | Use Case |
|---|---|---|---|
| LB-generated | `AWSALB` | ALB (encrypted, opaque) | Legacy apps requiring session affinity |
| Application-based | Custom name | Your application | App already manages sessions |
| None (disabled) | — | — | ✅ **Recommended for stateless ECS containers** |

> ✅ **Best Practice:** Stickiness is an anti-pattern for stateless ECS containers. Externalize session state to **ElastiCache (Redis)** and keep containers stateless. Only enable stickiness for legacy applications that cannot be made stateless.

> 📖 **AWS Docs:** [Sticky sessions (session affinity)](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/sticky-sessions.html)

---

## 4. ECS Integration — Deep Dive

### ECS Network Modes and Their Impact on ALB Integration

The ECS task **network mode** is the single most important configuration determining HOW the ALB connects to your containers.

| Network Mode | Launch Type | Target Type | Isolation | Recommended |
|---|---|---|---|---|
| `awsvpc` | EC2 + Fargate | `ip` | Task-level SG ✅ | ✅ **Always use this** |
| `bridge` | EC2 only | `instance` | Instance-level SG | Legacy / EC2 only |
| `host` | EC2 only | `instance` | No isolation | ❌ Almost never used |

**`awsvpc` mode (the modern standard):**
- Every task gets its own ENI with its own private IP from your VPC CIDR
- Container listens on its defined `containerPort` — no host port mapping
- Target group registers the task's ENI IP and container port directly
- Each task has its own security group → full task-level isolation
- Required for Fargate; strongly recommended for EC2 launch type

**`bridge` mode (EC2 legacy):**
- Container runs inside Docker bridge network
- Set `hostPort: 0` in task definition → Docker assigns random ephemeral port (32768–65535)
- ECS automatically registers `host-ip:dynamic-port` with target group
- All containers on same EC2 instance share the same security group

> 📖 **AWS Docs:** [ECS network modes](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-networking.html)  
> 📖 **AWS Docs:** [ECS task networking with awsvpc](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-networking-awsvpc.html)  
> 📖 **AWS Docs:** [Fargate networking](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-task-networking.html)

---

### ECS Service Load Balancer Configuration

When creating or updating an ECS service, you attach a load balancer configuration. The ECS service scheduler uses this to automatically register and deregister tasks with the target group.

```json
{
  "loadBalancers": [
    {
      "targetGroupArn": "arn:aws:elasticloadbalancing:us-east-1:123456789:targetgroup/api-tg/abc123",
      "containerName": "api",
      "containerPort": 8080
    }
  ]
}
```

**Three critical fields — all must be correct:**

| Field | Description | Common Mistake |
|---|---|---|
| `targetGroupArn` | ARN of the ALB target group | Wrong ARN / wrong region |
| `containerName` | Must **exactly** match the container name in the task definition | Case-sensitive mismatch |
| `containerPort` | The port the container actually listens on | Mismatch with task definition `containerPort` |

**ECS automatic registration lifecycle:**
1. Task reaches `RUNNING` state → ECS calls `RegisterTargets` on the target group
2. ALB begins health checks against the new target
3. Target passes health checks → marked `healthy` → begins receiving traffic
4. Task stops for any reason → ECS calls `DeregisterTargets`
5. ALB drains connections for `deregistration_delay` seconds
6. Target removed from rotation

> ✅ You **never** manually register or deregister targets during normal operations — ECS manages this entirely.

**Multiple load balancers per service** — an ECS service can be registered with up to **5 target groups** simultaneously. Use cases:
- Same service reachable on both an internet-facing and an internal ALB
- Service exposing multiple ports (e.g., HTTP on 8080 + metrics on 9090)

> 📖 **AWS Docs:** [ECS service load balancing](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-load-balancing.html)  
> 📖 **AWS Docs:** [Registering/deregistering ECS tasks with target groups](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/register-multiple-targetgroups.html)  
> 📖 **AWS Docs:** [ECS task definition — containerPort](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html#container_definition_portmappings)

---

### The Full Request Path

```
User Browser
    │
    ▼ DNS: api.myapp.com → Route 53 → ALB DNS → ALB Node IP (in user's AZ)
    │
    ▼ TLS Handshake (TCP:443) → ALB presents ACM cert → TLS terminates at ALB
    │
    ▼ ALB parses HTTP: method, headers, Host, path, query string
    │
    ▼ ALB evaluates listener rules (priority order, first match wins)
    │    Rule 1: Host=api.myapp.com AND Path=/v1/* → MATCH
    │    Action: Forward to Target Group "tg-api-v1"
    │
    ▼ ALB checks healthy registered targets in tg-api-v1
    │    10.0.1.45:8080  ✅ healthy
    │    10.0.2.67:8080  ✅ healthy
    │    10.0.3.89:8080  ✅ healthy
    │
    ▼ ALB selects target via load balancing algorithm (round-robin or LOR)
    │    Selected: 10.0.2.67:8080 (ECS task ENI in private subnet)
    │
    ▼ ALB opens/reuses HTTP connection to ECS task
    │    Adds headers: X-Forwarded-For, X-Forwarded-Proto, X-Forwarded-Port
    │
    ▼ ECS task (Python app) processes request
    │    → Calls Amazon Bedrock
    │    → Queries RDS / Aurora
    │    → Reads from S3 Knowledge Base
    │
    ▼ ECS task returns HTTP response to ALB
    │
    ▼ ALB returns response to client over original TLS connection
```

**Headers added by ALB to upstream requests:**

| Header | Value | Use In App |
|---|---|---|
| `X-Forwarded-For` | Original client IP | Logging, rate limiting, geo-fencing |
| `X-Forwarded-Proto` | `https` or `http` | HTTPS redirect enforcement in app |
| `X-Forwarded-Port` | `443` | URL construction |
| `X-Amzn-Trace-Id` | AWS X-Ray trace ID | Distributed tracing propagation |

---

### Security Group Chain — The Most Misunderstood Part

The security group configuration follows a strict **chain of trust**. This is where most production misconfiguration occurs.

```
Internet (0.0.0.0/0)
    │
    ▼ TCP 443, TCP 80
┌─────────────────────────────┐
│  SG-ALB (ALB Security Group)│
│  Inbound:                   │
│    TCP 443 from 0.0.0.0/0  │
│    TCP 80  from 0.0.0.0/0  │
│  Outbound:                  │
│    TCP {container-port}     │
│    to SG-ECS               │
└──────────────┬──────────────┘
               │ TCP {container-port}
               ▼ Source: SG-ALB (by SG ID, not CIDR)
┌─────────────────────────────┐
│  SG-ECS (ECS Task SG)       │
│  Inbound:                   │
│    TCP 8080 FROM SG-ALB    │  ← Reference by SG ID
│  Outbound:                  │
│    TCP 443 to 0.0.0.0/0    │  ← Bedrock, S3, internet
│    TCP 5432 to SG-RDS      │  ← Database
└─────────────────────────────┘
```

> ⚠️ **Critical Pattern:** The ECS task's inbound rule must reference the **ALB's security group ID** as the source — NOT a CIDR block like `10.0.0.0/16`. If you use a CIDR, anything in the VPC can reach your tasks directly, bypassing the ALB entirely.

> ✅ With `awsvpc` mode, health check traffic from ALB nodes is automatically permitted because the ALB nodes are members of `SG-ALB`, which is the source in the ECS task SG rule. No separate health check port rule needed.

> 📖 **AWS Docs:** [Security groups for ALB](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-update-security-groups.html)  
> 📖 **AWS Docs:** [ECS security groups with awsvpc](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-networking-awsvpc.html#awsvpc-security-groups)  
> 📖 **AWS Docs:** [VPC security group rules](https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html)

---

### ECS Service Deployment and the ALB

**Rolling Update Deployment (default) — key parameters:**

| Parameter | Description | Production Setting |
|---|---|---|
| `minimumHealthyPercent` | Min % of desired tasks that must be healthy during deployment | `100` — no downtime |
| `maximumPercent` | Max % of tasks (new + old) that can run simultaneously | `200` — doubles capacity during deploy |
| `healthCheckGracePeriodSeconds` | Window before ECS checks ALB health for new tasks | Startup time + 15–30s buffer |

> **Example:** App takes ~20s to start → set `healthCheckGracePeriodSeconds = 45`. If set too low, ECS kills starting tasks that haven't passed health checks yet, causing a deployment loop.

**Deployment Circuit Breaker** — always enable in production:
```hcl
deployment_circuit_breaker {
  enable   = true
  rollback = true  # Auto-rollback if deployment fails health checks
}
```

**Blue/Green Deployment (CodeDeploy integration):**

```
                    ┌─────────────────────┐
                    │  ALB Listener Rule  │
                    └──────────┬──────────┘
                               │
              ┌────────────────┴────────────────┐
              ▼                                 ▼
     ┌──────────────────┐             ┌──────────────────┐
     │  TG: Blue (v1)   │             │  TG: Green (v2)  │
     │  Current traffic │             │  New deployment  │
     └──────────────────┘             └──────────────────┘
```

**Traffic shifting strategies:**

| Strategy | Behavior |
|---|---|
| `ECSAllAtOnce` | Instant 100% cutover |
| `ECSLinear10PercentEvery1Minute` | Gradual 10%/min shift over 10 minutes |
| `ECSCanary10Percent5Minutes` | 10% canary for 5 min, then 90% |

Rollback: flip the ALB listener rule back to the blue target group. Original blue tasks remain running during the termination wait period.

> 📖 **AWS Docs:** [ECS rolling update deployments](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-ecs.html)  
> 📖 **AWS Docs:** [ECS blue/green deployment with CodeDeploy](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-bluegreen.html)  
> 📖 **AWS Docs:** [ECS deployment circuit breaker](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-circuit-breaker.html)  
> 📖 **AWS Docs:** [Health check grace period](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service_definition_parameters.html#sd-healthCheckGracePeriodSeconds)  
> 📖 **AWS Docs:** [CodeDeploy traffic shifting options](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-configurations.html#deployment-configuration-lambda)

---

## 5. Advanced ALB Patterns for ECS Architectures

### Pattern 1: Path-Based Microservices Routing

A single ALB as the entry point for an entire microservices platform, fanning out to multiple ECS services via path routing:

```
ALB Listener HTTPS:443
├── Rule 1  (priority 10):  Path /api/users/*   → TG: users-svc-tg    → ECS: users-service
├── Rule 2  (priority 20):  Path /api/orders/*  → TG: orders-svc-tg   → ECS: orders-service
├── Rule 3  (priority 30):  Path /api/search/*  → TG: search-svc-tg   → ECS: search-service
├── Rule 4  (priority 40):  Path /ws/*          → TG: websocket-tg    → ECS: ws-service
├── Rule 5  (priority 50):  Path /static/*      → TG: cdn-proxy-tg    → ECS: nginx-static
└── Default Rule:           Forward             → TG: frontend-tg     → ECS: react-frontend
```

Each ECS service has its own task definition, scaling policy, container config. Deploy and scale each microservice independently without touching others.

---

### Pattern 2: Host-Based Multi-Tenant / Multi-Environment Routing

```
ALB Listener HTTPS:443
├── Rule 1: Host api.prod.myapp.com      → TG: api-prod-tg      → ECS: production cluster
├── Rule 2: Host api.staging.myapp.com   → TG: api-staging-tg   → ECS: staging cluster
├── Rule 3: Host admin.myapp.com         → TG: admin-tg         → ECS: admin-service
└── Rule 4: Host *.myapp.com             → TG: catch-all-tg     → ECS: 404-handler
```

---

### Pattern 3: Header-Based Canary Routing

For canary deployments without CodeDeploy — route specific users to new versions via HTTP headers:

```
Rule 1 (priority 1):
  Condition: Header X-Canary-User = "true"
  Action: Forward to TG: api-v2-tasks (new image)

Rule 2 (priority 100):
  Condition: (catch-all)
  Action: Forward to TG: api-v1-tasks (stable image)
```

Your API gateway or client stamps specific users with the `X-Canary-User` header. Only those users hit the canary — zero traffic splitting infrastructure needed.

---

### Pattern 4: Weighted Target Groups for Gradual Traffic Shifting

```
Rule Action: Weighted Forward
  TG: api-v1-tasks   → weight: 90  (90% of traffic)
  TG: api-v2-tasks   → weight: 10  (10% of traffic)
```

Progressively update weights: `90/10 → 80/20 → 50/50 → 0/100`. No redeployment needed — update the ALB rule only.

---

### Pattern 5: WebSocket Support

ALB natively supports WebSocket upgrades. When a client sends `Upgrade: websocket`, the ALB upgrades and maintains a persistent bidirectional connection.

**Critical configurations for WebSocket ECS services:**

| Setting | Default | WebSocket Recommended |
|---|---|---|
| ALB idle timeout | 60s | **3600s** (max) |
| Deregistration delay | 300s | High enough to drain open connections |
| Load balancing algorithm | Round-robin | LOR (long-lived connections create imbalance) |
| Stickiness | Disabled | Consider enabling — clients reconnect to same task |

> 📖 **AWS Docs:** [ALB WebSocket support](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html#target-group-routing-configuration)

---

### Pattern 6: gRPC Microservices on ALB with ECS

```
Listener:       HTTPS:443 — ALPN policy: h2 (HTTP/2)
Target Group:   Protocol: HTTP | Protocol Version: gRPC
Health Check:   gRPC health check protocol + path /grpc.health.v1.Health/Check
Algorithm:      Least Outstanding Requests (not round-robin)
Container:      awsvpc mode, port 50051
```

> 📖 **AWS Docs:** [gRPC on ALB](https://aws.amazon.com/blogs/aws/new-application-load-balancer-support-for-end-to-end-http-2-and-grpc/)

---

### Pattern 7: Internal ALB for ECS East-West Traffic

For service-to-service communication within ECS:

```
internet-facing ALB (public subnets)
    └── ECS Service A (private subnet)
            │
            ▼ HTTP call to internal ALB DNS
    internal ALB (private subnets, internal scheme)
            └── ECS Service B (private subnet)
```

Benefits: full ALB routing features, health checking, and transparent backend swapping for east-west traffic.

> ✅ **Note:** For low-latency east-west traffic, also evaluate **ECS Service Connect** (AWS Cloud Map + Envoy proxy). Internal ALB provides more routing flexibility; Service Connect provides better observability and lower latency.

> 📖 **AWS Docs:** [ECS Service Connect](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-connect.html)  
> 📖 **AWS Docs:** [Internal load balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-application-load-balancer.html#configure-load-balancer)

---

## 6. IaC — Terraform Reference Patterns

```hcl
# ─────────────────────────────────────────────
# ALB
# ─────────────────────────────────────────────
resource "aws_lb" "main" {
  name               = "my-app-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = var.public_subnet_ids

  enable_deletion_protection = true
  idle_timeout               = 60

  access_logs {
    bucket  = aws_s3_bucket.alb_logs.bucket
    prefix  = "alb"
    enabled = true  # Always enable — essential for debugging & audit
  }

  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# ─────────────────────────────────────────────
# ALB Security Group
# ─────────────────────────────────────────────
resource "aws_security_group" "alb" {
  name   = "alb-sg"
  vpc_id = var.vpc_id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# ─────────────────────────────────────────────
# ECS Task Security Group — references ALB SG by ID
# ─────────────────────────────────────────────
resource "aws_security_group" "ecs_tasks" {
  name   = "ecs-tasks-sg"
  vpc_id = var.vpc_id

  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]  # Reference by SG ID, not CIDR
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]  # Outbound to Bedrock, S3, etc.
  }
}

# ─────────────────────────────────────────────
# HTTPS Listener with TLS 1.3
# ─────────────────────────────────────────────
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"  # TLS 1.3 preferred
  certificate_arn   = aws_acm_certificate.main.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.frontend.arn
  }
}

# ─────────────────────────────────────────────
# HTTP → HTTPS Redirect (port 80)
# ─────────────────────────────────────────────
resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.main.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

# ─────────────────────────────────────────────
# Target Group — API ECS Service (awsvpc / Fargate)
# ─────────────────────────────────────────────
resource "aws_lb_target_group" "api" {
  name        = "api-tg"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = var.vpc_id
  target_type = "ip"  # Required for awsvpc / Fargate

  health_check {
    enabled             = true
    healthy_threshold   = 2        # Faster task registration
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 10       # Faster failure detection (default: 30)
    path                = "/health"
    port                = "traffic-port"
    matcher             = "200"
  }

  deregistration_delay = 30  # ⚠️ Tune down from 300s default!

  stickiness {
    type    = "lb_cookie"
    enabled = false  # Stateless containers — no stickiness needed
  }

  tags = {
    Service = "api"
  }
}

# ─────────────────────────────────────────────
# Listener Rule — Path-based routing to API
# ─────────────────────────────────────────────
resource "aws_lb_listener_rule" "api" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 100

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }

  condition {
    path_pattern {
      values = ["/api/*"]
    }
  }
}

# ─────────────────────────────────────────────
# Listener Rule — Weighted canary (v1/v2 split)
# ─────────────────────────────────────────────
resource "aws_lb_listener_rule" "api_canary" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 90

  action {
    type = "forward"
    forward {
      target_group {
        arn    = aws_lb_target_group.api_v1.arn
        weight = 90
      }
      target_group {
        arn    = aws_lb_target_group.api_v2.arn
        weight = 10
      }
      stickiness {
        enabled  = false
        duration = 1
      }
    }
  }

  condition {
    path_pattern {
      values = ["/api/*"]
    }
  }
}

# ─────────────────────────────────────────────
# ECS Service — attached to ALB Target Group
# ─────────────────────────────────────────────
resource "aws_ecs_service" "api" {
  name            = "api-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.api.arn
  desired_count   = 3
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false  # Private subnet — NAT GW handles outbound
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.api.arn
    container_name   = "api"   # Must EXACTLY match task definition container name
    container_port   = 8080    # Must match containerPort in task definition
  }

  deployment_minimum_healthy_percent = 100  # Zero downtime
  deployment_maximum_percent         = 200  # Double capacity during deploy

  deployment_circuit_breaker {
    enable   = true
    rollback = true  # Auto-rollback on failed deployment
  }

  health_check_grace_period_seconds = 45  # Tune to app startup time + buffer

  deployment_controller {
    type = "ECS"  # Use "CODE_DEPLOY" for blue/green with CodeDeploy
  }

  depends_on = [aws_lb_listener_rule.api]  # Ensure rule exists before service
}
```

> 📖 **Terraform Registry:** [aws_lb](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb)  
> 📖 **Terraform Registry:** [aws_lb_listener](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_listener)  
> 📖 **Terraform Registry:** [aws_lb_target_group](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_target_group)  
> 📖 **Terraform Registry:** [aws_lb_listener_rule](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_listener_rule)  
> 📖 **Terraform Registry:** [aws_ecs_service](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ecs_service)

---

## 7. Observability — What to Monitor

### Key CloudWatch Metrics (ALB Namespace: `AWS/ApplicationELB`)

| Metric | Alarm Threshold | What It Means |
|---|---|---|
| `HTTPCode_ELB_5XX_Count` | Any non-zero | ALB-level errors — not from your app |
| `HTTPCode_Target_5XX_Count` | Exceeds SLA | Application error rate — your containers |
| `TargetResponseTime` (p99) | Exceeds SLA | End-to-end latency (ALB → target → ALB) |
| `UnHealthyHostCount` | > 0 | Unhealthy ECS tasks — immediate investigation |
| `TargetConnectionErrorCount` | Any non-zero | ALB can't reach targets — SG or task failure |
| `RejectedConnectionCount` | Any non-zero | ALB at surge queue limit — scale or investigate |
| `ActiveConnectionCount` | Capacity planning | Current open connections to ALB |
| `RequestCount` | Capacity planning | Total requests per interval |

### ALB Access Logs

Always enable ALB access logs to S3. Each log line contains:

```
timestamp | client:port | target:port | request_processing_time |
target_processing_time | response_processing_time | elb_status_code |
target_status_code | received_bytes | sent_bytes | request (verb+uri+proto) |
user_agent | ssl_cipher | ssl_protocol | target_group_arn | trace_id
```

Essential for: debugging latency issues, security forensics, cost attribution per service.

```hcl
# S3 bucket for ALB access logs — requires specific bucket policy
resource "aws_s3_bucket_policy" "alb_logs" {
  bucket = aws_s3_bucket.alb_logs.id
  policy = jsonencode({
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "elasticloadbalancing.amazonaws.com" }
      Action    = "s3:PutObject"
      Resource  = "${aws_s3_bucket.alb_logs.arn}/alb/AWSLogs/${data.aws_caller_identity.current.account_id}/*"
    }]
  })
}
```

### Distributed Tracing with X-Ray

Enable X-Ray tracing on the ALB — it stamps the `X-Amzn-Trace-Id` header on every request. Your ECS Python application picks this up via the X-Ray SDK and propagates the trace ID through calls to Bedrock, RDS, and S3 — giving end-to-end distributed tracing across the full request path.

```python
# Python ECS app — X-Ray middleware (FastAPI example)
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.ext.fastapi.middleware import XRayMiddleware

app.add_middleware(XRayMiddleware, recorder=xray_recorder)
xray_recorder.configure(service='api-service')
```

> 📖 **AWS Docs:** [ALB CloudWatch metrics](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-cloudwatch-metrics.html)  
> 📖 **AWS Docs:** [ALB access logs](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/enable-access-logging.html)  
> 📖 **AWS Docs:** [ALB request tracing with X-Ray](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-request-tracing.html)  
> 📖 **AWS Docs:** [X-Ray SDK for Python](https://docs.aws.amazon.com/xray/latest/devguide/xray-sdk-python.html)  
> 📖 **AWS Docs:** [ECS Container Insights](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cloudwatch-container-insights.html)

---

## 8. AWS Reference Links

### Core ALB Documentation
| Topic | Link |
|---|---|
| ALB Overview | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html |
| Create an ALB | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-application-load-balancer.html |
| Listeners | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html |
| Listener Rules | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html#listener-rules |
| Target Groups | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html |
| Health Checks | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html |
| Sticky Sessions | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/sticky-sessions.html |
| ALB Quotas (limits) | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-limits.html |
| ALB Security Groups | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-update-security-groups.html |
| HTTPS Listeners / ACM | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-https-listener.html |
| SNI — Multiple Certs | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/https-listener-certificates.html |
| TLS Security Policies | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/describe-ssl-policies.html |
| gRPC on ALB (blog) | https://aws.amazon.com/blogs/aws/new-application-load-balancer-support-for-end-to-end-http-2-and-grpc/ |
| Authenticate with Cognito | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-authenticate-users.html |
| ALB Access Logs | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/enable-access-logging.html |
| ALB CloudWatch Metrics | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-cloudwatch-metrics.html |
| X-Ray Request Tracing | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-request-tracing.html |

### ECS + Load Balancing
| Topic | Link |
|---|---|
| ECS Service Load Balancing | https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-load-balancing.html |
| ECS Load Balancer Types | https://docs.aws.amazon.com/AmazonECS/latest/developerguide/load-balancer-types.html |
| ECS awsvpc Network Mode | https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-networking-awsvpc.html |
| ECS Task Networking Overview | https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-networking.html |
| Register Multiple Target Groups | https://docs.aws.amazon.com/AmazonECS/latest/developerguide/register-multiple-targetgroups.html |
| ECS Rolling Update Deployments | https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-ecs.html |
| ECS Blue/Green (CodeDeploy) | https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-bluegreen.html |
| ECS Deployment Circuit Breaker | https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-circuit-breaker.html |
| ECS Health Check Grace Period | https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service_definition_parameters.html |
| Fargate Networking | https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-task-networking.html |
| ECS Service Connect | https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-connect.html |
| ECS Container Insights | https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cloudwatch-container-insights.html |
| ECS Task Definition — Port Mappings | https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html#container_definition_portmappings |

### Networking & Security
| Topic | Link |
|---|---|
| VPC Security Group Rules | https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html |
| AWS Certificate Manager (ACM) | https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html |
| Route 53 with ALB | https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-to-elb-load-balancer.html |
| NLB + ALB (static IPs for ALB) | https://aws.amazon.com/blogs/networking-and-content-delivery/using-static-ip-addresses-for-application-load-balancers/ |
| VPC Endpoints (PrivateLink) | https://docs.aws.amazon.com/vpc/latest/privatelink/what-is-privatelink.html |
| AWS X-Ray SDK for Python | https://docs.aws.amazon.com/xray/latest/devguide/xray-sdk-python.html |

### Terraform Registry
| Resource | Link |
|---|---|
| aws_lb | https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb |
| aws_lb_listener | https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_listener |
| aws_lb_listener_rule | https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_listener_rule |
| aws_lb_target_group | https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_target_group |
| aws_ecs_service | https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ecs_service |
| aws_ecs_task_definition | https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ecs_task_definition |
| aws_security_group | https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group |

### AWS Well-Architected & Best Practices
| Topic | Link |
|---|---|
| Well-Architected Framework — Reliability | https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html |
| ECS Best Practices Guide | https://docs.aws.amazon.com/AmazonECS/latest/bestpracticesguide/intro.html |
| ECS Load Balancing Best Practices | https://docs.aws.amazon.com/AmazonECS/latest/bestpracticesguide/load-balancing.html |
| ALB Pricing | https://aws.amazon.com/elasticloadbalancing/pricing/ |
| ECS Pricing | https://aws.amazon.com/ecs/pricing/ |

---

## Architecture Summary

```
                         ┌───────────────────────────────────────────┐
                         │              Route 53 (DNS)                │
                         └────────────────────┬──────────────────────┘
                                              │
                         ┌────────────────────▼──────────────────────┐
                         │  Internet-Facing ALB (public subnets)     │
                         │                                           │
                         │  Listener HTTPS:443                       │
                         │  ┌─────────────────────────────────────┐  │
                         │  │ Rules (evaluated in priority order)  │  │
                         │  │  /api/users/*  → TG: users-svc      │  │
                         │  │  /api/orders/* → TG: orders-svc     │  │
                         │  │  /ws/*         → TG: ws-svc         │  │
                         │  │  default       → TG: frontend       │  │
                         │  └─────────────────────────────────────┘  │
                         └────────────────────┬──────────────────────┘
                                              │ SG-ALB → SG-ECS (port 8080)
                         ┌────────────────────▼──────────────────────┐
                         │          VPC Private Subnets               │
                         │                                           │
                         │  ┌──────────┐ ┌──────────┐ ┌──────────┐  │
                         │  │ECS Task  │ │ECS Task  │ │ECS Task  │  │
                         │  │(AZ-1)    │ │(AZ-2)    │ │(AZ-3)    │  │
                         │  │awsvpc    │ │awsvpc    │ │awsvpc    │  │
                         │  │own ENI   │ │own ENI   │ │own ENI   │  │
                         │  └────┬─────┘ └────┬─────┘ └────┬─────┘  │
                         └───────┼────────────┼────────────┼─────────┘
                                 │            │            │
                    ┌────────────▼──┐  ┌──────▼──────┐  ┌─▼──────────────┐
                    │ Amazon Bedrock│  │  Amazon RDS  │  │   Amazon S3    │
                    │ (Gen AI / RAG)│  │  (Database)  │  │ (KB / Objects) │
                    └───────────────┘  └─────────────┘  └────────────────┘
```

---

*Generated for AWS DevOps Engineers — Principal Level*  
*For issues or contributions, open a PR.*
