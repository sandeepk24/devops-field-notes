# Kubernetes Ingress Mastery: Pass the CKAD Exam AND Ship It to Production on EKS 🎯

> **Two goals. One guide.**  
> Score full marks on every CKAD Ingress question. Then take that exact knowledge and run it on a real AWS EKS cluster.  
> For DevOps engineers who want to understand it.

---

## 📋 Table of Contents

1. [The Mental Model — What Is Ingress, Really?](#1-the-mental-model)
2. [The 4-Layer Traffic Stack in Kubernetes](#2-the-4-layer-traffic-stack)
3. [Ingress vs Service — The Critical Distinction](#3-ingress-vs-service)
4. [The Ingress Controller — The Part Most Guides Skip](#4-the-ingress-controller)
5. [Anatomy of an Ingress YAML — Every Field](#5-anatomy-of-an-ingress-yaml)
6. [CKAD Exam: Creating an Ingress (Question Type 1)](#6-ckad-creating-an-ingress)
7. [CKAD Exam: Troubleshooting Ingress (Question Types 2 & 3)](#7-ckad-troubleshooting)
8. [Path Types — The Exam Trap](#8-path-types)
9. [Name-Based Virtual Hosting (Multiple Domains)](#9-name-based-virtual-hosting)
10. [TLS/HTTPS on Ingress](#10-tlshttps-on-ingress)
11. [Real World: Nginx Ingress Controller on Bare Kubernetes](#11-nginx-ingress-on-kubernetes)
12. [Real World: AWS ALB Ingress Controller on EKS](#12-aws-alb-ingress-on-eks)
13. [Ingress Annotations — The Power Layer](#13-ingress-annotations)
14. [End-to-End: Deploy a Full App with Ingress on EKS](#14-end-to-end-on-eks)
15. [Common Failures & How to Debug Them](#15-debugging-guide)
16. [CKAD Exam Cheat Sheet — Last-Night Review](#16-ckad-cheat-sheet)

---

## 1. The Mental Model

Before any YAML, build the right mental model. This is what makes everything else stick.

### The Apartment Building Analogy

```
WITHOUT INGRESS (the old way):

  Internet → Service A (LoadBalancer) → App A   $$$
  Internet → Service B (LoadBalancer) → App B   $$$
  Internet → Service C (LoadBalancer) → App C   $$$

  Each service gets its OWN cloud load balancer.
  On AWS: 3 separate ALBs = ~$50/month each = $150/month
  On a real cluster with 20 services = $1,000+/month just on LBs.


WITH INGRESS (the smart way):

  Internet
      │
      ▼
  ONE Load Balancer (Ingress Controller)   $
      │
      ├─ /api        → Service A (ClusterIP)
      ├─ /web        → Service B (ClusterIP)
      └─ /admin      → Service C (ClusterIP)

  One entry point. Routes traffic by URL path or hostname.
  Internal services use cheap ClusterIP — no cloud LBs needed.
```

**Ingress = a smart traffic cop sitting at the door of your cluster.**

It reads the HTTP request (the URL path, the hostname, the headers) and decides which internal Service to forward it to.

```
Visitor arrives:   GET https://myapp.com/api/users

Ingress reads:
  Host: myapp.com        → matches rule for myapp.com
  Path: /api/users       → matches prefix /api
  → Forward to: api-service:8080

Visitor arrives:   GET https://myapp.com/dashboard

Ingress reads:
  Host: myapp.com        → matches rule for myapp.com
  Path: /dashboard       → matches prefix /dashboard
  → Forward to: frontend-service:3000
```

---

## 2. The 4-Layer Traffic Stack in Kubernetes

Understanding this stack makes CKAD troubleshooting questions straightforward.

```
┌─────────────────────────────────────────────────────────────────────┐
│  LAYER 4: INTERNET / CLIENT                                         │
│  Browser, curl, mobile app                                          │
│  Sends HTTP/HTTPS requests to a domain or IP                        │
└─────────────────────────────────┬───────────────────────────────────┘
                                   │ DNS resolves to Load Balancer IP
┌─────────────────────────────────▼───────────────────────────────────┐
│  LAYER 3: INGRESS CONTROLLER                                        │
│  (Nginx, ALB, Traefik, HAProxy, Kong...)                            │
│  A running Pod (or external LB) that:                               │
│  - Terminates TLS (HTTPS)                                           │
│  - Reads HTTP Host header and URL path                              │
│  - Looks up matching Ingress rules                                  │
│  - Proxies to the correct Service                                   │
└─────────────────────────────────┬───────────────────────────────────┘
                                   │ matches Ingress rule → forwards to Service
┌─────────────────────────────────▼───────────────────────────────────┐
│  LAYER 2: INGRESS RESOURCE (the YAML you write)                     │
│  A Kubernetes object that defines ROUTING RULES                     │
│  "if host=api.myapp.com AND path=/users → send to user-svc:80"      │
│  It's just config — by itself it does NOTHING                       │
│  The Ingress Controller READS this config and acts on it            │
└─────────────────────────────────┬───────────────────────────────────┘
                                   │ routes to matching Service
┌─────────────────────────────────▼───────────────────────────────────┐
│  LAYER 1: SERVICE (ClusterIP)                                       │
│  Stable internal IP + DNS for a set of Pods                         │
│  Load balances across healthy Pod replicas                          │
│  Selects Pods by label selector                                     │
└─────────────────────────────────┬───────────────────────────────────┘
                                   │ forwards to matching Pod
┌─────────────────────────────────▼───────────────────────────────────┐
│  LAYER 0: POD                                                       │
│  Your actual application container                                  │
│  Listening on a port inside the container                           │
└─────────────────────────────────────────────────────────────────────┘

EXAM TIP: When troubleshooting, ALWAYS check bottom-up:
  Pod healthy? → Service selects Pod? → Ingress points to Service? → Controller running?
```

---

## 3. Ingress vs Service — The Critical Distinction

This confusion causes failures in both the exam AND production.

```
┌────────────────────┬──────────────────────────────┬──────────────────────────────┐
│ Feature            │ Service                       │ Ingress                      │
├────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ OSI Layer          │ Layer 4 (TCP/UDP)             │ Layer 7 (HTTP/HTTPS)         │
│ Routing basis      │ IP + Port                     │ Host header + URL path       │
│ Protocols          │ TCP, UDP, SCTP                │ HTTP, HTTPS only             │
│ TLS termination    │ No (passthrough only)         │ Yes                          │
│ Path-based routing │ No                            │ Yes                          │
│ Host-based routing │ No                            │ Yes                          │
│ Cost (cloud)       │ 1 LB per LoadBalancer svc     │ 1 LB for all services        │
│ Works without LB   │ ClusterIP (internal only)     │ Needs a controller           │
│ CKAD objects       │ kubectl expose                │ Ingress YAML only            │
└────────────────────┴──────────────────────────────┴──────────────────────────────┘

SERVICE TYPES refresher:
  ClusterIP   → Internal only. Used BEHIND an Ingress. (default)
  NodePort    → Opens a port on every Node. No LB. Dev/testing.
  LoadBalancer→ Provisions a cloud LB. One LB per service. Expensive.
  ExternalName→ DNS alias to external service. Rare.

RULE: In production, your apps should use ClusterIP Services.
      Ingress is how the outside world reaches them.
      You only need LoadBalancer Service for the Ingress Controller itself.
```

---

## 4. The Ingress Controller — The Part Most Guides Skip

**This is the #1 concept gap that causes exam confusion and production failures.**

```
THE CRITICAL INSIGHT:

Ingress resource (your YAML)  ≠  Ingress Controller

┌─────────────────────────────────────────────────────────────┐
│  Ingress Resource                                            │
│  Kind: Ingress                                               │
│  → Just a config file stored in etcd                        │
│  → Does NOTHING by itself                                    │
│  → Like a nginx.conf file sitting on disk without nginx     │
└─────────────────────────────────────────────────────────────┘
                         +
┌─────────────────────────────────────────────────────────────┐
│  Ingress Controller                                          │
│  → A running Pod (usually Nginx/ALB)                        │
│  → Watches the Kubernetes API for Ingress objects           │
│  → Translates Ingress rules into its own config             │
│  → Actually proxies the traffic                             │
│  → Must be installed separately — NOT included in K8s       │
└─────────────────────────────────────────────────────────────┘
                         =
                    Traffic actually routes
```

### Popular Ingress Controllers

```
┌──────────────────────┬────────────────────────────┬────────────────────────────┐
│ Controller           │ Best For                    │ EKS Usage                  │
├──────────────────────┼────────────────────────────┼────────────────────────────┤
│ Nginx Ingress        │ General purpose, self-hosted│ Install via Helm on any K8s│
│ (kubernetes/ingress- │ Most common in CKAD exam    │ NLB + Nginx pods           │
│ nginx)               │ environments                │                            │
├──────────────────────┼────────────────────────────┼────────────────────────────┤
│ AWS ALB Controller   │ EKS native                  │ Each Ingress = 1 real ALB  │
│ (aws-load-balancer-  │ Auto-creates AWS ALBs        │ Deep AWS integration       │
│ controller)          │ WAF, Cognito, ACM TLS       │                            │
├──────────────────────┼────────────────────────────┼────────────────────────────┤
│ Traefik              │ Dynamic config, Let's       │ Popular for smaller EKS    │
│                      │ Encrypt auto-TLS            │ clusters                   │
├──────────────────────┼────────────────────────────┼────────────────────────────┤
│ Kong                 │ API Gateway features        │ Enterprise API management  │
│                      │ Rate limiting, auth plugins │                            │
└──────────────────────┴────────────────────────────┴────────────────────────────┘

CKAD EXAM NOTE:
  The exam environment typically has Nginx Ingress pre-installed.
  You WRITE Ingress resources. You don't install controllers.
  If a question says "create an Ingress" — the controller is already there.
```

### How the Controller Watches for Ingress Resources

```
Ingress Controller Pod
        │
        │  watches (LIST + WATCH)
        ▼
  Kubernetes API Server
        │
        │  returns Ingress objects
        ▼
  Ingress Controller sees new/updated Ingress
        │
        │  translates to nginx.conf / ALB rules
        ▼
  Updates its internal config
        │
        │  proxies live traffic per new rules
        ▼
  Traffic routes correctly
```

---

## 5. Anatomy of an Ingress YAML — Every Field

```yaml
apiVersion: networking.k8s.io/v1          # ← Always this. Not extensions/v1beta1 (deprecated)
kind: Ingress
metadata:
  name: my-ingress
  namespace: production                   # ← Must be same namespace as the backend Services
  annotations:
    # Annotations are controller-specific config — more on these in Section 13
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

spec:
  ingressClassName: nginx                 # ← Which controller handles this Ingress
                                          #   "nginx" for Nginx Ingress
                                          #   "alb"   for AWS ALB Controller
                                          #   Check: kubectl get ingressclass

  # ── DEFAULT BACKEND ─────────────────────────────────────────────
  # Catch-all: if no rules match, send here (404 page, error handler)
  defaultBackend:
    service:
      name: default-404-service
      port:
        number: 80

  # ── TLS (HTTPS) ─────────────────────────────────────────────────
  tls:
    - hosts:
        - myapp.com
        - api.myapp.com
      secretName: myapp-tls-secret        # ← K8s Secret of type kubernetes.io/tls
                                          #   Contains tls.crt and tls.key

  # ── RULES ───────────────────────────────────────────────────────
  rules:
    # Rule 1: Host-based routing
    - host: myapp.com                     # ← matches HTTP Host header exactly
                                          #   Omit host to match ALL hosts
      http:
        paths:
          # Path 1: Route /api/* to api-service
          - path: /api                    # ← URL path prefix to match
            pathType: Prefix              # ← Prefix, Exact, or ImplementationSpecific
            backend:
              service:
                name: api-service         # ← Name of the Service to route to
                port:
                  number: 8080            # ← Service port (not container port!)

          # Path 2: Route / to frontend
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80

    # Rule 2: Different host, different routes
    - host: admin.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 3000
```

### Field-by-Field Reference Card

```
apiVersion: networking.k8s.io/v1
  → Always use this. The old extensions/v1beta1 is removed in K8s 1.22+
  → CKAD exam uses 1.24+ — always use networking.k8s.io/v1

spec.ingressClassName
  → Tells Kubernetes WHICH controller should handle this Ingress
  → Without it: the controller with annotation "is-default-class: true" handles it
  → Check available classes: kubectl get ingressclass

spec.rules[].host
  → Exact hostname match. "myapp.com" ≠ "www.myapp.com"
  → Supports wildcards: "*.myapp.com" matches "api.myapp.com"
  → Omit for wildcard matching all hosts

spec.rules[].http.paths[].pathType
  → Prefix: /api matches /api, /api/, /api/users, /api/v2/anything
  → Exact:  /api matches ONLY /api (not /api/ or /api/users)
  → ImplementationSpecific: controller decides (avoid in exam)

spec.rules[].http.paths[].backend.service.port
  → MUST be the Service port, NOT the container/pod port
  → If Service spec.ports[].port: 80 → use number: 80
```

---

## 6. CKAD Exam: Creating an Ingress (Question Type 1)

### The Exam Question Format

> *"Create an Ingress named `webapp-ingress` in namespace `web` that routes traffic from host `webapp.internal` path `/` to service `webapp-svc` on port `80`. Use the nginx ingress class."*

### The Fastest Way: `kubectl create ingress`

```bash
# Syntax:
kubectl create ingress <name> \
  --rule="<host>/<path>=<service>:<port>" \
  --class=<ingressclass> \
  --namespace=<namespace>

# For the exam question above:
kubectl create ingress webapp-ingress \
  --rule="webapp.internal/*=webapp-svc:80" \
  --class=nginx \
  --namespace=web

# Multiple paths:
kubectl create ingress multi-ingress \
  --rule="myapp.com/api*=api-svc:8080" \
  --rule="myapp.com/*=frontend-svc:3000" \
  --namespace=production

# With TLS:
kubectl create ingress secure-ingress \
  --rule="myapp.com/*=frontend-svc:80,tls=myapp-tls" \
  --class=nginx \
  --namespace=production

# Generate YAML without applying (--dry-run):
kubectl create ingress webapp-ingress \
  --rule="webapp.internal/*=webapp-svc:80" \
  --class=nginx \
  --namespace=web \
  --dry-run=client -o yaml
```

### The `--rule` Flag Syntax Decoded

```
--rule="<host>/<path>=<service>:<port>"
          │       │       │        │
          │       │       │        └── Service port number
          │       │       └─────────── Service name
          │       └─────────────────── Path (use * for prefix match)
          └─────────────────────────── Hostname (omit for any host)

Examples:
  --rule="/*=svc:80"                  # Any host, any path → svc:80
  --rule="myapp.com/*=svc:80"         # Only myapp.com → svc:80
  --rule="myapp.com/api*=api-svc:8080" # myapp.com/api/* → api-svc:8080
  --rule="myapp.com/login=auth:3000"  # Exact path /login → auth:3000
```

### Complete Exam Workflow (Under 2 Minutes)

```bash
# Step 1: Verify the namespace exists
kubectl get namespace web

# Step 2: Verify the service exists and check its port
kubectl get svc webapp-svc -n web
# NAME         TYPE        CLUSTER-IP      PORT(S)
# webapp-svc   ClusterIP   10.100.200.50   80/TCP   ← use port 80

# Step 3: Check available IngressClass
kubectl get ingressclass
# NAME    CONTROLLER                      DEFAULT
# nginx   k8s.io/ingress-nginx            true

# Step 4: Create the Ingress
kubectl create ingress webapp-ingress \
  --rule="webapp.internal/*=webapp-svc:80" \
  --class=nginx \
  --namespace=web

# Step 5: Verify
kubectl get ingress -n web
# NAME             CLASS   HOSTS             ADDRESS        PORTS   AGE
# webapp-ingress   nginx   webapp.internal   10.0.1.5       80      10s

kubectl describe ingress webapp-ingress -n web
# Rules:
#   Host             Path  Backends
#   ────             ────  ────────
#   webapp.internal  /     webapp-svc:80 (10.244.0.5:8080)   ← pod IP shows if working
```

### When to Write YAML Instead of Using kubectl

If the question requires:
- `defaultBackend`
- Specific `pathType: Exact`
- Multiple annotations
- Complex TLS config

Use `kubectl create ingress --dry-run=client -o yaml > ingress.yaml` then edit it.

```bash
# Generate base then edit:
kubectl create ingress webapp-ingress \
  --rule="webapp.internal/*=webapp-svc:80" \
  --class=nginx \
  --namespace=web \
  --dry-run=client -o yaml > ingress.yaml

vim ingress.yaml   # Make edits

kubectl apply -f ingress.yaml
```

---

## 7. CKAD Exam: Troubleshooting Ingress (Question Types 2 & 3)

### The Exam Troubleshooting Scenario

> *"An Ingress named `app-ingress` in namespace `staging` is not routing traffic correctly. Users report a 503 error when accessing `app.internal/`. Fix it."*

### The Systematic Debug Framework

**Always work bottom-up through the 4 layers.**

```
STEP 1: Is the Pod running?
────────────────────────────────────────────────────────────
kubectl get pods -n staging
kubectl describe pod <pod-name> -n staging
# Look for: Running status, Ready 1/1

STEP 2: Does the Service select the Pod correctly?
────────────────────────────────────────────────────────────
kubectl get svc -n staging
kubectl describe svc <service-name> -n staging
# KEY CHECK: "Endpoints" section
# If Endpoints: <none>  → Service labels don't match Pod labels → BUG IS HERE

# Verify label matching:
kubectl get pods -n staging --show-labels
# Compare Pod labels with Service selector

STEP 3: Does the Ingress point to the right Service and Port?
────────────────────────────────────────────────────────────
kubectl describe ingress app-ingress -n staging
# KEY CHECK: "Backends" section
# webapp-svc:80 (10.244.0.5:8080) ← pod IP shows = working
# webapp-svc:80 (<none>)           ← no endpoints = broken

# Common bugs in the Ingress:
# - Wrong service name (typo)
# - Wrong service port (using container port instead of service port)
# - Wrong namespace
# - Wrong pathType causing no match

STEP 4: Is the Ingress Controller running?
────────────────────────────────────────────────────────────
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx <controller-pod> | tail -50
# Look for errors related to your ingress name
```

### The Most Common CKAD Troubleshooting Bugs

#### Bug Type 1: Wrong Service Name

```yaml
# BROKEN — typo in service name
spec:
  rules:
    - host: app.internal
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-servce     # ← TYPO: "servce" not "service"
                port:
                  number: 80

# FIX:
kubectl get svc -n staging           # Find the real service name
kubectl edit ingress app-ingress -n staging
# Change: app-servce → app-service
```

#### Bug Type 2: Wrong Service Port

```yaml
# BROKEN — using container port, not service port
spec:
  rules:
    - http:
        paths:
          - backend:
              service:
                name: api-service
                port:
                  number: 8080       # ← Container listens on 8080

# Check the ACTUAL service port:
kubectl get svc api-service -n staging
# PORT(S): 80/TCP     ← Service port is 80, not 8080!
# 80 is the service port, it forwards to 8080 on the pod

# FIX: Change port.number to 80
```

#### Bug Type 3: Service Selector Doesn't Match Pod Labels

```bash
# Check the service selector:
kubectl describe svc app-service -n staging
# Selector: app=webapp    ← looking for this label

# Check pod labels:
kubectl get pods -n staging --show-labels
# NAME          LABELS
# app-pod-xxx   app=web-app   ← MISMATCH: "web-app" vs "webapp"

# FIX Option A: Update the Service selector
kubectl patch svc app-service -n staging \
  -p '{"spec":{"selector":{"app":"web-app"}}}'

# FIX Option B: Update the Pod labels (via Deployment)
kubectl label pods app-pod-xxx app=webapp -n staging
```

#### Bug Type 4: Wrong Namespace

```yaml
# BROKEN — Ingress in different namespace than Service
metadata:
  namespace: default           # ← Ingress is in 'default'
spec:
  rules:
    - http:
        paths:
          - backend:
              service:
                name: app-service   # ← But this service is in 'staging'!

# RULE: Ingress MUST be in the SAME namespace as its backend Services.
# You cannot route cross-namespace with a standard Ingress.
# FIX: Move the Ingress to 'staging' namespace
```

#### Bug Type 5: Missing or Wrong IngressClass

```bash
# Symptom: Ingress created but ADDRESS column is empty after 2+ minutes
kubectl get ingress -n staging
# NAME          CLASS    HOSTS          ADDRESS    PORTS   AGE
# app-ingress   <none>   app.internal              80      5m   ← no address!

# Check ingressClassName in the YAML:
kubectl get ingress app-ingress -n staging -o yaml | grep ingressClass
# ingressClassName: wrongname    ← no controller claims this class

# Check what classes exist:
kubectl get ingressclass
# NAME    CONTROLLER
# nginx   k8s.io/ingress-nginx

# FIX:
kubectl patch ingress app-ingress -n staging \
  -p '{"spec":{"ingressClassName":"nginx"}}'
```

#### Bug Type 6: pathType Exact vs Prefix Mismatch

```yaml
# BROKEN — exact match won't catch /api/users
spec:
  rules:
    - http:
        paths:
          - path: /api
            pathType: Exact        # ← ONLY matches /api exactly
            backend:
              service:
                name: api-svc
                port:
                  number: 80

# User hits: /api/users → 404 (doesn't match Exact /api)

# FIX: Change to Prefix
            pathType: Prefix       # ← matches /api AND /api/users AND /api/v2/...
```

### Full Troubleshooting Command Sequence

```bash
# Copy-paste this entire block in the exam — covers everything:

NS=staging                          # Set your namespace
ING=app-ingress                     # Set your ingress name

echo "=== INGRESS ==="
kubectl describe ingress $ING -n $NS

echo "=== SERVICES ==="
kubectl get svc -n $NS

echo "=== ENDPOINTS ==="
kubectl get endpoints -n $NS        # Empty endpoints = selector mismatch

echo "=== PODS WITH LABELS ==="
kubectl get pods -n $NS --show-labels

echo "=== INGRESSCLASS ==="
kubectl get ingressclass

echo "=== CONTROLLER LOGS ==="
kubectl logs -n ingress-nginx \
  $(kubectl get pods -n ingress-nginx -o name | head -1) \
  --tail=30 | grep -i "error\|warn\|$ING"
```

---

## 8. Path Types — The Exam Trap

This is tested explicitly in the CKAD exam. Get it exactly right.

```
THREE PATH TYPES:

1. Prefix
───────────────────────────────────────────────────────────────
path: /app

MATCHES:          /app
                  /app/
                  /app/users
                  /app/v2/products
              `    /app/anything/deeply/nested

DOES NOT MATCH:   /applesauce    ← must be /app followed by / or end
                  /application

2. Exact
───────────────────────────────────────────────────────────────
path: /app

MATCHES:          /app

DOES NOT MATCH:   /app/
                  /app/users
                  /application

3. ImplementationSpecific
───────────────────────────────────────────────────────────────
Behavior defined by the controller.
For Nginx: same as Prefix.
AVOID in exam unless explicitly asked.
```

### Path Ordering Matters

```yaml
# When multiple paths could match, MORE SPECIFIC wins.
# Best practice: list most specific first.

spec:
  rules:
    - http:
        paths:
          - path: /api/admin          # More specific — checked first
            pathType: Prefix
            backend:
              service:
                name: admin-svc
                port:
                  number: 80

          - path: /api                # Less specific — checked second
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 8080

          - path: /                   # Catch-all — last
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 3000

# Request: /api/admin/users → matches /api/admin (first match)
# Request: /api/products    → matches /api (second match)
# Request: /dashboard       → matches / (catch-all)
```

---

## 9. Name-Based Virtual Hosting (Multiple Domains)

One Ingress controller, multiple domains — all routing to different services.

```yaml
# multi-host-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
  namespace: production
spec:
  ingressClassName: nginx
  rules:
    # Domain 1: public website
    - host: www.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: website-svc
                port:
                  number: 80

    # Domain 2: API
    - host: api.mycompany.com
      http:
        paths:
          - path: /v1
            pathType: Prefix
            backend:
              service:
                name: api-v1-svc
                port:
                  number: 8080
          - path: /v2
            pathType: Prefix
            backend:
              service:
                name: api-v2-svc
                port:
                  number: 8080

    # Domain 3: Admin panel (different subdomain)
    - host: admin.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-svc
                port:
                  number: 3000
```

```
Traffic routing:

  www.mycompany.com/about    → website-svc:80
  api.mycompany.com/v1/users → api-v1-svc:8080
  api.mycompany.com/v2/users → api-v2-svc:8080
  admin.mycompany.com/       → admin-svc:3000

All through ONE load balancer. ✅
```

---

## 10. TLS/HTTPS on Ingress

### Step 1: Create the TLS Secret

```bash
# For exam/testing: self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=myapp.com/O=myapp"

# Create the Kubernetes secret:
kubectl create secret tls myapp-tls \
  --cert=tls.crt \
  --key=tls.key \
  --namespace=production

# Verify:
kubectl get secret myapp-tls -n production
# NAME        TYPE                DATA   AGE
# myapp-tls   kubernetes.io/tls   2      5s
# The secret has two keys: tls.crt and tls.key
```

### Step 2: Reference in Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"   # HTTP → HTTPS redirect
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.com
        - www.myapp.com
      secretName: myapp-tls           # Must match secret name
  rules:
    - host: myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 80
```

### TLS Rules to Know

```
1. Secret MUST be in the SAME namespace as the Ingress
2. Secret type MUST be kubernetes.io/tls
3. Secret MUST contain keys: tls.crt and tls.key
4. The host in tls.hosts[] MUST match the rule host
5. Without ssl-redirect annotation: HTTP still works
   With ssl-redirect: HTTP redirects to HTTPS automatically

Exam shortcut to create the secret from a question prompt:
  kubectl create secret tls <name> --cert=<file> --key=<file> -n <ns>
```

---

## 11. Nginx Ingress Controller on Bare Kubernetes

### Installation

```bash
# Helm (recommended):
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.service.type=LoadBalancer

# Or bare YAML:
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/\
controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml

# Verify:
kubectl get pods -n ingress-nginx
# ingress-nginx-controller-xxx   1/1   Running

kubectl get svc -n ingress-nginx
# NAME                         TYPE           CLUSTER-IP      EXTERNAL-IP
# ingress-nginx-controller     LoadBalancer   10.100.0.50     1.2.3.4       ← public IP
```

### IngressClass Resource

```bash
# Check what was created:
kubectl get ingressclass
# NAME    CONTROLLER                      PARAMETERS   AGE
# nginx   k8s.io/ingress-nginx            <none>       2m

# If you need to create it manually:
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"  # Make it default
spec:
  controller: k8s.io/ingress-nginx
EOF
```

### Key Nginx Controller Annotations

```yaml
metadata:
  annotations:
    # Rewrite the URL path before sending to backend
    nginx.ingress.kubernetes.io/rewrite-target: /$2

    # Force HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"

    # Increase client body size (for file uploads)
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"

    # Timeouts
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "120"

    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://myapp.com"

    # Basic auth
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth-secret
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"

    # WebSocket support
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
```

### The Rewrite-Target Gotcha

```yaml
# SCENARIO: Your app expects requests at /
# But Ingress exposes it at /api
# Without rewrite: /api/users → backend receives /api/users
# With rewrite:    /api/users → backend receives /users

metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
                                                  # ↑ $2 = second capture group

spec:
  rules:
    - http:
        paths:
          - path: /api(/|$)(.*)              # $1 = /  or ""
            pathType: ImplementationSpecific  # $2 = rest of path
            backend:
              service:
                name: api-svc
                port:
                  number: 80

# Request: /api/users/123
# $1 = /
# $2 = users/123
# Rewrite: / + users/123 = /users/123
# Backend receives: /users/123 ✅
```

---

## 12. AWS ALB Ingress Controller on EKS

This is the production-grade approach most AWS shops use. Every Ingress object creates a **real AWS Application Load Balancer**.

### Architecture

```
Internet
    │
    ▼
AWS Application Load Balancer (ALB)
    │  created automatically per Ingress resource
    │
    ├── Listener: HTTP  :80  → redirect to HTTPS
    └── Listener: HTTPS :443
              │
              ├── Rule: Host=api.myapp.com, Path=/v1/*
              │         → Target Group: api-v1 pods
              │
              ├── Rule: Host=api.myapp.com, Path=/v2/*
              │         → Target Group: api-v2 pods
              │
              └── Rule: Default
                        → Target Group: frontend pods

AWS ALB Target Groups → EKS Pod IPs directly (not through Service ClusterIP)
This is called "IP mode" targeting — more efficient than instance mode
```

### Installing the AWS Load Balancer Controller

```bash
# Step 1: Create IAM policy
curl -o iam-policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/\
  main/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json

# Step 2: Create IAM role with IRSA (IAM Roles for Service Accounts)
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
CLUSTER_NAME=my-eks-cluster
REGION=us-east-1

eksctl create iamserviceaccount \
  --cluster=$CLUSTER_NAME \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Step 3: Install via Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=$REGION \
  --set vpcId=$(aws eks describe-cluster \
    --name $CLUSTER_NAME \
    --query "cluster.resourcesVpcConfig.vpcId" \
    --output text)

# Step 4: Verify
kubectl get deployment -n kube-system aws-load-balancer-controller
# NAME                           READY   UP-TO-DATE   AVAILABLE
# aws-load-balancer-controller   2/2     2            2          ✅

# Check IngressClass was created:
kubectl get ingressclass
# NAME   CONTROLLER                             DEFAULT
# alb    ingress.k8s.aws/alb                    false
```

### ALB Ingress Resource

```yaml
# alb-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-alb-ingress
  namespace: production
  annotations:
    # ── REQUIRED ──────────────────────────────────────────────────
    # The controller reads these to configure the ALB

    # ALB is internet-facing (public) or internal (private)
    alb.ingress.kubernetes.io/scheme: internet-facing
    # Options: internet-facing | internal

    # Target pods directly via their IP (recommended)
    alb.ingress.kubernetes.io/target-type: ip
    # Options: ip | instance
    # ip mode: ALB → Pod IP directly (faster, requires VPC CNI)
    # instance mode: ALB → NodePort → Pod (compatible with any CNI)

    # ── TLS WITH ACM ─────────────────────────────────────────────
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    # ACM Certificate ARN (request cert in ACM first):
    alb.ingress.kubernetes.io/certificate-arn: >-
      arn:aws:acm:us-east-1:123456789:certificate/abc123-def456

    # ── HEALTH CHECK ─────────────────────────────────────────────
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "15"
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: "5"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "3"

    # ── NETWORKING ────────────────────────────────────────────────
    # Subnets for the ALB (public subnets for internet-facing)
    alb.ingress.kubernetes.io/subnets: subnet-abc123, subnet-def456
    # OR: tag subnets with kubernetes.io/role/elb=1 and auto-discover

    # Security groups
    alb.ingress.kubernetes.io/security-groups: sg-abcdef123

    # ── WAF ───────────────────────────────────────────────────────
    alb.ingress.kubernetes.io/wafv2-acl-arn: >-
      arn:aws:wafv2:us-east-1:123456789:regional/webacl/my-waf/abc123

    # ── ACCESS LOGS ───────────────────────────────────────────────
    alb.ingress.kubernetes.io/load-balancer-attributes: >-
      access_logs.s3.enabled=true,
      access_logs.s3.bucket=my-alb-logs,
      access_logs.s3.prefix=myapp

spec:
  ingressClassName: alb

  rules:
    - host: myapp.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080

          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

### ALB vs Nginx on EKS — When to Use Which

```
USE NGINX INGRESS CONTROLLER WHEN:
───────────────────────────────────────────────────────────────
✅ Multiple services, ONE load balancer (cost optimization)
✅ Advanced rewrite rules (regex, rewrite-target)
✅ Rate limiting per path/IP
✅ Basic auth on specific paths
✅ WebSocket support
✅ Migrating from on-prem nginx config
✅ CKAD exam environment

1 NLB + N Nginx pods → handles all Ingress objects = cheapest

USE AWS ALB CONTROLLER WHEN:
───────────────────────────────────────────────────────────────
✅ Deep AWS integration (WAF, Cognito, ACM certs, Shield)
✅ Each service needs its own ALB (isolation requirement)
✅ ALB access logs to S3 for compliance
✅ Integration with AWS targets outside EKS (Lambda, EC2)
✅ Weighted routing for canary/blue-green (ALB target groups)
✅ You want AWS to manage the LB lifecycle

1 ALB per Ingress = more $$ but fully managed and AWS-native
```

---

## 13. Ingress Annotations — The Power Layer

Annotations are where the real power lives. They're controller-specific configuration that goes beyond the standard Kubernetes Ingress spec.

### Nginx Annotations Quick Reference

```yaml
annotations:
  # ── ROUTING ───────────────────────────────────────────────────
  nginx.ingress.kubernetes.io/rewrite-target: /         # Strip path prefix
  nginx.ingress.kubernetes.io/use-regex: "true"         # Enable regex in paths
  nginx.ingress.kubernetes.io/app-root: /app            # Redirect / to /app

  # ── TLS ───────────────────────────────────────────────────────
  nginx.ingress.kubernetes.io/ssl-redirect: "true"      # Force HTTPS
  nginx.ingress.kubernetes.io/ssl-passthrough: "true"   # Pass TLS to backend

  # ── PERFORMANCE ───────────────────────────────────────────────
  nginx.ingress.kubernetes.io/proxy-body-size: "10m"    # Max upload size
  nginx.ingress.kubernetes.io/proxy-buffer-size: "128k" # Response buffer
  nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
  nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "60"

  # ── SECURITY ──────────────────────────────────────────────────
  nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,192.168.0.0/16"
  nginx.ingress.kubernetes.io/auth-type: basic
  nginx.ingress.kubernetes.io/auth-secret: basic-auth

  # ── RATE LIMITING ─────────────────────────────────────────────
  nginx.ingress.kubernetes.io/limit-connections: "10"   # Per IP concurrent
  nginx.ingress.kubernetes.io/limit-rps: "5"            # Requests per second

  # ── CANARY DEPLOYMENTS ────────────────────────────────────────
  nginx.ingress.kubernetes.io/canary: "true"
  nginx.ingress.kubernetes.io/canary-weight: "20"       # 20% traffic to canary
```

### ALB Annotations Quick Reference

```yaml
annotations:
  # ── SCHEME & TARGET ───────────────────────────────────────────
  alb.ingress.kubernetes.io/scheme: internet-facing
  alb.ingress.kubernetes.io/target-type: ip

  # ── TLS ───────────────────────────────────────────────────────
  alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
  alb.ingress.kubernetes.io/ssl-redirect: "443"
  alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...

  # ── STICKINESS ────────────────────────────────────────────────
  alb.ingress.kubernetes.io/target-group-attributes: >-
    stickiness.enabled=true,stickiness.lb_cookie.duration_seconds=86400

  # ── TAGS ──────────────────────────────────────────────────────
  alb.ingress.kubernetes.io/tags: Environment=production,Team=backend

  # ── INGRESS GROUP (share one ALB across multiple Ingresses) ───
  alb.ingress.kubernetes.io/group.name: my-alb-group
  alb.ingress.kubernetes.io/group.order: "10"
```

---

## 14. End-to-End: Deploy a Full App with Ingress on EKS

A complete working example — the same flow a Sr. Cloud Engineer walks a dev team through.

### The Stack

```
Internet → ALB (or Nginx) → api.myapp.com/v1 → API (Python FastAPI)
                          → myapp.com/        → Frontend (React)
                          → myapp.com/admin   → Admin panel
```

### The Full Manifest

```yaml
# full-app-ingress.yaml

---
# ── NAMESPACE ────────────────────────────────────────────────────────
apiVersion: v1
kind: Namespace
metadata:
  name: production

---
# ── FRONTEND SERVICE ─────────────────────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: production
spec:
  type: ClusterIP               # ← Internal only — Ingress routes to it
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 3000

---
# ── API SERVICE ──────────────────────────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: api-svc
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8000

---
# ── ADMIN SERVICE ────────────────────────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: admin-svc
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: admin
  ports:
    - port: 80
      targetPort: 4000

---
# ── THE INGRESS (Nginx version) ──────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "20m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.com
        - api.myapp.com
      secretName: myapp-tls-secret
  rules:
    # Rule 1: API subdomain
    - host: api.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80

    # Rule 2: Main domain — admin first (more specific), then catch-all
    - host: myapp.com
      http:
        paths:
          - path: /admin
            pathType: Prefix
            backend:
              service:
                name: admin-svc
                port:
                  number: 80

          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 80
```

```bash
# Deploy and verify:
kubectl apply -f full-app-ingress.yaml

# Watch ingress get an address (takes 1-3 min for cloud LB provisioning):
kubectl get ingress -n production -w
# NAME             CLASS   HOSTS                        ADDRESS        PORTS     AGE
# myapp-ingress    nginx   api.myapp.com,myapp.com      203.0.113.10   80, 443   2m

# Test routing (add to /etc/hosts for local testing):
echo "203.0.113.10 myapp.com api.myapp.com" >> /etc/hosts

curl -I https://myapp.com/           # → frontend-svc
curl -I https://myapp.com/admin/     # → admin-svc
curl -I https://api.myapp.com/users  # → api-svc

# Verify endpoints are populated:
kubectl get endpoints -n production
# NAME            ENDPOINTS
# frontend-svc    10.244.0.5:3000,10.244.0.6:3000    ← pods are selected ✅
# api-svc         10.244.1.5:8000                    ← pod is selected ✅
# admin-svc       10.244.2.5:4000                    ← pod is selected ✅
```

---

## 15. Debugging Guide

### Complete Diagnostic Runbook

```bash
# ════════════════════════════════════════════════════════════════
# STEP 1: Check the Ingress resource itself
# ════════════════════════════════════════════════════════════════

kubectl describe ingress <name> -n <namespace>

# Healthy output looks like:
# Rules:
#   Host          Path  Backends
#   ────          ────  ────────
#   myapp.com     /     frontend-svc:80 (10.244.0.5:3000,10.244.0.6:3000)
#                                        ↑ Pod IPs listed here = WORKING

# Broken output:
# frontend-svc:80 (<none>)   ← No pod IPs = selector mismatch or no pods

# ════════════════════════════════════════════════════════════════
# STEP 2: Verify Service → Pod label matching
# ════════════════════════════════════════════════════════════════

# Get service selector:
kubectl get svc frontend-svc -n production -o jsonpath='{.spec.selector}'
# {"app":"frontend"}

# Get pod labels:
kubectl get pods -n production --show-labels | grep frontend
# frontend-xxx   Running   app=frontend   ← MATCH ✅
# frontend-xxx   Running   app=web        ← MISMATCH ❌

# ════════════════════════════════════════════════════════════════
# STEP 3: Check Endpoints directly
# ════════════════════════════════════════════════════════════════

kubectl get endpoints -n production
# If ENDPOINTS column is <none> → selector mismatch

# ════════════════════════════════════════════════════════════════
# STEP 4: Test service directly (bypass Ingress)
# ════════════════════════════════════════════════════════════════

# Port-forward to the service directly:
kubectl port-forward svc/frontend-svc 8080:80 -n production
# Then: curl http://localhost:8080
# If this works → Service + Pod is fine → Ingress config is the bug
# If this fails → Service or Pod is the bug

# ════════════════════════════════════════════════════════════════
# STEP 5: Check Ingress Controller logs
# ════════════════════════════════════════════════════════════════

# Nginx:
kubectl logs -n ingress-nginx \
  $(kubectl get pods -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx \
    -o jsonpath='{.items[0].metadata.name}') \
  --tail=50

# ALB Controller:
kubectl logs -n kube-system \
  $(kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller \
    -o jsonpath='{.items[0].metadata.name}') \
  --tail=50

# ════════════════════════════════════════════════════════════════
# STEP 6: Check IngressClass assignment
# ════════════════════════════════════════════════════════════════

kubectl get ingress <name> -n <ns> -o jsonpath='{.spec.ingressClassName}'
kubectl get ingressclass
# Verify the class in the Ingress matches an existing IngressClass

# ════════════════════════════════════════════════════════════════
# STEP 7: Events (often shows the root cause directly)
# ════════════════════════════════════════════════════════════════

kubectl get events -n production --sort-by='.lastTimestamp' | tail -20
```

### Error Code Reference

```
HTTP 502 Bad Gateway
  → Controller reached the Service but the Pod returned an error
  → Check: Pod logs, Pod health, application startup errors

HTTP 503 Service Unavailable
  → Controller couldn't reach any Pod
  → Check: Endpoints (kubectl get endpoints), Pod status, selector match

HTTP 404 Not Found (from Nginx)
  → No Ingress rule matched the request
  → Check: path, pathType, host header (curl -H "Host: myapp.com" ...)

HTTP 404 (from your app)
  → Ingress routed correctly, your app returned 404
  → The path reached the right service — app logic issue

connection refused / timeout
  → Controller can't reach the Service
  → Check: Service port matches Ingress port, namespace

Address is empty after 5+ minutes
  → IngressClass mismatch or controller not running
  → Check: kubectl get ingressclass, controller pod status
```

---

## 16. CKAD Exam Cheat Sheet — Last-Night Review

### The 5 Things Always Verified Before Submitting

```
1. apiVersion: networking.k8s.io/v1      ← NOT extensions/v1beta1
2. namespace: same as backend services   ← Ingress + Service must match
3. service.port.number: service port     ← NOT container/pod port
4. ingressClassName: nginx               ← Match what's in the cluster
5. pathType: Prefix or Exact             ← Prefix for most cases
```

### Speed Commands

```bash
# Create ingress fast:
kubectl create ingress NAME \
  --rule="HOST/PATH*=SVC:PORT" \
  --class=nginx \
  -n NAMESPACE

# Debug fast (paste entire block):
kubectl describe ingress NAME -n NS
kubectl get endpoints -n NS
kubectl get pods -n NS --show-labels

# Fix service port:
kubectl edit ingress NAME -n NS

# Fix selector:
kubectl patch svc SVC -n NS -p '{"spec":{"selector":{"app":"CORRECT"}}}'

# Generate YAML to edit:
kubectl create ingress NAME --rule="HOST/*=SVC:PORT" --dry-run=client -o yaml

# Check controller:
kubectl get pods -n ingress-nginx
kubectl get ingressclass
```

### The Troubleshooting Decision Tree

```
Traffic not reaching app via Ingress?
│
├── kubectl get ingress → ADDRESS empty?
│   └── YES → IngressClass wrong or controller not running
│       → kubectl get ingressclass
│       → kubectl get pods -n ingress-nginx
│
├── kubectl describe ingress → Backend shows <none>?
│   └── YES → Service name/port wrong OR Service has no Endpoints
│       → kubectl get endpoints -n NS
│       → kubectl get pods --show-labels → compare with Service selector
│
├── Endpoints populated but still 5xx?
│   └── kubectl port-forward svc/NAME 8080:PORT -n NS
│       → curl localhost:8080
│       → If 200: Ingress routing rule is wrong (path, host)
│       → If error: Pod or app is broken
│
└── Getting 404 from Nginx (not app)?
    └── pathType wrong or path doesn't match request URL
        → Change Exact → Prefix
        → Check host header matches exactly
```

### YAML Template — Copy-Paste in Exam

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: INGRESS_NAME
  namespace: NAMESPACE
spec:
  ingressClassName: nginx
  rules:
    - host: HOST
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: SERVICE_NAME
                port:
                  number: SERVICE_PORT
```

---

## 📁 Where This Fits in the Repo

```
devops-field-guide/
├── python/
├── eks/
│   └── ebs-volumes-on-eks-the-complete-guide-for-devops-engineers.md
├── ecs/
│   └── what-devops-engineers-should-understand-about-ecs.md
├── docker/
│   └── docker-compose-local-to-ecs-the-complete-devops-playbook.md
├── ckad/
│   └── kubernetes-ingress-ckad-exam-and-eks-production-mastery.md  ← THIS FILE
├── aws/
│   ├── what-happens-when-you-type-www-google-com.md
│   └── from-transistors-to-tensor-cores-cpu-gpu-for-devops.md
├── README.md
└── LICENSE
```

> **Best home:** `ckad/` — it is exam-focused with production-grade EKS depth. Cross-links naturally with the `eks/` and `aws/` folders.

---

*Part of the [devops-field-guide](../) series · Built for engineers who want to pass the exam AND ship to production.*

⭐ Star the repo if this helped you ace the CKAD Ingress questions.
