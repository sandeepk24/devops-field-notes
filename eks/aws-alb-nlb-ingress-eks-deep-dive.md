# AWS EKS Frontend Connectivity: ALB · NLB · Ingress — End-to-End Deep Dive
> **Audience:** DevOps Engineers · Senior DevOps Engineers · Principal DevOps Engineers  
> **Scope:** How ALB, NLB, and Kubernetes Ingress connect to EKS Clusters for reliable, secure, production-grade frontend traffic  
> **Context:** Migration/comparison from ECS service-based load balancing → EKS Kubernetes-native load balancing  
> **Last Updated:** 2026

---

## Table of Contents

1. [ECS vs EKS Load Balancing — The Mental Model Shift](#1-ecs-vs-eks-load-balancing--the-mental-model-shift)
2. [AWS Load Balancer Controller — The Foundation](#2-aws-load-balancer-controller--the-foundation)
3. [ALB Ingress on EKS — Deep Dive](#3-alb-ingress-on-eks--deep-dive)
   - [Ingress Resource Anatomy](#ingress-resource-anatomy)
   - [Path-Based Routing](#path-based-routing)
   - [Host-Based Routing](#host-based-routing)
   - [IngressClass and Multiple Controllers](#ingressclass-and-multiple-controllers)
4. [NLB with EKS — Deep Dive](#4-nlb-with-eks--deep-dive)
5. [Security Groups for Pods (SGP)](#5-security-groups-for-pods-sgp)
6. [Target Types: Instance vs IP Mode](#6-target-types-instance-vs-ip-mode)
7. [EKS Service Types and Their Load Balancer Behavior](#7-eks-service-types-and-their-load-balancer-behavior)
8. [Advanced Routing Patterns on EKS](#8-advanced-routing-patterns-on-eks)
9. [Certificates and TLS on EKS](#9-certificates-and-tls-on-eks)
10. [Terraform IaC Reference](#10-iac--terraform-reference-patterns)
11. [Helm / Kubernetes YAML Reference](#11-kubernetes-yaml--helm-reference)
12. [Observability on EKS](#12-observability-on-eks)
13. [AWS Reference Links](#13-aws-reference-links)

---

## 1. ECS vs EKS Load Balancing — The Mental Model Shift

This is the most important concept before diving in. In ECS, load balancing is **AWS-native** — you configure target groups and ECS registers tasks automatically. In EKS, load balancing is **Kubernetes-mediated** — AWS resources are created and managed by a Kubernetes controller that watches Kubernetes API objects.

| Concept | ECS Equivalent | EKS Equivalent |
|---|---|---|
| Unit of traffic | ECS Task | Kubernetes Pod |
| Load balancer registration | ECS Service auto-registers tasks to TG | AWS LBC watches Ingress/Service → creates/manages ALB or NLB |
| Routing config | ALB Listener Rules (AWS Console/IaC) | Kubernetes `Ingress` resource (YAML) |
| Service discovery | ECS Service DNS / Cloud Map | Kubernetes `Service` (ClusterIP, NodePort, LoadBalancer) |
| Health check config | Target Group health check settings | Kubernetes readinessProbe + LBC annotations |
| Network isolation | ECS Task security groups (awsvpc) | Security Groups for Pods (SGP) or node-level SGs |
| Controller plane | AWS ECS Service Scheduler | AWS Load Balancer Controller (runs inside EKS cluster) |

> **Key Insight:** In EKS, you write Kubernetes YAML — the AWS Load Balancer Controller translates that into real AWS ALB/NLB resources. AWS resources are **side effects** of Kubernetes objects, not the primary configuration layer.

---

## 2. AWS Load Balancer Controller — The Foundation

The **AWS Load Balancer Controller (LBC)** is a Kubernetes controller that runs as a Deployment inside your EKS cluster. It is the bridge between Kubernetes objects and AWS load balancer resources.

> 📖 **AWS Docs:** [AWS Load Balancer Controller — Installation](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html)  
> 📖 **AWS Docs:** [LBC GitHub Repository](https://github.com/kubernetes-sigs/aws-load-balancer-controller)

### What it watches and what it creates

| Kubernetes Object | AWS Resource Created |
|---|---|
| `Ingress` (class: alb) | Application Load Balancer + Listeners + Target Groups + Rules |
| `Service` (type: LoadBalancer) with NLB annotations | Network Load Balancer + Listeners + Target Groups |
| `TargetGroupBinding` (CRD) | Binds existing TG to Kubernetes pods directly |

### Architecture Overview

> 🎨 **Architecture Diagram — EKS + ALB Ingress Overview:**
>
> [![EKS ALB Architecture](https://design.canva.ai/OKERF6XXENQhvId)](https://www.canva.com/d/2FR4WftgVwXPRAa)
>
> *Click to open and edit in Canva*

```
┌─────────────────────────────────────────────────────────────────────┐
│                         AWS Account                                  │
│                                                                      │
│  Route 53 ──► ALB (public subnets) ──► EKS Cluster (private subnets)│
│                    ▲                          │                      │
│                    │                          │ watches              │
│             AWS LBC creates                   │                      │
│                    │                          ▼                      │
│              ┌─────┴───────────────────────────────────────┐         │
│              │  EKS Control Plane (AWS Managed)             │         │
│              │  kube-apiserver / etcd / scheduler           │         │
│              └─────────────────────────────────────────────┘         │
│                          │                                           │
│              ┌───────────▼──────────────────────────────────┐        │
│              │  Worker Node Group / Fargate Profile          │        │
│              │  ┌──────────────────────────────────────────┐│        │
│              │  │  AWS Load Balancer Controller (Pod)      ││        │
│              │  │  Watches: Ingress + Service objects      ││        │
│              │  │  Creates: ALB/NLB/TG via AWS API         ││        │
│              │  └──────────────────────────────────────────┘│        │
│              │  ┌────────┐  ┌────────┐  ┌────────┐          │        │
│              │  │ Pod    │  │ Pod    │  │ Pod    │          │        │
│              │  │ app:   │  │ app:   │  │ app:   │          │        │
│              │  │ users  │  │ orders │  │ search │          │        │
│              │  └────────┘  └────────┘  └────────┘          │        │
│              └──────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────┘
```

### Installing the AWS Load Balancer Controller

The LBC requires:
1. An **OIDC provider** associated with the EKS cluster (for IAM Roles for Service Accounts — IRSA)
2. An **IAM policy** granting the LBC permission to create/manage ALB, NLB, target groups, security groups
3. A **Kubernetes Service Account** annotated with the IAM role ARN
4. The LBC deployed via Helm

```bash
# 1. Create OIDC provider for the cluster
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --region us-east-1 \
  --approve

# 2. Download the IAM policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json

# 3. Create the IAM policy
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# 4. Create IAM role + service account (IRSA)
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn arn:aws:iam::ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# 5. Install via Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=vpc-XXXXXXXX
```

> 📖 **AWS Docs:** [IAM Roles for Service Accounts (IRSA)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)  
> 📖 **AWS Docs:** [LBC IAM policy](https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json)  
> 📖 **Helm Chart:** [eks/aws-load-balancer-controller](https://artifacthub.io/packages/helm/aws/aws-load-balancer-controller)

---

## 3. ALB Ingress on EKS — Deep Dive

When the LBC sees a Kubernetes `Ingress` resource with `ingressClassName: alb`, it:
1. Creates a real AWS ALB in the subnets you annotate
2. Creates an HTTPS/HTTP Listener on the ALB
3. Creates a Target Group per Kubernetes Service referenced in the Ingress rules
4. Creates Listener Rules mapping paths/hosts to Target Groups
5. Registers Pod IPs directly as targets (IP mode) or Node IPs + NodePort (instance mode)
6. Continuously reconciles — Ingress changes → ALB changes, automatically

> 🎨 **Ingress Routing Diagram:**
>
> [![EKS Ingress Routing](https://design.canva.ai/E42vf1-zwazLqeu)](https://www.canva.com/d/w4wivNpQ3B6cdIt)
>
> *Click to open and edit in Canva*

---

### Ingress Resource Anatomy

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  namespace: production
  annotations:
    # ── Core LBC annotations ─────────────────────────────────────
    kubernetes.io/ingress.class: alb                         # Use ALB (legacy annotation)
    alb.ingress.kubernetes.io/scheme: internet-facing        # or 'internal'
    alb.ingress.kubernetes.io/target-type: ip                # 'ip' (recommended) or 'instance'

    # ── Subnets ──────────────────────────────────────────────────
    alb.ingress.kubernetes.io/subnets: subnet-pub-1a,subnet-pub-1b,subnet-pub-1c

    # ── TLS / Certificate ────────────────────────────────────────
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:ACCOUNT:certificate/XXXX
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS13-1-2-2021-06
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'             # Force HTTP → HTTPS redirect

    # ── Security Groups ──────────────────────────────────────────
    alb.ingress.kubernetes.io/security-groups: sg-alb-XXXX   # Custom ALB security group

    # ── Health Check ─────────────────────────────────────────────
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '10'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '3'
    alb.ingress.kubernetes.io/success-codes: '200'

    # ── Load Balancer Attributes ─────────────────────────────────
    alb.ingress.kubernetes.io/load-balancer-attributes: |
      idle_timeout.timeout_seconds=60,
      access_logs.s3.enabled=true,
      access_logs.s3.bucket=my-alb-logs-bucket,
      access_logs.s3.prefix=eks-alb

    # ── Target Group Attributes ───────────────────────────────────
    alb.ingress.kubernetes.io/target-group-attributes: |
      deregistration_delay.timeout_seconds=30,
      slow_start.duration_seconds=0

    # ── WAF (optional) ───────────────────────────────────────────
    alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:us-east-1:ACCOUNT:regional/webacl/XXXX/YYYY

    # ── Tags ─────────────────────────────────────────────────────
    alb.ingress.kubernetes.io/tags: Environment=production,Team=platform

spec:
  ingressClassName: alb
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /api/users
            pathType: Prefix
            backend:
              service:
                name: users-service
                port:
                  number: 80
          - path: /api/orders
            pathType: Prefix
            backend:
              service:
                name: orders-service
                port:
                  number: 80
    - host: admin.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 80
```

---

### Path-Based Routing

Path-based routing on EKS/ALB works identically to the ECS pattern but is declared in Kubernetes YAML instead of AWS console/IaC.

```
Single ALB Ingress
├── host: api.myapp.com
│   ├── /api/users/*      → Service: users-svc:80    → Pod IPs: 10.0.1.12, 10.0.2.34
│   ├── /api/orders/*     → Service: orders-svc:80   → Pod IPs: 10.0.1.56, 10.0.3.78
│   └── /api/search/*     → Service: search-svc:80   → Pod IPs: 10.0.2.90, 10.0.3.11
└── host: app.myapp.com
    └── /                 → Service: frontend-svc:80  → Pod IPs: 10.0.1.22, 10.0.2.44
```

**`pathType` options — critical to understand:**

| pathType | Behavior | Example |
|---|---|---|
| `Exact` | Exact path match only | `/api/users` matches ONLY `/api/users` |
| `Prefix` | Matches path prefix (recommended) | `/api/users` matches `/api/users`, `/api/users/123` |
| `ImplementationSpecific` | LBC-specific behavior — maps to ALB wildcard `/*` | Use when you need ALB wildcard matching |

---

### Host-Based Routing

Host-based routing is declared at the `rules[].host` field level. The LBC translates each host into an ALB listener rule condition.

```yaml
spec:
  rules:
    - host: api.myapp.com       # → ALB rule: Host header = api.myapp.com
      http:
        paths: [...]
    - host: admin.myapp.com     # → ALB rule: Host header = admin.myapp.com
      http:
        paths: [...]
    - host: "*.myapp.com"       # → ALB rule: Host header wildcard
      http:
        paths: [...]
```

---

### IngressClass and Multiple Controllers

On EKS you may run multiple ingress controllers (ALB + NGINX simultaneously). The `ingressClassName` field on the Ingress object, or the `kubernetes.io/ingress.class` annotation, determines which controller picks it up.

```yaml
# Using the modern IngressClass API (preferred)
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: alb
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: ingress.k8s.aws/alb
```

```yaml
# Ingress uses IngressClass
spec:
  ingressClassName: alb    # ← LBC picks this up
```

```yaml
# NGINX picks up ingresses with ingressClassName: nginx
spec:
  ingressClassName: nginx  # ← NGINX ingress controller picks this up
```

> 📖 **AWS Docs:** [ALB Ingress annotations reference](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.11/guide/ingress/annotations/)  
> 📖 **AWS Docs:** [Ingress — path types](https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types)  
> 📖 **AWS Docs:** [IngressClass](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class)  
> 📖 **AWS Docs:** [LBC — multiple Ingress resources sharing one ALB](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.11/guide/ingress/annotations/#group.name)

---

## 4. NLB with EKS — Deep Dive

For high-throughput, low-latency, or TCP/UDP workloads on EKS, the AWS Load Balancer Controller creates an NLB from a Kubernetes **Service of type `LoadBalancer`** with NLB annotations.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-tcp-service
  namespace: production
  annotations:
    # ── NLB vs CLB ──────────────────────────────────────────────
    service.beta.kubernetes.io/aws-load-balancer-type: "external"     # Use LBC, not legacy in-tree
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip" # Pod IPs directly (recommended)
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"

    # ── Subnets ──────────────────────────────────────────────────
    service.beta.kubernetes.io/aws-load-balancer-subnets: subnet-pub-1a,subnet-pub-1b

    # ── TLS termination on NLB ───────────────────────────────────
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:us-east-1:ACCOUNT:certificate/XXXX
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "tcp"

    # ── Static IP / Elastic IP (NLB strength) ───────────────────
    service.beta.kubernetes.io/aws-load-balancer-eip-allocations: eipalloc-XXXX,eipalloc-YYYY

    # ── Source IP Preservation ───────────────────────────────────
    service.beta.kubernetes.io/aws-load-balancer-target-group-attributes: |
      preserve_client_ip.enabled=true

    # ── Health Check ─────────────────────────────────────────────
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: "/health"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: "HTTP"

    # ── Cross-zone ───────────────────────────────────────────────
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"

spec:
  type: LoadBalancer
  selector:
    app: my-tcp-app
  ports:
    - name: https
      port: 443
      targetPort: 8443
      protocol: TCP
```

### ALB vs NLB — Decision Matrix for EKS

| Requirement | Use ALB | Use NLB |
|---|---|---|
| HTTP/HTTPS routing (path, host, header) | ✅ | ❌ |
| gRPC microservices | ✅ (with ALPN h2) | ✅ (TCP passthrough) |
| WebSocket | ✅ | ✅ |
| Static IP / Elastic IP (for firewall whitelisting) | ❌ | ✅ |
| Ultra-low latency (microseconds) | ❌ | ✅ |
| Source IP preservation natively | ❌ (requires X-Forwarded-For) | ✅ |
| TLS passthrough (end-to-end encryption) | ❌ (terminates TLS) | ✅ |
| OIDC authentication offload | ✅ | ❌ |
| WAF integration | ✅ | ❌ |
| Multiple SSL certificates (SNI) | ✅ | ✅ |
| Millions of req/sec | ❌ (slower scale-out) | ✅ |
| Path/host-based routing rules | ✅ | ❌ |

> 📖 **AWS Docs:** [NLB on EKS — Service annotations](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.11/guide/service/annotations/)  
> 📖 **AWS Docs:** [NLB target type: ip](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/target-group-register-targets.html)

---

## 5. Security Groups for Pods (SGP)

In ECS, security group isolation happens at the **task** level via `awsvpc` network mode. In EKS, the equivalent is **Security Groups for Pods (SGP)** — a feature unique to EKS that assigns security groups directly to individual Kubernetes pods.

> 🎨 **Security Group Chain Diagram:**
>
> [![EKS Security Group Chain](https://design.canva.ai/giuf7ZD3Jo-iG6X)](https://www.canva.com/d/KzL4pn__lxLXiry)
>
> *Click to open and edit in Canva*

### How Security Groups for Pods Works

Without SGP, all pods on a node share the node's security group. With SGP enabled:
- Each pod gets its own branch ENI (Elastic Network Interface) attached to the worker node
- The security group is applied at the branch ENI level
- Traffic to/from the pod is controlled at the pod level — identical to ECS `awsvpc`
- Enabled by the **VPC CNI plugin** (`amazon-vpc-cni-k8s`) with `ENABLE_POD_ENI=true`

```
Without SGP:
  Worker Node SG ──► All Pods on node (shared security boundary)

With SGP:
  Worker Node SG ──► Node-level control
  Pod SG (pod-sg-1) ──► Pod A (branch ENI, pod IP)
  Pod SG (pod-sg-2) ──► Pod B (branch ENI, pod IP)
```

### Enabling Security Groups for Pods

```bash
# Enable pod ENI on the VPC CNI DaemonSet
kubectl set env daemonset aws-node \
  -n kube-system \
  ENABLE_POD_ENI=true

# Verify
kubectl get daemonset aws-node -n kube-system -o yaml | grep ENABLE_POD_ENI
```

### SecurityGroupPolicy CRD

Once SGP is enabled, assign security groups to pods via the `SecurityGroupPolicy` CRD (installed by the VPC CNI):

```yaml
apiVersion: vpcresources.k8s.aws/v1beta1
kind: SecurityGroupPolicy
metadata:
  name: api-pod-sgp
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-service        # Applies to pods with this label
  securityGroups:
    groupIds:
      - sg-pod-XXXXXXXX       # The security group assigned to matching pods
```

### Security Group Chain on EKS (with SGP)

```
Internet (0.0.0.0/0)
    │ TCP 443, TCP 80
    ▼
┌──────────────────────────┐
│  SG-ALB                  │
│  In:  443/80 from 0.0.0.0│
│  Out: {pod-port} → SG-POD│
└──────────┬───────────────┘
           │ TCP {pod-port} → Source: SG-ALB
           ▼
┌──────────────────────────┐
│  SG-NODE (Worker Node)   │  ← Still needed for node-level traffic
│  In:  {nodeport} from VPC│     (kubelet, kube-proxy, node-to-node)
│  Out: All               │
└──────────────────────────┘
           │
           ▼ (branch ENI — SGP)
┌──────────────────────────┐
│  SG-POD                  │
│  In:  {app-port} from    │
│       SG-ALB             │  ← Reference ALB SG, not CIDR
│  Out: 443 → 0.0.0.0/0   │  ← Bedrock, S3
│  Out: 5432 → SG-RDS      │  ← Database
│  Out: 6379 → SG-REDIS    │  ← ElastiCache
└──────────────────────────┘
```

> ⚠️ **Important:** SGP requires instance types that support branch ENIs. Not all instance types are supported. Check the [supported instance types list](https://docs.aws.amazon.com/eks/latest/userguide/security-groups-for-pods.html#supported-instance-types) before choosing worker node types.

> 📖 **AWS Docs:** [Security Groups for Pods](https://docs.aws.amazon.com/eks/latest/userguide/security-groups-for-pods.html)  
> 📖 **AWS Docs:** [VPC CNI plugin](https://docs.aws.amazon.com/eks/latest/userguide/managing-vpc-cni.html)  
> 📖 **AWS Docs:** [SecurityGroupPolicy CRD reference](https://aws.github.io/aws-eks-best-practices/networking/sgpp/)  
> 📖 **AWS Docs:** [Supported instance types for SGP](https://docs.aws.amazon.com/eks/latest/userguide/security-groups-for-pods.html#supported-instance-types)

---

## 6. Target Types: Instance vs IP Mode

This is the EKS equivalent of ECS `awsvpc` vs `bridge` mode — the most critical architectural decision for how ALB/NLB traffic reaches your pods.

| | Instance Mode | IP Mode |
|---|---|---|
| **How it works** | ALB → NodePort on EC2 node → kube-proxy → Pod | ALB → Pod IP directly (bypasses kube-proxy) |
| **Target registered** | EC2 instance ID + NodePort | Pod IP + containerPort |
| **Network hops** | 2 (ALB → Node → Pod) | 1 (ALB → Pod directly) |
| **Latency** | Higher (extra hop + iptables) | Lower |
| **Source IP** | Pod sees Node IP, not client IP | Pod sees actual client IP |
| **Fargate** | ❌ Not supported | ✅ Required |
| **kube-proxy dependency** | ✅ Required | ❌ Not needed |
| **Cross-AZ traffic risk** | Higher (pod may be in different AZ than node target) | Lower (direct pod targeting) |
| **Recommendation** | Legacy / EC2 only workloads | ✅ **Always use for new builds** |

```yaml
# IP mode — annotation on Ingress
alb.ingress.kubernetes.io/target-type: ip

# IP mode — annotation on NLB Service
service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
```

> 📖 **AWS Docs:** [LBC target type: IP vs instance](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.11/guide/ingress/annotations/#target-type)

---

## 7. EKS Service Types and Their Load Balancer Behavior

Understanding how Kubernetes Service types map to AWS load balancers is foundational.

| Service Type | AWS Behavior | When to Use |
|---|---|---|
| `ClusterIP` | No AWS LB created. Internal cluster-only DNS. | Pod-to-pod communication inside the cluster |
| `NodePort` | No AWS LB created. Port exposed on every node. | Intermediate step; manually pair with LB |
| `LoadBalancer` | LBC creates NLB (with proper annotation). Legacy in-tree creates CLB. | Direct NLB exposure for TCP/UDP workloads |
| `ExternalName` | DNS CNAME mapping. No LB. | Routing to external services via DNS |

**The Ingress approach (recommended for HTTP/HTTPS):**

Instead of `Service type: LoadBalancer` per service (which creates one NLB per service = expensive), use:
```
One Ingress → One ALB → Many Services (ClusterIP) → Many Pod sets
```

This is the canonical cost-efficient EKS pattern for HTTP workloads.

```
❌ Anti-pattern (one NLB per service — expensive):
  Service A (LoadBalancer) → NLB A
  Service B (LoadBalancer) → NLB B
  Service C (LoadBalancer) → NLB C

✅ Correct pattern (one ALB for all HTTP services):
  Ingress → ALB
    ├── /api/a → Service A (ClusterIP) → Pods A
    ├── /api/b → Service B (ClusterIP) → Pods B
    └── /api/c → Service C (ClusterIP) → Pods C
```

> 📖 **AWS Docs:** [Kubernetes Service types](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)  
> 📖 **AWS Docs:** [EKS networking best practices — services](https://aws.github.io/aws-eks-best-practices/networking/loadbalancing/)

---

## 8. Advanced Routing Patterns on EKS

### Pattern 1: IngressGroup — Shared ALB Across Multiple Ingress Resources

By default, each Ingress resource creates one ALB. For microservices with separate Ingress manifests per team, you can **share a single ALB** using `IngressGroup`:

```yaml
# team-users/ingress.yaml
metadata:
  annotations:
    alb.ingress.kubernetes.io/group.name: shared-alb    # ← All ingresses with same group share one ALB
    alb.ingress.kubernetes.io/group.order: "10"          # Priority within the group
```

```yaml
# team-orders/ingress.yaml
metadata:
  annotations:
    alb.ingress.kubernetes.io/group.name: shared-alb    # ← Same group = same ALB
    alb.ingress.kubernetes.io/group.order: "20"
```

Cost impact: **dramatic**. Instead of N ALBs (one per service team), you get 1 ALB handling all routing.

---

### Pattern 2: Canary / Weighted Routing with Ingress Annotations

```yaml
# Canary ingress (routes 10% of traffic)
metadata:
  annotations:
    alb.ingress.kubernetes.io/group.name: api-group
    alb.ingress.kubernetes.io/actions.forward-weighted: |
      {"type":"forward","forwardConfig":{"targetGroups":[
        {"serviceName":"api-v1","servicePort":"80","weight":90},
        {"serviceName":"api-v2","servicePort":"80","weight":10}
      ]}}
spec:
  rules:
    - http:
        paths:
          - path: /api/*
            pathType: ImplementationSpecific
            backend:
              service:
                name: forward-weighted   # References the action annotation
                port:
                  name: use-annotation
```

---

### Pattern 3: Header-Based Routing for Canary

```yaml
metadata:
  annotations:
    alb.ingress.kubernetes.io/conditions.canary-users: |
      [{"field":"http-header","httpHeaderConfig":{"httpHeaderName":"X-Canary-User","values":["true"]}}]
    alb.ingress.kubernetes.io/actions.canary-users: |
      {"type":"forward","targetGroupARN":"arn:aws:elasticloadbalancing:..."}
```

---

### Pattern 4: Internal ALB for East-West (Service-to-Service) Traffic

```yaml
metadata:
  annotations:
    alb.ingress.kubernetes.io/scheme: internal   # Only reachable within VPC
    alb.ingress.kubernetes.io/subnets: subnet-priv-1a,subnet-priv-1b
```

Use case: Service A calls Service B via the internal ALB's DNS name. Provides ALB-level routing, health checking, and SSL termination for intra-cluster traffic. Alternative: use **EKS Service Connect** or a service mesh (AWS App Mesh / Istio) for lower-latency east-west.

---

### Pattern 5: NLB in Front of ALB (Static IPs for ALB)

ALBs don't have static IPs. For enterprise environments requiring IP whitelisting:

```
Client (IP whitelist enforcement)
    │
    ▼ Static Elastic IPs
NLB (TCP passthrough, static IPs per AZ)
    │
    ▼ TCP:443 to ALB nodes
ALB (TLS termination, HTTP routing)
    │
    ▼ HTTP to EKS Pods
```

```yaml
# NLB Service pointing to ALB
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "alb"   # NLB → ALB
    service.beta.kubernetes.io/aws-load-balancer-eip-allocations: "eipalloc-XXXX,eipalloc-YYYY"
```

> 📖 **AWS Docs:** [LBC IngressGroup](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.11/guide/ingress/ingress_group/)  
> 📖 **AWS Docs:** [LBC — weighted target groups](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.11/guide/ingress/annotations/#actions)  
> 📖 **AWS Docs:** [Static IPs with NLB + ALB](https://aws.amazon.com/blogs/networking-and-content-delivery/using-static-ip-addresses-for-application-load-balancers/)

---

## 9. Certificates and TLS on EKS

### Option 1: ACM Certificate via Ingress Annotation (Recommended)

```yaml
alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:ACCOUNT:certificate/CERT-ID
```

The LBC attaches the ACM certificate to the ALB listener. TLS terminates at the ALB; traffic to pods is HTTP. Multiple certs (SNI) supported:

```yaml
alb.ingress.kubernetes.io/certificate-arn: |
  arn:aws:acm:...:certificate/CERT-A,
  arn:aws:acm:...:certificate/CERT-B
```

---

### Option 2: cert-manager with Let's Encrypt

For clusters that need automated certificate management with Let's Encrypt or internal CAs:

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml
```

```yaml
# ClusterIssuer for Let's Encrypt
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform@myapp.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: alb
```

```yaml
# Ingress using cert-manager
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - api.myapp.com
      secretName: api-tls-secret
```

> 📖 **AWS Docs:** [ACM with ALB on EKS](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-https-listener.html)  
> 📖 **AWS Docs:** [cert-manager on EKS](https://cert-manager.io/docs/installation/helm/)

---

## 10. IaC — Terraform Reference Patterns

```hcl
# ─────────────────────────────────────────────────────────────────
# EKS Cluster with VPC CNI and OIDC for IRSA
# ─────────────────────────────────────────────────────────────────
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "my-app-cluster"
  cluster_version = "1.31"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  cluster_endpoint_public_access = true

  # Enable IRSA (OIDC)
  enable_irsa = true

  eks_managed_node_groups = {
    app = {
      min_size       = 2
      max_size       = 10
      desired_size   = 3
      instance_types = ["m6i.large"]  # Must support branch ENIs for SGP
    }
  }

  tags = {
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

# ─────────────────────────────────────────────────────────────────
# IAM Role for AWS Load Balancer Controller (IRSA)
# ─────────────────────────────────────────────────────────────────
module "lb_controller_irsa" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  version = "~> 5.0"

  role_name = "aws-load-balancer-controller"

  attach_load_balancer_controller_policy = true

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:aws-load-balancer-controller"]
    }
  }
}

# ─────────────────────────────────────────────────────────────────
# AWS Load Balancer Controller via Helm
# ─────────────────────────────────────────────────────────────────
resource "helm_release" "aws_load_balancer_controller" {
  name       = "aws-load-balancer-controller"
  repository = "https://aws.github.io/eks-charts"
  chart      = "aws-load-balancer-controller"
  namespace  = "kube-system"
  version    = "1.11.0"

  set {
    name  = "clusterName"
    value = module.eks.cluster_name
  }

  set {
    name  = "serviceAccount.create"
    value = "false"
  }

  set {
    name  = "serviceAccount.name"
    value = "aws-load-balancer-controller"
  }

  set {
    name  = "region"
    value = var.aws_region
  }

  set {
    name  = "vpcId"
    value = module.vpc.vpc_id
  }

  depends_on = [module.lb_controller_irsa]
}

# ─────────────────────────────────────────────────────────────────
# ALB Security Group
# ─────────────────────────────────────────────────────────────────
resource "aws_security_group" "alb" {
  name   = "eks-alb-sg"
  vpc_id = module.vpc.vpc_id

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

  tags = { Name = "eks-alb-sg" }
}

# ─────────────────────────────────────────────────────────────────
# Pod Security Group (for SGP)
# ─────────────────────────────────────────────────────────────────
resource "aws_security_group" "pod" {
  name   = "eks-pod-sg"
  vpc_id = module.vpc.vpc_id

  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]  # Reference ALB SG by ID
  }

  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # Bedrock, S3, ECR
  }

  tags = { Name = "eks-pod-sg" }
}

# ─────────────────────────────────────────────────────────────────
# Subnet tagging — required for LBC auto-discovery
# ─────────────────────────────────────────────────────────────────
# Public subnets must have tag: kubernetes.io/role/elb = 1
# Private subnets must have tag: kubernetes.io/role/internal-elb = 1
# All cluster subnets must have tag: kubernetes.io/cluster/{cluster-name} = owned

resource "aws_ec2_tag" "public_subnet_elb" {
  for_each    = toset(module.vpc.public_subnets)
  resource_id = each.value
  key         = "kubernetes.io/role/elb"
  value       = "1"
}

resource "aws_ec2_tag" "private_subnet_internal_elb" {
  for_each    = toset(module.vpc.private_subnets)
  resource_id = each.value
  key         = "kubernetes.io/role/internal-elb"
  value       = "1"
}
```

> ⚠️ **Critical — Subnet Tagging:** The LBC discovers which subnets to place the ALB/NLB in via AWS resource tags. Without these tags, the controller cannot create load balancers and will emit errors. Always tag subnets.

| Tag | Value | Required On |
|---|---|---|
| `kubernetes.io/role/elb` | `1` | Public subnets (internet-facing ALB/NLB) |
| `kubernetes.io/role/internal-elb` | `1` | Private subnets (internal ALB/NLB) |
| `kubernetes.io/cluster/{cluster-name}` | `owned` or `shared` | All subnets used by the cluster |

> 📖 **Terraform Registry:** [terraform-aws-modules/eks](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest)  
> 📖 **Terraform Registry:** [iam-role-for-service-accounts-eks](https://registry.terraform.io/modules/terraform-aws-modules/iam/aws/latest/submodules/iam-role-for-service-accounts-eks)  
> 📖 **AWS Docs:** [Subnet tagging for LBC](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.11/deploy/subnet_discovery/)

---

## 11. Kubernetes YAML / Helm Reference

### Complete Deployment + Service + Ingress Stack

```yaml
# ─────────────────────────────────────────────
# Deployment
# ─────────────────────────────────────────────
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  namespace: production
  labels:
    app: api-service
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-service
  template:
    metadata:
      labels:
        app: api-service
        version: v1
    spec:
      serviceAccountName: api-service-sa  # IRSA service account for Bedrock/S3 access
      containers:
        - name: api
          image: ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/api-service:latest
          ports:
            - containerPort: 8080
          readinessProbe:           # ← ALB health checks pass through here
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 30
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 1Gi
          env:
            - name: AWS_REGION
              value: us-east-1

---
# ─────────────────────────────────────────────
# Service (ClusterIP — no LB per service)
# ─────────────────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: production
spec:
  type: ClusterIP       # No AWS LB created — ALB Ingress handles external traffic
  selector:
    app: api-service
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP

---
# ─────────────────────────────────────────────
# Ingress (creates ALB via LBC)
# ─────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:ACCOUNT:certificate/XXXX
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "10"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/target-group-attributes: deregistration_delay.timeout_seconds=30
    alb.ingress.kubernetes.io/load-balancer-attributes: |
      access_logs.s3.enabled=true,
      access_logs.s3.bucket=my-alb-logs,
      access_logs.s3.prefix=eks
    alb.ingress.kubernetes.io/group.name: production-shared-alb
    alb.ingress.kubernetes.io/tags: Environment=production,ManagedBy=kubernetes
spec:
  ingressClassName: alb
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /api/users
            pathType: Prefix
            backend:
              service:
                name: users-service
                port:
                  number: 80
          - path: /api/orders
            pathType: Prefix
            backend:
              service:
                name: orders-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80

---
# ─────────────────────────────────────────────
# SecurityGroupPolicy (SGP for pods)
# ─────────────────────────────────────────────
apiVersion: vpcresources.k8s.aws/v1beta1
kind: SecurityGroupPolicy
metadata:
  name: api-service-sgp
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-service
  securityGroups:
    groupIds:
      - sg-pod-XXXXXXXX

---
# ─────────────────────────────────────────────
# HorizontalPodAutoscaler
# ─────────────────────────────────────────────
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## 12. Observability on EKS

### ALB Metrics (same as ECS — via CloudWatch `AWS/ApplicationELB`)

| Metric | Alarm | Notes |
|---|---|---|
| `HTTPCode_Target_5XX_Count` | > SLA threshold | App errors from pods |
| `TargetResponseTime` p99 | > SLA threshold | End-to-end latency |
| `UnHealthyHostCount` | > 0 | Failed pod health checks |
| `HTTPCode_ELB_5XX_Count` | Any non-zero | ALB-level errors |

### EKS-Specific Observability

**Container Insights** — enables CloudWatch Metrics and Logs for EKS nodes and pods:

```bash
# Enable Container Insights via CloudWatch agent
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name amazon-cloudwatch-observability \
  --region us-east-1
```

**AWS Distro for OpenTelemetry (ADOT)** — the EKS-native distributed tracing + metrics collector. Replaces the need for individual language SDKs for X-Ray:

```bash
# Install ADOT operator
kubectl apply -f https://amazon-eks.s3.amazonaws.com/docs/addons-otel-permissions.yaml
aws eks create-addon --cluster-name my-cluster --addon-name adot
```

**Pod-level network observability with VPC Flow Logs** — enable on the VPC to capture traffic to/from pod branch ENIs when SGP is in use.

### Readiness vs Liveness — ALB Health Check Interaction

```
readinessProbe: FAILED → Pod removed from Service endpoints → ALB target deregistered → No traffic
livenessProbe:  FAILED → Pod restarted by kubelet → ECS behavior equivalent
```

> ⚠️ Always implement a proper `/health` endpoint that the `readinessProbe` calls. This is what the ALB health check ultimately validates via the Kubernetes readiness gate.

> 📖 **AWS Docs:** [CloudWatch Container Insights for EKS](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-EKS.html)  
> 📖 **AWS Docs:** [AWS Distro for OpenTelemetry on EKS](https://aws-otel.github.io/docs/getting-started/adot-eks-add-on)  
> 📖 **AWS Docs:** [ALB access logs](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/enable-access-logging.html)

---

## 13. AWS Reference Links

### AWS Load Balancer Controller
| Topic | Link |
|---|---|
| LBC Installation Guide | https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html |
| LBC GitHub | https://github.com/kubernetes-sigs/aws-load-balancer-controller |
| LBC Ingress Annotations | https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.11/guide/ingress/annotations/ |
| LBC Service Annotations (NLB) | https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.11/guide/service/annotations/ |
| LBC IngressGroup | https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.11/guide/ingress/ingress_group/ |
| LBC Subnet Discovery | https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.11/deploy/subnet_discovery/ |
| LBC Target Type: IP | https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.11/guide/ingress/annotations/#target-type |
| LBC Weighted Target Groups | https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.11/guide/ingress/annotations/#actions |
| LBC IAM Policy | https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json |

### EKS Networking
| Topic | Link |
|---|---|
| EKS Networking Overview | https://docs.aws.amazon.com/eks/latest/userguide/eks-networking.html |
| EKS Task Networking (awsvpc equivalent) | https://docs.aws.amazon.com/eks/latest/userguide/network-load-balancing.html |
| Security Groups for Pods | https://docs.aws.amazon.com/eks/latest/userguide/security-groups-for-pods.html |
| SGP Supported Instance Types | https://docs.aws.amazon.com/eks/latest/userguide/security-groups-for-pods.html#supported-instance-types |
| VPC CNI Plugin | https://docs.aws.amazon.com/eks/latest/userguide/managing-vpc-cni.html |
| EKS Service Connect | https://docs.aws.amazon.com/eks/latest/userguide/eks-networking.html |
| IAM Roles for Service Accounts (IRSA) | https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html |
| Pod Identity (new IRSA alternative) | https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html |
| EKS Best Practices — Networking | https://aws.github.io/aws-eks-best-practices/networking/loadbalancing/ |
| EKS Best Practices — Security | https://aws.github.io/aws-eks-best-practices/security/ |

### ALB / NLB Core Docs
| Topic | Link |
|---|---|
| ALB Overview | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html |
| ALB Listener Rules | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html |
| ALB Target Groups | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html |
| ALB Health Checks | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html |
| NLB Overview | https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html |
| NLB on EKS (blog) | https://aws.amazon.com/blogs/containers/expose-amazon-eks-pods-through-cross-account-load-balancer/ |
| gRPC on ALB | https://aws.amazon.com/blogs/aws/new-application-load-balancer-support-for-end-to-end-http-2-and-grpc/ |
| Static IPs for ALB (NLB+ALB) | https://aws.amazon.com/blogs/networking-and-content-delivery/using-static-ip-addresses-for-application-load-balancers/ |
| ACM with ALB | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-https-listener.html |
| WAFv2 with ALB | https://docs.aws.amazon.com/waf/latest/developerguide/waf-alb-integration.html |

### Observability
| Topic | Link |
|---|---|
| CloudWatch Container Insights for EKS | https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-EKS.html |
| ALB CloudWatch Metrics | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-cloudwatch-metrics.html |
| ALB Access Logs | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/enable-access-logging.html |
| AWS Distro for OpenTelemetry (ADOT) | https://aws-otel.github.io/docs/getting-started/adot-eks-add-on |
| X-Ray on EKS | https://docs.aws.amazon.com/xray/latest/devguide/xray-daemon-eks.html |

### Terraform
| Resource | Link |
|---|---|
| terraform-aws-modules/eks | https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest |
| iam-role-for-service-accounts-eks | https://registry.terraform.io/modules/terraform-aws-modules/iam/aws/latest/submodules/iam-role-for-service-accounts-eks |
| aws_eks_cluster | https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eks_cluster |
| helm_release (LBC) | https://registry.terraform.io/providers/hashicorp/helm/latest/docs/resources/release |
| aws_security_group | https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group |

### AWS Pricing
| Topic | Link |
|---|---|
| EKS Pricing | https://aws.amazon.com/eks/pricing/ |
| ALB Pricing | https://aws.amazon.com/elasticloadbalancing/pricing/ |
| NLB Pricing | https://aws.amazon.com/elasticloadbalancing/pricing/ |

---

## Full Architecture Summary

```
                    ┌─────────────────────────────────────────────────────┐
                    │                  Route 53 (DNS)                      │
                    └──────────────────────┬──────────────────────────────┘
                                           │
                    ┌──────────────────────▼──────────────────────────────┐
                    │  Internet-Facing ALB  (public subnets)               │
                    │  ┌──────────────────────────────────────────────┐   │
                    │  │ Listener HTTPS:443 (ACM cert, TLS 1.3)       │   │
                    │  │ Rules:                                        │   │
                    │  │  /api/users/*  → TG: users-svc (pod IPs)    │   │
                    │  │  /api/orders/* → TG: orders-svc (pod IPs)   │   │
                    │  │  default       → TG: frontend-svc (pod IPs) │   │
                    │  └──────────────────────────────────────────────┘   │
                    └──────────────────────┬──────────────────────────────┘
                                           │ SG-ALB → SG-POD (port 8080)
                    ┌──────────────────────▼──────────────────────────────┐
                    │           EKS Cluster  (private subnets)             │
                    │  ┌─────────────────────────────────────────────┐    │
                    │  │  AWS Load Balancer Controller (kube-system) │    │
                    │  │  Watches Ingress → Manages ALB via AWS API  │    │
                    │  └─────────────────────────────────────────────┘    │
                    │                                                      │
                    │  AZ-1                AZ-2               AZ-3        │
                    │  ┌──────────┐       ┌──────────┐      ┌──────────┐ │
                    │  │ Pod      │       │ Pod      │      │ Pod      │ │
                    │  │ (branch  │       │ (branch  │      │ (branch  │ │
                    │  │  ENI)    │       │  ENI)    │      │  ENI)    │ │
                    │  │ SG-POD ✅│       │ SG-POD ✅│      │ SG-POD ✅│ │
                    │  └────┬─────┘       └────┬─────┘      └────┬─────┘ │
                    └───────┼─────────────────┼───────────────────┼───────┘
                            │                 │                   │
               ┌────────────▼──┐   ┌──────────▼──┐   ┌──────────▼─────┐
               │Amazon Bedrock │   │ Amazon RDS   │   │   Amazon S3    │
               │(Gen AI / RAG) │   │ (Database)   │   │(KB / Objects)  │
               └───────────────┘   └─────────────┘   └────────────────┘
```

---

## ECS → EKS Quick Reference Migration Table

| ECS Concept | EKS Equivalent |
|---|---|
| ECS Task | Kubernetes Pod |
| ECS Service | Kubernetes Deployment + Service |
| Task Definition | Pod Spec / Deployment Spec |
| ECS Task Role (IAM) | IRSA Service Account / Pod Identity |
| Target Group (auto-registered by ECS) | LBC creates TG from Ingress / Service |
| ALB Listener Rule (in Terraform) | Kubernetes Ingress YAML annotation |
| `awsvpc` network mode | Security Groups for Pods (SGP) |
| `healthCheckGracePeriodSeconds` | `initialDelaySeconds` on readinessProbe |
| ECS Deployment (rolling) | Kubernetes RollingUpdate strategy |
| ECS Blue/Green (CodeDeploy) | Argo Rollouts / Flagger / weighted Ingress |
| ECS Service Connect | EKS Service Connect / App Mesh / Istio |
| CloudWatch Container Insights | CloudWatch Container Insights (addon) |

---

*Generated for AWS DevOps Engineers — Principal Level*  
*Related guide: [aws-alb-ecs-deep-dive.md](./aws-alb-ecs-deep-dive.md)*  
*For contributions or corrections, open a PR.*
