# You're One Bad Deploy Away From 3AM Pages — Here's Every Deployment Strategy, When to Use It, and How to Pitch It 🚀

> **For Principal DevOps Engineers, Staff Engineers, and the person their manager just asked:**  
> _"What deployment strategy should we use?"_  
>
> Every strategy. Deep internals. Real CI/CD pipeline proposals for startups AND Fortune 500s.  
> The document your architecture review board actually wants to see.

---

## 📋 Table of Contents

1. [Why Deployment Strategy Is a Business Decision, Not a Technical One](#1-why-deployment-strategy-is-a-business-decision)
2. [The Risk/Complexity Matrix — How to Choose](#2-the-riskcomplexity-matrix)
3. [Recreate — The Nuclear Option](#3-recreate)
4. [Rolling Deployment — The Sensible Default](#4-rolling-deployment)
5. [Blue/Green Deployment — The Instant Rollback](#5-bluegreen-deployment)
6. [Canary Deployment — The Scientific Method](#6-canary-deployment)
7. [A/B Testing Deployment — The Product Manager's Weapon](#7-ab-testing-deployment)
8. [Shadow Deployment — The Risk-Free Validator](#8-shadow-deployment)
9. [Feature Flags — Deployment Without Deploying](#9-feature-flags)
10. [Multi-Region / Progressive Delivery](#10-multi-region--progressive-delivery)
11. [The CI/CD Pipeline Proposal — Startup (Series A/B)](#11-cicd-pipeline-proposal--startup)
12. [The CI/CD Pipeline Proposal — Top 10 Company (Enterprise)](#12-cicd-pipeline-proposal--enterprise)
13. [Database Migrations — The Part Everyone Forgets](#13-database-migrations)
14. [Rollback Strategies — The Thing You Need Before You Deploy](#14-rollback-strategies)
15. [Metrics That Prove Your Strategy Is Working](#15-metrics-that-prove-your-strategy-is-working)
16. [The Architecture Review Pitch — Templates](#16-the-architecture-review-pitch)
17. [Quick Reference Decision Card](#17-quick-reference-decision-card)

---

## 1. Why Deployment Strategy Is a Business Decision

Most engineers think about deployment strategy as a technical problem. Principal engineers know it's a **business risk problem**.

Before recommending any strategy, a Principal DevOps engineer answers five questions:

```
┌─────────────────────────────────────────────────────────────────────┐
│            THE 5 QUESTIONS BEFORE CHOOSING A STRATEGY               │
│                                                                      │
│  1. BLAST RADIUS                                                     │
│     If this deploy goes wrong, how many users are affected?          │
│     100? 100,000? 100 million?                                       │
│                                                                      │
│  2. RECOVERY TIME                                                    │
│     If this breaks, how fast can we recover?                         │
│     Rollback in 30 seconds? Or re-deploy for 20 minutes?            │
│                                                                      │
│  3. COST OF DOWNTIME                                                 │
│     Is $1,000/minute of downtime acceptable? $10,000? $1M?          │
│     A startup at 2AM: probably fine. NYSE trading system: never.    │
│                                                                      │
│  4. TEAM MATURITY                                                    │
│     Can your team operate canary traffic splitting at 2AM?           │
│     Do you have the observability to know if canary is failing?      │
│                                                                      │
│  5. INFRASTRUCTURE COST                                              │
│     Blue/green doubles your infrastructure during deploy.            │
│     Can your budget handle that? Even temporarily?                   │
└─────────────────────────────────────────────────────────────────────┘
```

### The Deployment Spectrum

```
LOW RISK ◄─────────────────────────────────────────────► HIGH RISK
LOW COST                                                  HIGH COST

  Recreate    Rolling    Blue/Green    Canary    A/B    Shadow
     │            │           │           │        │        │
  Fast &      Simple &    Safe &      Careful   User    Zero
  Brutal      Default     Instant     Science   Split   Risk
              Choice     Rollback    Deploy           (readonly)

COMPLEXITY:
  Simple ─────────────────────────────────────────── Complex

TEAM REQUIRED:
  1 person ──────────────────────────────── Full Platform Team
```

---

## 2. The Risk/Complexity Matrix — How to Choose

```
                    LOW COMPLEXITY          HIGH COMPLEXITY
                ┌───────────────────┬──────────────────────────┐
                │                   │                           │
   LOW RISK     │   Rolling         │   Canary                  │
                │   (sweet spot     │   Shadow                  │
                │   for most teams) │   Progressive Delivery    │
                │                   │                           │
                ├───────────────────┼──────────────────────────┤
                │                   │                           │
   HIGH RISK    │   Recreate        │   A/B Testing             │
                │   (only for       │   (requires product,      │
                │   dev/staging)    │   stats, feature flags)   │
                │                   │                           │
                └───────────────────┴──────────────────────────┘

Blue/Green sits between Low-Risk/Low-Complexity and Low-Risk/High-Complexity
depending on whether you automate the switchover.

STARTUP RECOMMENDATION:        Rolling → Blue/Green → Canary (in that order, as you scale)
ENTERPRISE RECOMMENDATION:     Canary + Feature Flags + Progressive Delivery
```

---

## 3. Recreate — The Nuclear Option

### What It Is

Kill everything. Start fresh. Simplest possible strategy.

```
BEFORE DEPLOY               DEPLOY                    AFTER DEPLOY
─────────────────           ──────────────────        ─────────────────

Pod v1 [████████]           ALL pods v1 die           Pod v2 [████████]
Pod v1 [████████]    →      ↓↓↓↓↓↓↓↓↓↓↓↓↓     →     Pod v2 [████████]
Pod v1 [████████]           Downtime window            Pod v2 [████████]
                            (could be 30s–5min)
```

### How It Works

```yaml
# Kubernetes Deployment with Recreate strategy
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: Recreate         # ← ALL old pods terminated before new ones start
  template:
    spec:
      containers:
        - name: myapp
          image: myapp:v2.0.0
```

```bash
# GitHub Actions step:
- name: Deploy (Recreate)
  run: |
    kubectl set image deployment/myapp myapp=myapp:${{ github.sha }}
    kubectl rollout status deployment/myapp
    # ↑ Waits for rollout to complete. Exits non-zero if fails.
```

### Traffic During Recreate

```
Time →
────────────────────────────────────────────────────────
v1 traffic:  ████████████████████░░░░░░░░░░████████████
v2 traffic:  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░████████████
                                   ↑
                              DOWNTIME GAP
                         (users see 503 errors)
```

### When to Use

```
✅ USE RECREATE WHEN:
   → Development or staging environments
   → Scheduled maintenance windows (comms sent, users expect downtime)
   → Breaking schema changes that CANNOT run alongside old version
   → Internal tools, batch jobs, non-user-facing services
   → Your team is small and simplicity > everything else

❌ NEVER USE RECREATE WHEN:
   → Production user-facing services
   → Financial transactions are in-flight
   → SLA requires 99.9%+ uptime (that's ~8h downtime/year — Recreate uses it in one deploy)
   → You have any paying customers
```

### Rollback

```bash
kubectl rollout undo deployment/myapp
# Kills all v2 pods, restarts all v1 pods
# Same downtime window as the original deploy
```

---

## 4. Rolling Deployment — The Sensible Default

### What It Is

Replace pods one at a time (or in small batches). Zero downtime. Simple to operate.

```
STEP 1          STEP 2          STEP 3          STEP 4 (DONE)
─────────       ─────────       ─────────       ─────────────
v1 [████]       v2 [████]       v2 [████]       v2 [████]
v1 [████]  →    v1 [████]  →    v2 [████]  →    v2 [████]
v1 [████]       v1 [████]       v1 [████]       v2 [████]

During EACH step: old and new version BOTH serve traffic
```

### How It Works — Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2          # Allow 2 extra pods during rollout (6+2=8 max)
      maxUnavailable: 0    # Never kill old pod until new one is healthy
      # ↑ This combination = zero-downtime rolling deploy
      #
      # maxSurge: 0, maxUnavailable: 2 = no extra capacity (cost-saving)
      # maxSurge: 2, maxUnavailable: 0 = zero-downtime (recommended prod)
      # maxSurge: 50%, maxUnavailable: 25% = balanced (scales with replicas)

  minReadySeconds: 30      # Wait 30s after pod is Ready before proceeding
                           # Catches slow startup failures

  template:
    spec:
      containers:
        - name: myapp
          image: myapp:v2.0.0

          # CRITICAL: readinessProbe prevents routing to unhealthy pods
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3    # 3 failures → pod NOT ready → rollout pauses
```

### Traffic During Rolling Deploy

```
Time →
─────────────────────────────────────────────────────────────────
v1 traffic:  ████████████████▓▓▓▓▓▓░░░░░░░░░░░░░░░░
v2 traffic:  ░░░░░░░░░░░░░░░░▒▒▒▒▒▒████████████████

           ↑                ↑
    100% v1 serving    Both versions serving simultaneously
                       (v1 + v2 requests mixed in flight!)
```

### The Biggest Rolling Deployment Risk

```
BOTH VERSIONS RUN SIMULTANEOUSLY.

This means your code must be:
  ✅ Backward compatible with old API contracts
  ✅ Database schema compatible with old version (no breaking migrations)
  ✅ Feature flags if new features are incompatible

EXAMPLE OF A ROLLING DEPLOY THAT BREAKS:
  v1 writes:  { "user_id": 123, "name": "Alice" }
  v2 writes:  { "userId": 123,  "name": "Alice" }  ← renamed field!

  During rollout: some pods run v1, some run v2
  v1 reads v2's "userId" field → field not found → crash
  v2 reads v1's "user_id" field → field not found → crash

  FIX: Deploy in 3 phases:
    Phase 1: v2 reads BOTH old and new field names
    Phase 2: Rolling deploy of v2
    Phase 3: Remove old field support in v3
```

### CI/CD Pipeline — Rolling (GitHub Actions + EKS)

```yaml
# .github/workflows/deploy-rolling.yml
name: Rolling Deploy to EKS

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-deploy-role
          aws-region: us-east-1

      - name: Login to ECR
        id: ecr-login
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push image
        env:
          ECR_REGISTRY: ${{ steps.ecr-login.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/myapp:$IMAGE_TAG .
          docker push $ECR_REGISTRY/myapp:$IMAGE_TAG

      - name: Run tests against new image
        run: |
          docker run --rm $ECR_REGISTRY/myapp:$IMAGE_TAG python -m pytest tests/

      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name my-cluster --region us-east-1

      - name: Rolling deploy
        env:
          IMAGE_TAG: ${{ github.sha }}
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ steps.ecr-login.outputs.registry }}/myapp:$IMAGE_TAG \
            -n production

      - name: Wait for rollout
        run: |
          kubectl rollout status deployment/myapp -n production --timeout=5m
          # Exit 1 if rollout fails → triggers GitHub Actions failure notification

      - name: Smoke test
        run: |
          ENDPOINT=$(kubectl get svc myapp-svc -n production \
            -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
          curl -f "http://$ENDPOINT/health" || exit 1

      - name: Rollback on failure
        if: failure()
        run: kubectl rollout undo deployment/myapp -n production
```

### When to Use

```
✅ USE ROLLING WHEN:
   → Your DEFAULT for most production services
   → Team is small (1-5 engineers)
   → Services are stateless (APIs, web servers)
   → API is backward compatible between versions
   → You want zero downtime without doubling infrastructure cost

⚠️ WATCH OUT FOR:
   → Stateful services (databases, session stores) — need extra care
   → Long-running requests (a 5-minute job can be killed mid-flight)
   → Breaking API changes (both versions run simultaneously)
```

---

## 5. Blue/Green Deployment — The Instant Rollback

### What It Is

Two identical production environments. Traffic flips from one to the other atomically. Old environment stays up as your instant rollback.

```
BEFORE DEPLOY:
─────────────────────────────────────────────────────────────────
                            ┌─────────────────┐
   INTERNET ──────────────▶ │   BLUE (v1)     │ ← 100% traffic
                            │   3 pods live   │
                            └─────────────────┘
                            ┌─────────────────┐
                            │   GREEN (v2)    │ ← idle / non-existent
                            └─────────────────┘

DEPLOY NEW VERSION:
─────────────────────────────────────────────────────────────────
   Spin up GREEN (v2) pods while BLUE (v1) still serves 100% traffic
   Run smoke tests against GREEN directly
   GREEN is healthy → flip the switch

AFTER DEPLOY:
─────────────────────────────────────────────────────────────────
                            ┌─────────────────┐
                            │   BLUE (v1)     │ ← idle (KEEP IT!)
                            └─────────────────┘
                            ┌─────────────────┐
   INTERNET ──────────────▶ │   GREEN (v2)    │ ← 100% traffic
                            │   3 pods live   │
                            └─────────────────┘

ROLLBACK = flip switch back to BLUE. Done in seconds.
```

### Implementation on Kubernetes

```yaml
# blue-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
  labels:
    app: myapp
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      slot: blue
  template:
    metadata:
      labels:
        app: myapp
        slot: blue
        version: v1.2.0
    spec:
      containers:
        - name: myapp
          image: myapp:v1.2.0

---
# green-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
  labels:
    app: myapp
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      slot: green
  template:
    metadata:
      labels:
        app: myapp
        slot: green
        version: v1.3.0
    spec:
      containers:
        - name: myapp
          image: myapp:v1.3.0

---
# service.yaml — THE TRAFFIC SWITCH
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  selector:
    app: myapp
    slot: blue        # ← CHANGE THIS to "green" to flip traffic
  ports:
    - port: 80
      targetPort: 8080
```

```bash
# The flip switch command (atomic — takes effect in seconds):
kubectl patch svc myapp-svc \
  -p '{"spec":{"selector":{"app":"myapp","slot":"green"}}}'

# Verify:
kubectl get svc myapp-svc -o jsonpath='{.spec.selector}'
# {"app":"myapp","slot":"green"}

# Rollback (just flip back):
kubectl patch svc myapp-svc \
  -p '{"spec":{"selector":{"app":"myapp","slot":"blue"}}}'
```

### Traffic During Blue/Green

```
Time →
─────────────────────────────────────────────────────────────────────
Blue traffic:  ██████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░
Green traffic: ░░░░░░░░░░░░░░░░░░░░░░░░░░██████████████████████████

                                          ↑
                                   Instant switch
                                   No mixed traffic
                                   No in-between state
```

### Blue/Green with AWS — ALB Weighted Target Groups

```bash
# On EKS with AWS ALB Controller, you can use weighted target groups
# for a more AWS-native blue/green:

# Blue service (current):
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: myapp-blue
  annotations:
    # ALB target group binding
spec:
  selector:
    slot: blue
  ports: [{port: 80, targetPort: 8080}]
EOF

# The Ingress uses ALB action annotations for weighted routing:
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    alb.ingress.kubernetes.io/actions.myapp-switch: |
      {
        "type":"forward",
        "forwardConfig":{
          "targetGroups":[
            {"serviceName":"myapp-blue","servicePort":"80","weight":100},
            {"serviceName":"myapp-green","servicePort":"80","weight":0}
          ]
        }
      }
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-switch   # References the action annotation
                port:
                  name: use-annotation
EOF

# Flip to green (patch annotation weight):
# blue: 0, green: 100
```

### CI/CD Pipeline — Blue/Green (GitHub Actions)

```yaml
# .github/workflows/deploy-bluegreen.yml
name: Blue/Green Deploy

jobs:
  deploy:
    steps:
      - name: Determine current slot
        id: current
        run: |
          CURRENT=$(kubectl get svc myapp-svc \
            -o jsonpath='{.spec.selector.slot}')
          echo "current=$CURRENT" >> $GITHUB_OUTPUT
          echo "next=$([ "$CURRENT" = "blue" ] && echo "green" || echo "blue")" \
            >> $GITHUB_OUTPUT

      - name: Deploy to inactive slot
        run: |
          # Deploy new version to the NON-live slot
          kubectl set image deployment/myapp-${{ steps.current.outputs.next }} \
            myapp=$ECR_REGISTRY/myapp:${{ github.sha }}
          kubectl rollout status deployment/myapp-${{ steps.current.outputs.next }} \
            --timeout=5m

      - name: Smoke test inactive slot (before switching!)
        run: |
          # Test the new slot directly via its own service
          kubectl port-forward svc/myapp-${{ steps.current.outputs.next }}-internal \
            8080:80 &
          sleep 5
          curl -f http://localhost:8080/health
          curl -f http://localhost:8080/api/version
          kill %1

      - name: Switch traffic to new slot
        run: |
          kubectl patch svc myapp-svc \
            -p "{\"spec\":{\"selector\":{\"slot\":\"${{ steps.current.outputs.next }}\"}}}"

      - name: Post-switch validation (30 second soak)
        run: |
          sleep 30
          # Check error rate via metrics
          ERROR_RATE=$(kubectl exec -n monitoring prometheus-0 -- \
            curl -s 'http://localhost:9090/api/v1/query?query=rate(http_requests_total{status=~"5.."}[1m])' \
            | jq '.data.result[0].value[1]')
          if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
            echo "Error rate too high: $ERROR_RATE"
            exit 1
          fi

      - name: Rollback if post-switch fails
        if: failure()
        run: |
          kubectl patch svc myapp-svc \
            -p "{\"spec\":{\"selector\":{\"slot\":\"${{ steps.current.outputs.current }}\"}}}'"
          echo "Rolled back to ${{ steps.current.outputs.current }}"
```

### When to Use

```
✅ USE BLUE/GREEN WHEN:
   → Instant rollback is a hard requirement (financial systems, healthcare)
   → You can afford double infrastructure cost temporarily
   → Breaking changes where both versions cannot run simultaneously
   → You need to validate the full environment before switching
   → Compliance requires human approval before traffic shift
   → You have scheduled maintenance windows

❌ DON'T USE WHEN:
   → Budget is tight (doubles instance costs during deploy)
   → Database schema is shared — both deployments hit same DB
     (solve with DB migration strategy first)
   → You deploy 20+ times/day (double infra cost adds up fast)

COST EXAMPLE (EKS, 10x m5.large nodes):
   Normal:    10 nodes × $0.096/hr = $0.96/hr
   B/G deploy: 20 nodes × $0.096/hr = $1.92/hr
   30-min deploy: ~$0.48 extra per deploy
   100 deploys/month = $48/month extra
```

---

## 6. Canary Deployment — The Scientific Method

### What It Is

Send a small percentage of REAL production traffic to the new version. Measure. Expand if healthy. Roll back if not.

```
PHASE 1: 5% Canary
─────────────────────────────────────────────────────────────────
100 users arrive
  ├── 95 → v1 (stable)   ████████████████████████████████████████
  └──  5 → v2 (canary)   ██

Observe for 10-30 minutes. Error rate, latency, business metrics OK?

PHASE 2: 25% Canary
─────────────────────────────────────────────────────────────────
  ├── 75 → v1   ██████████████████████████████
  └── 25 → v2   ██████████

Still healthy after 30 minutes?

PHASE 3: 50% → 100%
─────────────────────────────────────────────────────────────────
  └── 100 → v2   ████████████████████████████████████████████████

ROLLBACK = reduce canary weight to 0% instantly.
```

### Implementation — Kubernetes with Nginx Ingress

```yaml
# Method 1: Pod replica ratio (simple — no special controller needed)
# 10% canary = 1 canary pod vs 9 stable pods

# stable-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 9          # 90% of traffic
  selector:
    matchLabels:
      app: myapp
      track: stable
  template:
    metadata:
      labels:
        app: myapp
        track: stable
    spec:
      containers:
        - name: myapp
          image: myapp:v1.2.0

---
# canary-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1          # 10% of traffic
  selector:
    matchLabels:
      app: myapp
      track: canary
  template:
    metadata:
      labels:
        app: myapp
        track: canary
    spec:
      containers:
        - name: myapp
          image: myapp:v1.3.0

---
# service.yaml — selects BOTH stable and canary pods
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  selector:
    app: myapp     # ← No "track" selector = selects ALL pods
                   # Traffic distributed proportional to replica count
  ports:
    - port: 80
      targetPort: 8080
```

```yaml
# Method 2: Nginx Ingress annotations (precise weight control)
# stable-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-stable
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-stable-svc
                port:
                  number: 80

---
# canary-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-canary
  annotations:
    # ← These 3 annotations make this a canary ingress:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"   # 10% of traffic
    # nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"   # OR: header-based
    # nginx.ingress.kubernetes.io/canary-by-cookie: "canary"      # OR: cookie-based
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-canary-svc
                port:
                  number: 80
```

```bash
# Progressively increase canary weight:
kubectl annotate ingress myapp-canary \
  nginx.ingress.kubernetes.io/canary-weight="25" --overwrite

kubectl annotate ingress myapp-canary \
  nginx.ingress.kubernetes.io/canary-weight="50" --overwrite

kubectl annotate ingress myapp-canary \
  nginx.ingress.kubernetes.io/canary-weight="100" --overwrite

# Rollback (set weight to 0 — instantly stops canary traffic):
kubectl annotate ingress myapp-canary \
  nginx.ingress.kubernetes.io/canary-weight="0" --overwrite
```

### Automated Canary with Argo Rollouts

```yaml
# The production-grade way — Argo Rollouts
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 5          # 5% traffic
        - pause: {duration: 10m}# Wait 10 minutes — humans can review
        - setWeight: 25
        - pause: {duration: 10m}
        - setWeight: 50
        - pause: {}             # Indefinite pause — needs manual promotion
        - setWeight: 100

      # Automatic rollback based on Prometheus metrics:
      analysis:
        templates:
          - templateName: success-rate
        startingStep: 1
        args:
          - name: service-name
            value: myapp-svc

---
# AnalysisTemplate — defines "what does healthy mean?"
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 2m
      successCondition: result[0] >= 0.99     # 99% success rate required
      failureLimit: 3                          # 3 failures = auto-rollback
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}",
              status!~"5.."}[2m])) /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[2m]))
```

```bash
# Argo Rollouts commands:
kubectl argo rollouts get rollout myapp --watch   # Live status
kubectl argo rollouts promote myapp              # Manual promotion to next step
kubectl argo rollouts abort myapp                # Rollback immediately
kubectl argo rollouts dashboard                  # Web UI
```

### When to Use

```
✅ USE CANARY WHEN:
   → High-traffic production services (risk-adjusted by blast radius)
   → Performance-sensitive changes (latency regression caught at 5%)
   → New features with uncertain reliability
   → You have good observability (Prometheus, Grafana, error tracking)
   → You can define "healthy" in measurable metrics
   → Platform team manages the rollout (automation essential)

❌ DON'T USE CANARY WHEN:
   → Your team can't observe the difference between v1 and v2 traffic
   → You have < 1,000 requests/hour (too few for statistical significance)
   → Breaking changes (database schema) — both run simultaneously!
   → You can't tolerate ANY users hitting the new version
```

---

## 7. A/B Testing Deployment — The Product Manager's Weapon

### What It Is

Route specific USER SEGMENTS to different versions. Not random — intentional. Measure BUSINESS outcomes (conversion, revenue, engagement), not just error rates.

```
CANARY:           Random 5% of users  → new version
                  Goal: Detect bugs/regressions

A/B TESTING:      Users in segment A  → version A (control)
                  Users in segment B  → version B (variant)
                  Goal: Measure business impact of a product change

SEGMENTS can be based on:
  → User ID hash (consistent — same user always gets same variant)
  → Geography (US users see A, EU users see B)
  → Device type (mobile: A, desktop: B)
  → Account tier (free: A, paid: B)
  → Custom header or cookie
  → Feature flag assignment
```

### Implementation — Header-Based Routing (Nginx)

```yaml
# A/B Ingress with header-based routing
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-variant-b
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    # Route users with specific header to variant B:
    nginx.ingress.kubernetes.io/canary-by-header: "X-AB-Test"
    nginx.ingress.kubernetes.io/canary-by-header-value: "variant-b"
    # OR route by cookie (persistent across sessions):
    # nginx.ingress.kubernetes.io/canary-by-cookie: "ab_variant"
    # Cookie value "variant-b" → routes to this ingress
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-b-svc
                port:
                  number: 80
```

```python
# Your application assigns users to variants on first visit:
import hashlib

def get_user_variant(user_id: str) -> str:
    """Consistent hash-based A/B assignment.
    Same user always gets same variant."""
    # Hash user_id → 0–99
    hash_value = int(hashlib.md5(user_id.encode()).hexdigest(), 16) % 100
    if hash_value < 50:
        return "control"    # 50% get variant A
    else:
        return "variant-b"  # 50% get variant B

# Set cookie in response:
# Set-Cookie: ab_variant=variant-b; Max-Age=2592000; Path=/
```

### A/B Testing vs Canary — The Critical Distinction

```
┌─────────────────────┬───────────────────────┬──────────────────────────┐
│ Dimension           │ Canary                │ A/B Testing              │
├─────────────────────┼───────────────────────┼──────────────────────────┤
│ User assignment     │ Random                │ Deterministic (same user │
│                     │                       │ always same variant)      │
│ Duration            │ Hours (until stable)  │ Days/weeks (statistical  │
│                     │                       │ significance)             │
│ Success metric      │ Error rate, latency   │ Conversion, revenue,     │
│                     │                       │ engagement, retention    │
│ Owns it             │ DevOps/Platform       │ Product + Data Science   │
│ Rollback trigger    │ Error spike           │ Business metric drops    │
│ Infrastructure      │ Temporary             │ Long-running parallel    │
└─────────────────────┴───────────────────────┴──────────────────────────┘
```

---

## 8. Shadow Deployment — The Risk-Free Validator

### What It Is

Send a copy of ALL real production traffic to the new version. The new version processes everything but its responses are **discarded** — users only see responses from the stable version.

```
PRODUCTION REQUEST:
  User sends: POST /api/process-payment
                │
                ├──────────────────────▶ v1 (STABLE) → response sent to user ✅
                │
                └──────── (mirrored) ──▶ v2 (SHADOW) → response DISCARDED 🗑️
                                          ^ v2 does REAL work but result ignored
```

### Implementation — Nginx Ingress Mirroring

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-mirror
  annotations:
    # Mirror all traffic to shadow service (Nginx feature):
    nginx.ingress.kubernetes.io/mirror-uri: "http://myapp-shadow-svc.production.svc.cluster.local"
    nginx.ingress.kubernetes.io/mirror-request-body: "on"
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-stable-svc
                port:
                  number: 80
```

### Implementation — Istio Traffic Mirroring

```yaml
# Istio VirtualService with mirroring
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp-mirror
spec:
  hosts:
    - myapp-svc
  http:
    - route:
        - destination:
            host: myapp-svc
            subset: stable
          weight: 100
      mirror:
        host: myapp-svc
        subset: shadow          # Mirror to shadow subset
      mirrorPercentage:
        value: 100.0            # Mirror 100% of traffic
```

### When to Use

```
✅ PERFECT FOR:
   → Payment processing systems (test new payment library with real transactions)
   → ML model validation (compare new model outputs vs old in production conditions)
   → Major infrastructure rewrites (rewrite in Go? Shadow test it first)
   → Compliance-heavy systems (can't risk user impact)
   → Performance regression testing (measure new version latency under real load)

⚠️ WATCH OUT FOR:
   → Side effects: shadow service still writes to DBs, sends emails, charges cards!
   → Shadow env needs to use test/stub external services
   → Double infrastructure cost (both versions run at full capacity)
   → Request volume doubles on backends (rate limit aware)
```

---

## 9. Feature Flags — Deployment Without Deploying

### What It Is

Separate **code deployment** from **feature release**. Deploy code to 100% of users, but features are gated behind runtime toggles controlled without re-deploying.

```
TRADITIONAL:
  Code change → Deploy → Feature is live for everyone
  Rollback = re-deploy old code (minutes)

WITH FEATURE FLAGS:
  Code change → Deploy (flag OFF, feature hidden)
  Ready to release → Toggle flag ON (milliseconds)
  Problem? → Toggle flag OFF (milliseconds, no deploy)

The deploy and the release are DECOUPLED.
```

### Feature Flag Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    Feature Flag Service                           │
│   (LaunchDarkly / Flagsmith / Unleash / Flipt / AWS AppConfig)   │
│                                                                   │
│   Flag: "new-checkout-flow"                                       │
│   Rules:                                                          │
│     → Internal users (email ends in @mycompany.com) → ON         │
│     → Beta users (segment: early-adopter) → ON                   │
│     → US region → 10% rollout → ON                               │
│     → Everyone else → OFF                                        │
└───────────────────────────┬──────────────────────────────────────┘
                            │ SDK call (< 1ms, cached)
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Your Application                               │
│                                                                   │
│   if feature_flags.is_enabled("new-checkout-flow", user):        │
│       return new_checkout_flow()   # new code path               │
│   else:                                                           │
│       return old_checkout_flow()   # old code path               │
└──────────────────────────────────────────────────────────────────┘
```

### Implementation — Self-Hosted with Flagsmith on EKS

```yaml
# flagsmith-deployment.yaml (self-hosted feature flags)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flagsmith
  namespace: platform
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: flagsmith
          image: flagsmith/flagsmith:latest
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: flagsmith-secrets
                  key: database-url
          ports:
            - containerPort: 8000
```

```python
# Python application using feature flags:
from flagsmith import Flagsmith

flagsmith = Flagsmith(environment_key=os.getenv("FLAGSMITH_ENV_KEY"))

@app.get("/checkout")
async def checkout(user: User = Depends(get_current_user)):
    flags = flagsmith.get_identity_flags(
        identifier=str(user.id),
        traits={"plan": user.plan, "country": user.country}
    )

    if flags.is_feature_enabled("new-checkout-flow"):
        return await new_checkout_handler(user)
    else:
        return await legacy_checkout_handler(user)
```

```bash
# AWS AppConfig (managed feature flags on EKS):
aws appconfig create-application --name myapp
aws appconfig create-environment --application-id $APP_ID --name production
aws appconfig create-configuration-profile \
  --application-id $APP_ID \
  --name feature-flags \
  --location-uri hosted
# Update flags without any deployment:
aws appconfig start-deployment \
  --application-id $APP_ID \
  --environment-id $ENV_ID \
  --configuration-profile-id $PROFILE_ID \
  --deployment-strategy-id $STRATEGY_ID \
  --configuration-version 2
```

---

## 10. Multi-Region / Progressive Delivery

### What It Is

Roll out changes geographically, one region at a time. Region = blast radius boundary.

```
DAY 1:  Deploy to ap-southeast-1 (Singapore) — 5% of global traffic
        Monitor 24 hours. All green? Continue.

DAY 2:  Deploy to eu-west-1 (Ireland) — 25% of global traffic
        Monitor 24 hours.

DAY 3:  Deploy to us-east-1 (N. Virginia) — 45% of global traffic
        Monitor.

DAY 4:  Deploy to us-west-2 (Oregon) — 100% of global traffic
        Done.

If Singapore blows up on Day 1:
→ Roll back Singapore
→ EU, US never see the change
→ 95% of users never affected
```

---

## 11. CI/CD Pipeline Proposal — Startup (Series A/B)

### Context

- Team: 5-15 engineers
- Services: 5-20 microservices
- Scale: Up to 100K users
- Budget: Cost-sensitive
- Velocity: Ship multiple times per day

### The Proposed Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    STARTUP CI/CD PIPELINE                                │
│                                                                          │
│  Developer                                                               │
│  laptop ──▶ git push ──▶ GitHub ──────────────────────────────────────┐ │
│                             │                                           │ │
│                    ┌────────▼────────┐                                  │ │
│                    │  GitHub Actions │                                  │ │
│                    │  (CI Pipeline)  │                                  │ │
│                    └────────┬────────┘                                  │ │
│                             │                                           │ │
│              ┌──────────────▼──────────────┐                           │ │
│              │       CI STAGES             │                           │ │
│              │  1. Lint & format check     │                           │ │
│              │  2. Unit tests              │                           │ │
│              │  3. Build Docker image      │                           │ │
│              │  4. Security scan (Trivy)   │                           │ │
│              │  5. Push to ECR             │                           │ │
│              │  6. Integration tests       │                           │ │
│              └──────────────┬──────────────┘                           │ │
│                             │  on merge to main                        │ │
│              ┌──────────────▼──────────────┐                           │ │
│              │    STAGING DEPLOY           │                           │ │
│              │  Rolling → EKS staging      │                           │ │
│              │  Smoke tests (automated)    │                           │ │
│              │  Slack notification         │                           │ │
│              └──────────────┬──────────────┘                           │ │
│                             │  manual approval gate (1 click)          │ │
│              ┌──────────────▼──────────────┐                           │ │
│              │    PRODUCTION DEPLOY        │                           │ │
│              │  Strategy: Rolling          │   ← Start here            │ │
│              │  (upgrade to B/G when       │     Simple & reliable     │ │
│              │   you hit 50K users)        │                           │ │
│              │  Auto-rollback on failure   │                           │ │
│              │  PagerDuty alert if broken  │                           │ │
│              └─────────────────────────────┘                           │ │
└─────────────────────────────────────────────────────────────────────────┘

TOOLS STACK:
  Source Control:   GitHub
  CI:               GitHub Actions (free tier covers most startups)
  Registry:         AWS ECR
  CD:               kubectl via GitHub Actions (simple start)
                    → Argo CD when team grows (GitOps)
  Kubernetes:       EKS (managed, less ops burden)
  Monitoring:       Grafana Cloud (free tier) + Loki (logs)
  Alerting:         PagerDuty (or Slack + GitHub Actions on-call)
  Secrets:          AWS Secrets Manager + External Secrets Operator
  Security:         Trivy (image scanning, free)
```

### Complete Startup Pipeline YAML

```yaml
# .github/workflows/startup-pipeline.yml
name: Startup CI/CD Pipeline

on:
  push:
    branches: [main, staging]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: myapp
  EKS_CLUSTER: myapp-cluster

jobs:
  # ── CI: Runs on every push and PR ──────────────────────────────────
  ci:
    name: CI — Test & Build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: "pip"

      - name: Install dependencies
        run: pip install -r requirements.txt -r requirements-dev.txt

      - name: Lint
        run: |
          ruff check .
          black --check .

      - name: Unit tests with coverage
        run: |
          pytest tests/unit/ \
            --cov=app \
            --cov-report=xml \
            --cov-fail-under=80   # Fail if coverage drops below 80%

      - name: Upload coverage
        uses: codecov/codecov-action@v4

      - name: Configure AWS credentials
        if: github.event_name == 'push'
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to ECR
        if: github.event_name == 'push'
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push image
        if: github.event_name == 'push'
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build \
            --build-arg BUILD_SHA=$IMAGE_TAG \
            --target production \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:latest \
            .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest

      - name: Trivy security scan
        if: github.event_name == 'push'
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
          format: "table"
          exit-code: "1"          # Fail on HIGH/CRITICAL CVEs
          severity: "HIGH,CRITICAL"

  # ── CD: Staging ──────────────────────────────────────────────────────
  deploy-staging:
    name: Deploy → Staging
    needs: ci
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging          # GitHub Environment (can add protection rules)
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS and kubectl
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions
          aws-region: ${{ env.AWS_REGION }}

      - run: aws eks update-kubeconfig --name ${{ env.EKS_CLUSTER }}

      - name: Deploy to staging (Rolling)
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ secrets.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }} \
            -n staging
          kubectl rollout status deployment/myapp -n staging --timeout=5m

      - name: Smoke tests
        run: |
          ENDPOINT=$(kubectl get svc myapp -n staging \
            -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
          curl -f "https://$ENDPOINT/health"
          curl -f "https://$ENDPOINT/api/v1/ping"

      - name: Notify Slack
        uses: rtCamp/action-slack-notify@v2
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
          SLACK_MESSAGE: "✅ Staging deploy complete: ${{ github.sha }}"

  # ── CD: Production ───────────────────────────────────────────────────
  deploy-production:
    name: Deploy → Production
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com      # Shown in GitHub Deployments tab
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS and kubectl
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-prod
          aws-region: ${{ env.AWS_REGION }}

      - run: aws eks update-kubeconfig --name ${{ env.EKS_CLUSTER }}-prod

      - name: Rolling deploy to production
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ secrets.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }} \
            -n production
          kubectl rollout status deployment/myapp -n production --timeout=10m

      - name: Production smoke test
        run: |
          sleep 30   # Wait for traffic to shift
          curl -f "https://myapp.com/health"

      - name: Auto-rollback on failure
        if: failure()
        run: |
          kubectl rollout undo deployment/myapp -n production
          # Send PagerDuty alert
          curl -X POST ${{ secrets.PAGERDUTY_ENDPOINT }} \
            -H "Content-Type: application/json" \
            -d '{"event_action":"trigger","payload":{"summary":"Production deploy failed"}}'
```

---

## 12. CI/CD Pipeline Proposal — Enterprise (Top 10 Company)

### Context

- Team: 500-5000 engineers
- Services: 500-5000 microservices
- Scale: 10M–1B+ users
- Budget: Infrastructure cost matters but reliability > cost
- Velocity: Multiple deploys/minute across all teams
- Compliance: SOC2, ISO 27001, PCI-DSS, GDPR

### The Proposed Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    ENTERPRISE CI/CD PLATFORM                              │
│                                                                           │
│  100s of Dev Teams                                                        │
│  └─▶ GitHub Enterprise / GitLab ──▶ Platform-provided pipeline templates │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │              INTERNAL DEVELOPER PLATFORM (IDP)                      │ │
│  │              (Backstage / Port / Cortex)                            │ │
│  │  Service catalog, golden paths, self-service infra                  │ │
│  └──────────────────────────┬──────────────────────────────────────────┘ │
│                             │                                             │
│  ┌──────────────────────────▼──────────────────────────────────────────┐ │
│  │                    CI PLATFORM                                       │ │
│  │            Jenkins / GitHub Actions Enterprise / Tekton             │ │
│  │                                                                      │ │
│  │  Stage 1: Pre-commit hooks (client-side gates)                       │ │
│  │  Stage 2: Build + SAST (Sonarqube/Semgrep)                          │ │
│  │  Stage 3: Unit + integration tests (parallel matrix)                │ │
│  │  Stage 4: Container build (multi-arch AMD64 + ARM64)                │ │
│  │  Stage 5: Image sign (Sigstore/Cosign — supply chain security)      │ │
│  │  Stage 6: DAST scan (OWASP ZAP)                                     │ │
│  │  Stage 7: Push to internal registry (Harbor / ECR)                  │ │
│  └──────────────────────────┬──────────────────────────────────────────┘ │
│                             │                                             │
│  ┌──────────────────────────▼──────────────────────────────────────────┐ │
│  │                    CD PLATFORM (GitOps)                              │ │
│  │                    Argo CD / Flux                                    │ │
│  │                                                                      │ │
│  │  Environments:                                                        │ │
│  │  dev → qa → staging → canary (5%) → canary (25%) → production       │ │
│  │                                                                      │ │
│  │  Promotion gates:                                                    │ │
│  │  → Automated: Prometheus metrics must pass AnalysisTemplate         │ │
│  │  → Manual: Security team approval for production                    │ │
│  │  → Compliance: Change management ticket (ServiceNow integration)    │ │
│  └──────────────────────────┬──────────────────────────────────────────┘ │
│                             │                                             │
│  ┌──────────────────────────▼──────────────────────────────────────────┐ │
│  │               PRODUCTION ENVIRONMENT                                 │ │
│  │               Multi-region EKS (us-east-1, eu-west-1, ap-east-1)   │ │
│  │               Strategy: Canary + Progressive Delivery               │ │
│  │               Argo Rollouts for automated promotion/rollback        │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                           │
│  OBSERVABILITY STACK:                                                     │
│  Metrics:   Prometheus + Thanos (long-term storage) + Grafana            │
│  Logs:      Fluentd → OpenSearch / Splunk                               │
│  Traces:    OpenTelemetry → Jaeger / Tempo                               │
│  Alerts:    PagerDuty + OpsGenie                                         │
│  SLOs:      Sloth (SLO as code) → burns down error budgets              │
│                                                                           │
│  SECURITY:                                                                │
│  Image signing:      Cosign + policy enforcement (Kyverno)              │
│  Secret management:  HashiCorp Vault + External Secrets Operator        │
│  Policy as code:     OPA Gatekeeper / Kyverno                           │
│  Network policy:     Cilium (eBPF-based, L7 aware)                      │
│  Supply chain:       SBOM generation (Syft) + Grype scanning            │
└──────────────────────────────────────────────────────────────────────────┘
```

### Enterprise Pipeline as Code (Tekton / Argo Workflows)

```yaml
# enterprise-pipeline-template.yaml
# This is the GOLDEN PATH template every team uses
# Teams don't write their own pipelines — they use this

apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: enterprise-ci-pipeline
  namespace: argo-workflows
spec:
  entrypoint: ci-pipeline
  templates:
    - name: ci-pipeline
      dag:
        tasks:
          - name: sast-scan
            template: sonarqube-scan
          - name: unit-tests
            template: run-tests
            arguments:
              parameters:
                - name: test-command
                  value: "pytest tests/unit/ --cov --cov-fail-under=80"
          - name: integration-tests
            template: run-tests
            dependencies: [unit-tests]
            arguments:
              parameters:
                - name: test-command
                  value: "pytest tests/integration/"
          - name: build-image
            template: kaniko-build
            dependencies: [sast-scan, unit-tests]
          - name: sign-image
            template: cosign-sign
            dependencies: [build-image]
          - name: dast-scan
            template: zap-scan
            dependencies: [build-image]
          - name: promote-staging
            template: argo-cd-sync
            dependencies: [sign-image, dast-scan, integration-tests]
            arguments:
              parameters:
                - name: environment
                  value: staging

---
# Enterprise Argo Rollout with full observability gates:
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-service
  namespace: production
spec:
  replicas: 50          # Large scale — careful with canary steps
  strategy:
    canary:
      maxSurge: "20%"
      maxUnavailable: 0
      steps:
        - setWeight: 1            # 1% — internal traffic only
        - pause: {duration: 5m}
        - setWeight: 5            # 5% real users
        - pause: {duration: 15m}
        - setWeight: 20
        - pause: {duration: 30m}
        - setWeight: 50
        - pause: {}               # Indefinite — needs SRE approval
        - setWeight: 100

      analysis:
        templates:
          - templateName: payment-success-rate
          - templateName: p99-latency
          - templateName: business-metrics   # Conversion rate, revenue/min
        startingStep: 2
        args:
          - name: service-name
            value: payment-service

      trafficRouting:
        istio:
          virtualService:
            name: payment-vs
          destinationRule:
            name: payment-dr
            stableSubsetName: stable
            canarySubsetName: canary
```

### Enterprise Security Gates

```yaml
# kyverno-policy.yaml — Policy enforcement on every deploy
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: enterprise-deployment-standards
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-signed-images
      match:
        resources:
          kinds: [Pod]
      verifyImages:
        - image: "*"
          attestors:
            - entries:
                - keyless:
                    rekor: {url: "https://rekor.sigstore.dev"}
                    issuer: "https://token.actions.githubusercontent.com"
                    subject: "https://github.com/mycompany/*"

    - name: require-resource-limits
      validate:
        message: "CPU and memory limits are required"
        pattern:
          spec:
            containers:
              - resources:
                  limits:
                    memory: "?*"
                    cpu: "?*"

    - name: no-latest-tag
      validate:
        message: "Do not use :latest tag in production"
        deny:
          conditions:
            - key: "{{request.object.spec.containers[].image}}"
              operator: AnyIn
              value: ["*:latest"]
```

---

## 13. Database Migrations — The Part Everyone Forgets

**The #1 reason deployment strategies fail in production.**

```
THE GOLDEN RULE OF DB MIGRATIONS:
  Schema changes must be BACKWARD COMPATIBLE
  with the version currently in production.

  Deploy happens BEFORE migration runs.
  Both old and new code must work with the schema.

THE EXPAND/CONTRACT PATTERN (4-phase migration):

Phase 1: EXPAND (backward compatible change)
  Deploy: v1 code unchanged
  Migrate: ADD new column "user_email" (nullable, old code ignores it)
  Both v1 and v2 code work ✅

Phase 2: MIGRATE DATA (background job)
  Backfill: Copy data from "email" → "user_email"
  No deploy needed

Phase 3: SWITCH (deploy new code)
  Deploy: v2 code reads "user_email" instead of "email"
  Both columns exist — either code version works ✅

Phase 4: CONTRACT (cleanup)
  Deploy: v3 code (or keep v2)
  Migrate: DROP old "email" column
  OLD CODE CANNOT BE DEPLOYED NOW — only after this phase is stable
```

### Migration Safety Checklist

```
SAFE MIGRATION OPERATIONS:
  ✅ ADD column (nullable or with default)
  ✅ ADD index (use CONCURRENTLY in Postgres)
  ✅ ADD table
  ✅ ADD new enum value (at end)
  ✅ RENAME table (with view maintaining old name)

DANGEROUS — NEEDS EXPAND/CONTRACT:
  ❌ DROP column (old code might still read it)
  ❌ RENAME column (both old and new code must handle transition)
  ❌ CHANGE column type (data loss risk)
  ❌ ADD NOT NULL constraint without default

MIGRATION TOOLS:
  Python:  Alembic (with --autogenerate)
  Node:    Knex / Prisma Migrate
  Go:      golang-migrate / Atlas
  Java:    Flyway / Liquibase
  Ruby:    Rails ActiveRecord migrations
```

```yaml
# Kubernetes Job for migrations (runs BEFORE app deploy):
# .github/workflows/deploy.yml
- name: Run database migration
  run: |
    kubectl run migration-${{ github.sha }} \
      --image=$ECR_REGISTRY/myapp:${{ github.sha }} \
      --restart=Never \
      --namespace=production \
      --env="DATABASE_URL=${{ secrets.DATABASE_URL }}" \
      -- python -m alembic upgrade head

    # Wait for migration to complete:
    kubectl wait pod/migration-${{ github.sha }} \
      --for=condition=complete \
      --timeout=5m \
      -n production

    # Check exit code:
    kubectl logs migration-${{ github.sha }} -n production

    # Cleanup:
    kubectl delete pod migration-${{ github.sha }} -n production
```

---

## 14. Rollback Strategies — The Thing You Need Before You Deploy

```
IF YOU DON'T HAVE A ROLLBACK PLAN BEFORE YOU DEPLOY,
YOU DON'T HAVE PERMISSION TO DEPLOY.
        — Every experienced SRE, eventually
```

### Rollback by Strategy

```
STRATEGY         ROLLBACK METHOD              TIME     NOTES
────────────     ──────────────────────────   ──────   ───────────────────────
Recreate         kubectl rollout undo         2-5 min  Downtime during rollback
Rolling          kubectl rollout undo         30s-2m   Zero downtime
Blue/Green       Switch service selector back < 5s     Instant — old env still up
Canary           Set canary weight to 0       < 5s     1% users affected max
A/B Test         Disable variant flag         < 1s     Feature flag toggle
Shadow           Nothing — users unaffected   N/A      Shadow was invisible
Feature Flags    Toggle flag off              < 1s     No deploy needed
```

### The Rollback Runbook (Save This)

```bash
# ROLLING / RECREATE ROLLBACK:
kubectl rollout undo deployment/<name> -n <namespace>
# Or to a specific revision:
kubectl rollout history deployment/<name> -n <namespace>
kubectl rollout undo deployment/<name> --to-revision=3 -n <namespace>

# BLUE/GREEN ROLLBACK:
kubectl patch svc myapp-svc \
  -p '{"spec":{"selector":{"slot":"blue"}}}'  # Switch back to blue

# CANARY ROLLBACK (Nginx):
kubectl annotate ingress myapp-canary \
  nginx.ingress.kubernetes.io/canary-weight="0" --overwrite

# CANARY ROLLBACK (Argo Rollouts):
kubectl argo rollouts abort myapp -n production

# VERIFY ROLLBACK:
kubectl rollout status deployment/<name> -n <namespace>
kubectl get pods -n <namespace>   # All pods should be on old version
```

---

## 15. Metrics That Prove Your Strategy Is Working

### The Four Golden Signals (SRE Book)

```
1. LATENCY
   → p50, p95, p99 response time
   → Alert if p99 increases by > 20% during deploy
   → Prometheus query:
     histogram_quantile(0.99,
       rate(http_request_duration_seconds_bucket[5m]))

2. TRAFFIC
   → Requests per second
   → Verify traffic shifted correctly during canary/blue-green
   → Prometheus query:
     sum(rate(http_requests_total[5m])) by (version)

3. ERRORS
   → HTTP 5xx error rate
   → ALERT THRESHOLD: Error rate > 1% for > 2 minutes = rollback
   → Prometheus query:
     sum(rate(http_requests_total{status=~"5.."}[5m])) /
     sum(rate(http_requests_total[5m]))

4. SATURATION
   → CPU, memory, connection pool utilization
   → New version might leak memory or use more CPU
   → Prometheus query:
     container_memory_working_set_bytes{container="myapp"} /
     container_spec_memory_limit_bytes
```

### DORA Metrics — Proving Your CI/CD Is World-Class

```
DORA METRIC              ELITE PERFORMER        HOW TO MEASURE
───────────────────────  ─────────────────────  ──────────────────────────────
Deployment Frequency     Multiple/day           GitHub releases per day
Lead Time for Changes    < 1 hour               PR merged → production live
Change Failure Rate      < 5%                   Rollbacks / total deploys
MTTR (recovery time)     < 1 hour               Incident open → resolution

Track these in:
  LinearB, Sleuth, Faros, or custom Grafana dashboard
  from GitHub API + PagerDuty + Kubernetes events
```

### Deployment Dashboard — Key Panels

```
┌─────────────────────────────────────────────────────────────────────┐
│  DEPLOYMENT HEALTH DASHBOARD                                         │
│                                                                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────────────┐ │
│  │ Deploy Status   │  │ Error Rate      │  │ P99 Latency          │ │
│  │ ✅ v1.3.0 LIVE  │  │ 0.02%  ████▏   │  │ 142ms ████████▏      │ │
│  │ 100% traffic    │  │ OK (< 1%)       │  │ OK (< 200ms)         │ │
│  └─────────────────┘  └─────────────────┘  └──────────────────────┘ │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  CANARY ANALYSIS (if running)                                   │ │
│  │  Stable:  Error=0.01%, P99=138ms                                │ │
│  │  Canary:  Error=0.03%, P99=145ms   ← within acceptable range   │ │
│  │  Weight:  10% → [PROMOTE] [ROLLBACK]                           │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 16. The Architecture Review Pitch — Templates

### Startup Pitch (5-minute version)

> **"Here's what I'm proposing for our deployment strategy."**
>
> We currently deploy with Recreate — this causes downtime every deploy. We ship 3 times a week.  
> That's 3 potential outages per week. As we approach 10K users, that's unacceptable.
>
> **Proposal: Rolling deploys on EKS with automated rollback.**
>
> - Zero downtime. Kubernetes replaces one pod at a time.
> - If the new pod fails its health check, the rollout pauses. GitHub Actions calls `kubectl rollout undo` automatically.
> - Our engineers need to write zero new code — it's a 10-line YAML change to the Deployment spec.
> - We add a smoke test step in GitHub Actions (5 lines, 30 minutes to implement).
>
> **Investment:** 2 engineer-days.  
> **Return:** Zero downtime deployments forever. When we hit 50K users, we upgrade to Blue/Green — same team, no new tools.

### Enterprise Pitch (Architecture Review Board)

> **"Deployment Strategy Modernization — Platform Engineering Q3 Initiative"**
>
> **Current State:** Teams manage their own deployment configs. 47 different deployment patterns across 200 services. Average MTTR: 45 minutes. Change failure rate: 18%.
>
> **Target State:** Golden path adoption. Canary + Argo Rollouts for all P0/P1 services. Feature flags for all product changes. GitOps via Argo CD.
>
> **Proposed Stack:**
> - Argo CD (GitOps) + Argo Rollouts (progressive delivery)
> - Prometheus/Grafana for automatic canary analysis
> - Kyverno for policy enforcement on all deploys
> - LaunchDarkly for feature flags (FIPS-compliant, SOC2 certified)
>
> **Expected Outcomes (6 months):**
> - Deployment frequency: 2x/week → 5x/day per team
> - Change failure rate: 18% → < 5%
> - MTTR: 45 min → < 10 min (automatic rollback)
> - Zero high-severity incidents from bad deploys (canary catches at 1%)
>
> **Investment:** 3 platform engineers × 2 quarters.  
> **Risk mitigation:** Phased rollout by team criticality. Opt-in for first 3 months.

---

## 17. Quick Reference Decision Card

```
                    CHOOSE YOUR DEPLOYMENT STRATEGY
                    ─────────────────────────────────

  Can you tolerate ANY downtime?
  │
  ├── YES (dev/staging, internal tools, scheduled window)
  │   └── RECREATE ← simplest, no overhead
  │
  └── NO (production, user-facing, SLA-bound)
      │
      ├── Is your API backward compatible between versions?
      │   │
      │   ├── YES
      │   │   │
      │   │   ├── Do you have < 50K users and small team?
      │   │   │   └── ROLLING ← start here, default choice
      │   │   │
      │   │   └── Do you have > 50K users, need blast radius control?
      │   │       └── CANARY ← with Argo Rollouts + Prometheus analysis
      │   │
      │   └── NO (breaking changes, new DB schema, protocol change)
      │       └── BLUE/GREEN ← atomic switch, instant rollback
      │
      ├── Do you need to test with REAL production traffic safely?
      │   └── SHADOW ← mirror traffic, discard responses
      │
      ├── Do you want to measure BUSINESS impact of a change?
      │   └── A/B TESTING ← header/cookie routing + stats analysis
      │
      └── Do you want to deploy code without releasing features?
          └── FEATURE FLAGS ← decouple deploy from release

─────────────────────────────────────────────────────────────────────
STARTUP (< 50K users):    Rolling → Blue/Green (when team is ready)
GROWING (50K-1M users):   Blue/Green + Canary for critical services
ENTERPRISE (> 1M users):  Canary + Feature Flags + Progressive Delivery
─────────────────────────────────────────────────────────────────────

TOOLS CHEAT SHEET:
  Rolling/Recreate:    kubectl (built-in)
  Blue/Green:          kubectl patch svc + 2 Deployments
  Canary:              Argo Rollouts OR Nginx canary annotations
  Feature Flags:       LaunchDarkly (hosted) / Flagsmith (self-hosted)
  GitOps CD:           Argo CD / Flux
  Pipeline CI:         GitHub Actions / GitLab CI / Tekton
  Image Security:      Trivy + Cosign
  Policy Enforcement:  Kyverno / OPA Gatekeeper
  Observability:       Prometheus + Grafana + OpenTelemetry
```


---

*Part of the [devops-field-guide](../) series · For DevOps engineers who want to propose, implement, and defend deployment decisions with confidence.*

⭐ Star the repo if your manager approved your deployment strategy proposal after reading this.
