# CKAD Deployments Mastery — Part 1: The Blueprint & Core Patterns


---

*Part 2 (Advanced Patterns, Cheat Sheet & The 10 Rules) publishes tomorrow.*

---

## What's in this series

1. Why Deployments dominate the CKAD
2. The Deployment YAML — fully annotated skeleton
3. Field anatomy — what every key actually does
4. Exam Pattern 1 — Create a Deployment from scratch
5. Exam Pattern 2 — Rolling Update and Rollback

---

## Why Deployments dominate the CKAD

The CKAD exam is 2 hours, 15–20 performance-based tasks, and roughly 66% of the curriculum touches Deployments directly or indirectly. Every question about rolling updates, scaling, readiness probes, resource limits, init containers, labels, selectors, and environment injection — it all flows through the `Deployment` resource.

The exam doesn't test whether you've memorised YAML. It tests whether you can *operate* Kubernetes under time pressure. That means reading a broken manifest and spotting what's wrong in 90 seconds. That means knowing that `kubectl rollout undo` saves you when a question asks you to rollback without explaining how. That means understanding *why* a pod isn't starting — not just that it isn't.

This two-part series is structured exactly like the exam: real prompts, real YAML, real commands, and the deep insight that separates a 72% pass from a 91% pass.

> **How to use this series:** Don't just read it. As you work through each pattern, open a terminal with a local cluster (kind or minikube) and run every command. The CKAD is a hands-on performance exam — muscle memory beats reading every time.

---

## The Deployment YAML — fully annotated skeleton

Before any exam pattern makes sense, you need to own this file. Every field, every default, every reason it exists. Read this once slowly. You'll be generating variations of it under time pressure for the rest of the exam.

```yaml
apiVersion: apps/v1                    # Never forget apps/ — 'v1' alone means core API group (Pods, Services)
kind:       Deployment
metadata:
  name:      api-server
  namespace: production               # Always set — omitting uses 'default', wrong in most exam tasks
  labels:
    app: api-server
    tier: backend

spec:
  replicas: 3                         # How many Pods to maintain simultaneously

  selector:                           # CRITICAL: Must match template.metadata.labels EXACTLY
    matchLabels:
      app: api-server

  strategy:
    type: RollingUpdate               # Default. Alternative: Recreate
    rollingUpdate:
      maxSurge:       1               # Max pods ABOVE desired count during update
      maxUnavailable: 0               # 0 = zero-downtime rollout

  minReadySeconds: 10                 # Pod must be Ready for 10s before counted as available
  revisionHistoryLimit: 5             # Keep last 5 ReplicaSets for rollback. Default is 10.
  progressDeadlineSeconds: 600        # Deployment fails if not progressed in 10 min

  template:                           # This IS a Pod spec — everything under here is pod config
    metadata:
      labels:
        app: api-server               # MUST match spec.selector.matchLabels exactly
      annotations:
        prometheus.io/scrape: "true"

    spec:
      terminationGracePeriodSeconds: 30  # Time to finish requests before SIGKILL

      initContainers:                 # Run to completion BEFORE any container starts
      - name: wait-for-db
        image: busybox:1.35
        command:
        - sh
        - -c
        - until nc -z postgres-svc 5432; do echo waiting; sleep 2; done

      containers:
      - name:  api
        image: myrepo/api:v1.4.2      # Always pin a tag. 'latest' is an anti-pattern.
        ports:
        - containerPort: 8080

        env:
        - name:  NODE_ENV
          value: "production"
        - name:  DB_HOST               # From ConfigMap key
          valueFrom:
            configMapKeyRef:
              name: app-config
              key:  database.host
        - name:  DB_PASSWORD           # From Secret key — never hardcode
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key:  password

        resources:                     # ALWAYS set in exam — missing resources = failed tasks
          requests:
            cpu:    "250m"             # 250 millicores = 0.25 CPU
            memory: "128Mi"
          limits:
            cpu:    "500m"
            memory: "256Mi"

        readinessProbe:                # Removes pod from Service endpoints if failing
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds:       5
          failureThreshold:    3

        livenessProbe:                 # Restarts pod if failing
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 30
          periodSeconds:       10
          failureThreshold:    3

        volumeMounts:
        - name:      config-vol
          mountPath: /etc/config
          readOnly:  true

      volumes:
      - name: config-vol
        configMap:
          name: app-config
```

> ⚠️ **Exam trap — the selector/label mismatch**
>
> The single most common broken YAML on the CKAD is a mismatch between `spec.selector.matchLabels` and `spec.template.metadata.labels`. Kubernetes will reject the Deployment at admission with *"selector does not match template labels"*. These two blocks must be byte-for-byte identical on every key you use in the selector.

---

## Field anatomy — what every key actually does

| Field | Required | What it controls + exam relevance |
|---|---|---|
| `apiVersion` | **required** | `apps/v1` — never just `v1`. Deployments live in the `apps` API group. Writing `v1` is an instant failure — the API server rejects it with "no kind Deployment is registered". |
| `spec.replicas` | default: 1 | Desired pod count. Omitting it means 1. "Scale to 5" → use `kubectl scale` or edit the YAML. Both valid. |
| `spec.selector` | **required** | Immutable after creation. The Deployment uses this to find and own Pods. To change it — delete and recreate. |
| `strategy.type` | default: RollingUpdate | `RollingUpdate` replaces pods gradually. `Recreate` kills all pods then restarts — causes downtime but guarantees no mixed versions. Use Recreate for database schema change deploys. |
| `maxSurge` | default: 25% | How many extra pods can exist above `replicas` during rollout. `1` = one extra pod. Set to `0` if resource-constrained. |
| `maxUnavailable` | default: 25% | How many pods can be unavailable during rollout. Set to `0` for zero-downtime deployments (requires maxSurge ≥ 1). "Ensure no downtime" → set this to 0. |
| `minReadySeconds` | default: 0 | Pod must be continuously Ready for this many seconds before being counted as "available." Prevents a pod that crashes immediately from being counted as a successful rollout step. |
| `revisionHistoryLimit` | default: 10 | Number of old ReplicaSets to keep for rollback. If rollback fails on the exam, check this wasn't set to 0. |
| `progressDeadlineSeconds` | default: 600 | Deployment marked failed with `ProgressDeadlineExceeded` if no progress for this many seconds. Check `kubectl describe deployment` for this condition. |

---


### `[Task 01]` Create a Deployment with specific resource constraints · *Easy*

```
Create a Deployment named 'web-app' in namespace 'staging' using image nginx:1.25.
Set 3 replicas. Container must request 100m CPU and 128Mi memory,
with limits of 200m CPU and 256Mi memory.
Add a label 'environment: staging' to the pod template.
```

### The imperative-first approach (saves 3–4 minutes)

Never write YAML from scratch in the exam. Generate it imperatively, redirect to a file, edit what you can't set with flags, then apply. This is the single biggest time-saving technique in the CKAD.

```bash
# Step 1: Generate the base YAML — do NOT apply yet (--dry-run=client)
kubectl create deployment web-app \
  --image=nginx:1.25 \
  --replicas=3 \
  --namespace=staging \
  --dry-run=client -o yaml > web-app.yaml

# Step 2: Open in vim, add resources and the extra label
vim web-app.yaml

# Step 3: Apply the completed manifest
kubectl apply -f web-app.yaml

# Step 4: Verify — always verify before moving to the next task
kubectl get deployment web-app -n staging
kubectl describe deployment web-app -n staging | grep -A4 Resources
```

```yaml
# web-app.yaml — after editing
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: staging
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
        environment: staging          # added manually
    spec:
      containers:
      - name: web-app
        image: nginx:1.25
        resources:                    # added manually — not generated by kubectl create deployment
          requests:
            cpu:    "100m"
            memory: "128Mi"
          limits:
            cpu:    "200m"
            memory: "256Mi"
```

> 💡 **First thing you do in every exam session**
>
> Run `:set expandtab tabstop=2 shiftwidth=2` in vim before touching any YAML. YAML indentation errors are silent until apply time — the API server rejects the manifest with a cryptic error. Two spaces, no tabs, always.

> 💡 **What `kubectl create deployment` generates vs what you must add manually**
>
> Generated automatically: `apiVersion`, `kind`, `metadata`, `spec.selector`, `spec.template`, basic container with image.
>
> NOT generated — you add these manually: `resources`, `readinessProbe`, `livenessProbe`, `env`, `volumeMounts`, `volumes`, extra labels beyond `app: NAME`.
>
> Know this list. Those are the fields you're always editing after generation.

---


### `[Task 02]` Perform rolling update then roll back to previous version · *Medium*

```
The deployment 'backend' in namespace 'prod' currently runs image 'myapp:v1'.
Update it to image 'myapp:v2'. Verify the rollout completes. Then the team reports
v2 is broken. Roll back to the previous version and confirm all pods are running the old image.
```

```bash
# Always check current state before touching anything
kubectl get deployment backend -n prod
kubectl rollout history deployment/backend -n prod

# Update the image — kubectl set image is the fastest command for this
kubectl set image deployment/backend \
  backend=myapp:v2 \
  -n prod

# Watch the rollout in real-time — this command BLOCKS until complete
kubectl rollout status deployment/backend -n prod
# Output: Waiting for deployment "backend" rollout to finish: 1 out of 3 new replicas...
# Output: deployment "backend" successfully rolled out

# Verify — all pods should show the new image
kubectl get pods -n prod -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'

# ─── v2 is broken. Roll back immediately. ───

# Option A: Roll back one step (to whatever ran before)
kubectl rollout undo deployment/backend -n prod

# Option B: Roll back to a specific revision number
kubectl rollout history deployment/backend -n prod   # find the revision
kubectl rollout undo deployment/backend --to-revision=1 -n prod

# Confirm rollback completed
kubectl rollout status deployment/backend -n prod
kubectl get pods -n prod -o wide
```

> ⚠️ **Critical trap — rollout undo fires instantly**
>
> `kubectl rollout undo` does NOT give you a confirmation prompt. It fires immediately. Always check the revision history first so you know what "one step back" actually means. If `revisionHistoryLimit` was set to `0`, you have nothing to roll back to — this is an exam gotcha.

### What Kubernetes is actually doing during a rolling update

When you trigger a rollout, the Deployment controller creates a new ReplicaSet for the new version and slowly scales it up while scaling down the old ReplicaSet — constrained by `maxSurge` and `maxUnavailable`. Each old ReplicaSet is preserved (up to `revisionHistoryLimit`), so rollback is just adjusting replica counts between existing ReplicaSets. No new container images are pulled during rollback — it reuses what's already cached on the nodes.

```bash
# Watch the ReplicaSet handoff live
kubectl get replicasets -n prod --watch

# During rollout:
# NAME                    DESIRED  CURRENT  READY
# backend-7d9f8c5b4         3        3        3    ← old, scaling down
# backend-6b8a2d1c9         0        0        0    ← new, scaling up
#
# ... midway ...
# backend-7d9f8c5b4         1        1        1
# backend-6b8a2d1c9         2        2        2
#
# ... complete ...
# backend-7d9f8c5b4         0        0        0    ← kept for rollback
# backend-6b8a2d1c9         3        3        3    ← fully live
```

### Exam phrase → YAML field mapping

| Exam phrase | Field to set | Value |
|---|---|---|
| "ensure no downtime" | `maxUnavailable` | `0` |
| "at most N extra pods" | `maxSurge` | `N` |
| "healthy for N seconds before next replacement" | `minReadySeconds` | `N` |
| "kill all pods then restart" | `strategy.type` | `Recreate` |
| "keep N rollback points" | `revisionHistoryLimit` | `N` |

> 💡 **Annotating rollout history**
>
> The CHANGE-CAUSE column in `kubectl rollout history` is populated by the `kubernetes.io/change-cause` annotation. Set it manually:
> ```bash
> kubectl annotate deployment/backend \
>   kubernetes.io/change-cause="updated to v2 for performance fix" -n prod
> ```
> This makes `--to-revision=N` meaningful — you can read the history and pick the right revision with confidence.

---


---

*These are not actual exam questions but what concepts they test in this exam. All YAML and commands validated against Kubernetes 1.31. The CKAD exam currently runs on Kubernetes 1.31 as of 2025. Verify the current exam version at [training.linuxfoundation.org](https://training.linuxfoundation.org) before your exam date. `kubectl explain` is available during the exam and reflects the actual cluster version — use it freely.*
