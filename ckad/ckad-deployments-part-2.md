# CKAD Deployments Mastery — Part 2: Advanced Patterns & The 10 Rules


---

*If you haven't read [Part 1](./ckad-deployments-part-1.md) (The Blueprint & Core Patterns), start there.*

---

## From Part 1 — what you should already own

- Full annotated Deployment YAML skeleton
- All 9 `spec` fields and their exam relevance
- Imperative generation with `--dry-run=client`
- `kubectl set image` + `rollout status`
- `kubectl rollout undo` + `--to-revision`
- The ReplicaSet handoff during rolling updates

---


### `[Task 03]` Add readiness and liveness probes to an existing Deployment · *Medium*

```
The deployment 'order-service' in namespace 'payments' needs health checks.
Add a readinessProbe that checks GET /ready on port 3000 every 5 seconds
with an initial delay of 10 seconds.
Add a livenessProbe that checks GET /healthz on port 3000 every 15 seconds
with an initial delay of 30 seconds.
Ensure the rollout completes successfully.
```

```yaml
# order-service.yaml — probe section to add under containers[0]
    spec:
      containers:
      - name: order-service
        image: myrepo/orders:v3.1

        readinessProbe:               # Pod added to Service endpoints ONLY when this passes
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 10     # Wait 10s after container starts before first check
          periodSeconds:       5      # Check every 5 seconds
          failureThreshold:    3      # 3 consecutive failures = pod removed from endpoints
          successThreshold:    1      # 1 pass puts it back — can be >1 for readiness

        livenessProbe:                # Pod RESTARTED when this fails
          httpGet:
            path: /healthz
            port: 3000
          initialDelaySeconds: 30     # Must be >= readiness initialDelay
          periodSeconds:       15
          failureThreshold:    3
          successThreshold:    1      # MUST be exactly 1 for liveness — API rejects any other value
```

```bash
# Export the live deployment YAML, edit it, reapply
kubectl get deployment order-service -n payments -o yaml > order-service.yaml
vim order-service.yaml    # add probe blocks under containers[0]
kubectl apply -f order-service.yaml

# Watch rollout — new pods will be tested against both probes
kubectl rollout status deployment/order-service -n payments

# Verify probes are registered
kubectl describe pod -n payments -l app=order-service | grep -A8 "Readiness\|Liveness"
```

The critical distinction the exam tests: **readiness** controls traffic routing — removes the pod from Service endpoints when failing but the pod keeps running. **Liveness** controls pod lifecycle — restarts the container when failing. These are completely independent. A pod can be not-ready-but-alive, or passing liveness while failing readiness.

### Probe types — behaviour and exam signals

| Probe type | On failure | Exam signal phrase | `successThreshold` |
|---|---|---|---|
| `readinessProbe` | Removed from Service endpoints. Pod keeps running. | "ensure no traffic until ready" | Any value ≥ 1 |
| `livenessProbe` | Container killed and restarted. | "restart container if unresponsive" | Must be exactly **1** |
| `startupProbe` | Container killed if startup doesn't pass within `failureThreshold × period`. | "app takes 2 minutes to initialize" | Any value ≥ 1 |

> 💡 **All three probe mechanisms the exam tests**
>
> - `httpGet` — HTTP GET request; 200–399 = success. Most common.
> - `tcpSocket` — checks if a TCP port accepts connections. Good for databases and non-HTTP services.
> - `exec` — runs a command inside the container; exit code 0 = success, anything else = failure.

```yaml
# All three probe mechanisms — know the syntax for each

readinessProbe:               # httpGet — most common
  httpGet:
    path: /ready
    port: 8080

livenessProbe:                # tcpSocket — for non-HTTP ports
  tcpSocket:
    port: 5432
  initialDelaySeconds: 15

startupProbe:                 # exec — run a command, exit 0 = healthy
  exec:
    command:
    - sh
    - -c
    - test -f /tmp/ready       # exits 0 if file exists
  failureThreshold: 30         # 30 × periodSeconds(10) = 300s startup window
  periodSeconds:    10
```

---


### `[Task 04]` Implement a Canary deployment alongside the stable version · *Hard*

```
You have a stable deployment 'frontend' with 4 replicas running 'webapp:v1'.
A new version 'webapp:v2' needs to be canary-tested with roughly 20% of traffic.
A Service named 'frontend-svc' uses selector 'app: frontend' to route traffic.
Implement the canary without modifying the stable deployment or the Service.
```

The CKAD canary pattern exploits Kubernetes' label-based traffic routing. Both Deployments share the same label the Service selects on — so traffic distributes proportionally by pod count. No Istio, no ingress controllers, no service mesh required.

```yaml
# canary-deployment.yaml — create this
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-canary            # Different name — entirely separate Deployment object
spec:
  replicas: 1                      # 1 canary + 4 stable = 5 total → 1/5 = 20% of traffic
  selector:
    matchLabels:
      app: frontend                # SAME label — Service will route to this pod
      version: canary              # Extra label — prevents stable Deployment adopting this pod
  template:
    metadata:
      labels:
        app: frontend              # Service selector matches — pod receives traffic
        version: canary            # Canary selector matches — this Deployment owns this pod
    spec:
      containers:
      - name: frontend
        image: webapp:v2           # The new version under test
        resources:
          requests:
            cpu:    "100m"
            memory: "128Mi"
          limits:
            cpu:    "200m"
            memory: "256Mi"
```

```bash
kubectl apply -f canary-deployment.yaml

# Verify: 4 stable pods + 1 canary pod, all behind the same Service
kubectl get pods -l app=frontend --show-labels
# frontend-xxx-1 ... version=stable
# frontend-xxx-2 ... version=stable
# frontend-xxx-3 ... version=stable
# frontend-xxx-4 ... version=stable
# frontend-canary-xxx ... version=canary  ← v2

# Confirm Service is routing to all 5 pods
kubectl get endpoints frontend-svc
# Should show 5 IP addresses

# ── If canary is healthy: promote ──
kubectl set image deployment/frontend frontend=webapp:v2
kubectl delete deployment frontend-canary

# ── If canary is bad: abort ──
kubectl delete deployment frontend-canary
# Traffic immediately returns 100% to stable v1
```

> 💡 **Why the extra `version` label is non-negotiable**
>
> Without `version: canary` in the canary Deployment's `selector.matchLabels`, the stable Deployment's selector (`app: frontend`) would match the canary pods and try to own them — creating a controller conflict. Both Deployments would fight over the same pods. By making the canary selector more specific, each Deployment owns only its pods. The Service doesn't care about the version label — it only needs `app: frontend` — so it routes to all five pods regardless.

### Traffic math — sizing your canary

Kubernetes distributes traffic uniformly across all pods a Service selects. There's no weighted routing in vanilla Kubernetes — it's pure pod count ratio.

**Formula:** `canary_pods = (stable_pods × target%) / (100 - target%)`

| Target canary % | Stable replicas | Canary replicas | Actual % |
|---|---|---|---|
| ~10% | 9 | 1 | 10.0% |
| ~20% | 4 | 1 | 20.0% |
| ~25% | 3 | 1 | 25.0% |
| ~33% | 2 | 1 | 33.3% |
| ~50% | 1 | 1 | 50.0% |

---


### `[Task 05]` Diagnose and fix a Deployment that fails to create pods · *Hard*

```
A deployment 'data-processor' in namespace 'analytics' was applied but no pods are running.
The deployment shows 0/3 ready. Fix all issues in the manifest and ensure
all 3 pods reach Running state.
```

The broken manifest below contains 6 deliberate bugs. Try to find them all before reading the comments.

```yaml
# broken-deployment.yaml — 6 bugs hidden inside
apiVersion: v1                         # BUG 1: Should be apps/v1
kind: Deployment
metadata:
  name: data-processor
  namespace: analytics
spec:
  replicas: 3
  selector:
    matchLabels:
      app: processor                   # BUG 2: 'processor' ≠ 'data-processor' in template below
  template:
    metadata:
      labels:
        app: data-processor            # BUG 2: selector/template label mismatch
    spec:
      containers:
      - name: processor
        image: python:3.11
        command: ["python", "app.py"]
        resources:
          requests:
            memory: 512GB              # BUG 3: 'GB' is not a valid K8s unit — use Mi or Gi
          limits:
            memory: 1GB               # BUG 3: Same — should be '1Gi'
        readinessProbe:
          httpGet:
            path: /status
            port: 8080
          initialDelaySeconds: 5
          successThreshold: 3          # Valid for readinessProbe (>1 is allowed)
        livenessProbe:
          exec:
            command:
            - python
            - healthcheck.py
          successThreshold: 2          # BUG 4: livenessProbe successThreshold MUST be 1
        env:
        - name:  SECRET_KEY
          valueFrom:
            secretKeyRef:
              name:     app-secrets
              key:      api-key
              optional: false          # BUG 5: Pod won't start if Secret 'app-secrets' is missing
      volumes:
      - name: data-vol
        emptyDir: {}
      - name: config-vol
        configMap:
          name: missing-config         # BUG 6: ConfigMap 'missing-config' likely doesn't exist
```

### The systematic debugging sequence — always in this exact order

```bash
# Step 1: Is the Deployment itself valid?
kubectl describe deployment data-processor -n analytics
# Look at: Conditions, Events, selector mismatch errors, resource parsing

# Step 2: Did any pods get created?
kubectl get pods -n analytics -l app=data-processor
# No pods = selector/template issue, apiVersion wrong, or namespace missing
# Pods exist but not running = check pod status next

# Step 3: What is the pod status and events?
kubectl describe pod <pod-name> -n analytics
# Events section (bottom) shows:
# - Failed pulling image          → wrong image name/tag or missing pull secret
# - OOMKilled                     → memory limit too low
# - CrashLoopBackOff              → app crashing — check logs
# - CreateContainerConfigError    → Secret or ConfigMap missing in namespace
# - Unschedulable                 → resource requests exceed any node capacity

# Step 4: App logs (only if container is starting but dying)
kubectl logs <pod-name> -n analytics
kubectl logs <pod-name> -n analytics --previous    # crashed container logs

# Step 5: Namespace-level events — catches things describe misses
kubectl get events -n analytics \
  --sort-by=.metadata.creationTimestamp | tail -20
```

### Symptom → cause → fix reference

| Symptom | Root cause | Fix |
|---|---|---|
| 0 pods, no ReplicaSet | `apiVersion` wrong, selector mismatch, or namespace missing | Check `kubectl describe deployment` for admission error |
| Pods Pending forever | Resource requests exceed any node, or nodeSelector/taint mismatch | `kubectl describe pod` → Events → "Unschedulable" |
| ImagePullBackOff | Wrong image name/tag, or private registry without pull secret | Fix image tag; add `imagePullSecrets` if needed |
| CreateContainerConfigError | Referenced Secret or ConfigMap doesn't exist in namespace | Create the missing resource, or set `optional: true` |
| CrashLoopBackOff | Container crashes on startup — wrong command, app error, probe too aggressive | `kubectl logs --previous` |
| OOMKilled | Memory limit too low for the workload | Increase `resources.limits.memory` |
| Pods Running, 0/N ready | Readiness probe failing — wrong path/port, or initialDelaySeconds too short | Fix probe config or increase `initialDelaySeconds` |

> ⚠️ **The silent failure — `optional: false` on a missing Secret**
>
> When `secretKeyRef.optional` is `false` (the default) and the Secret doesn't exist, the pod fails at `CreateContainerConfigError` with no clear error in the deployment events. You only see it in `kubectl describe pod`. If a task gives you a deployment that can't start and the YAML references a Secret, always check: `kubectl get secret app-secrets -n analytics`.

---


### `[Task 06]` Configure zero-downtime rolling update constraints · *Medium*

```
Update deployment 'checkout' in namespace 'ecommerce' to ensure:
zero pods are unavailable during a rollout, at most 2 extra pods can run simultaneously
during rollout, and each new pod must remain healthy for at least 20 seconds
before the next replacement begins. Then scale the deployment to 6 replicas.
```

```yaml
# checkout — strategy section to update
spec:
  replicas: 6                        # "scale to 6"

  minReadySeconds: 20                # "healthy for 20 seconds before next replacement"

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0              # "zero pods unavailable" = 0
      maxSurge:       2              # "at most 2 extra pods" = 2 absolute
```

```bash
# Export, edit, apply
kubectl get deployment checkout -n ecommerce -o yaml > checkout.yaml
vim checkout.yaml    # update strategy + minReadySeconds + replicas
kubectl apply -f checkout.yaml

# Or patch the strategy fields and scale separately
kubectl patch deployment checkout -n ecommerce \
  --patch '{"spec":{"minReadySeconds":20,"strategy":{"rollingUpdate":{"maxUnavailable":0,"maxSurge":2}}}}'
kubectl scale deployment checkout --replicas=6 -n ecommerce

# Verify all three settings
kubectl get deployment checkout -n ecommerce \
  -o jsonpath='{.spec.strategy}{"\n"}{.spec.minReadySeconds}{"\n"}{.spec.replicas}'
```

---


### `[Task 07]` Add an init container that pre-populates a shared volume · *Hard*

```
Create a deployment 'report-gen' in namespace 'finance' with 2 replicas.
Before the main container starts, an init container must copy config files
from a ConfigMap into a shared volume at /etc/app-config.
The main container (image: python:3.11-slim) should read from /etc/app-config.
```

```yaml
# report-gen.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: report-gen
  namespace: finance
spec:
  replicas: 2
  selector:
    matchLabels:
      app: report-gen
  template:
    metadata:
      labels:
        app: report-gen
    spec:

      initContainers:                # Runs sequentially, BEFORE any container in 'containers'
      - name: config-loader
        image: busybox:1.35
        command:
        - sh
        - -c
        - cp /tmp/source/* /etc/app-config/   # Copy from CM mount → shared emptyDir
        volumeMounts:
        - name:      config-source            # ConfigMap mounted read-only here
          mountPath: /tmp/source
          readOnly:  true
        - name:      shared-config            # emptyDir — init writes, main reads
          mountPath: /etc/app-config

      containers:
      - name: report-generator
        image: python:3.11-slim
        command: ["python", "/app/generate.py"]
        volumeMounts:
        - name:      shared-config            # Reads what init container wrote
          mountPath: /etc/app-config
          readOnly:  true
        resources:
          requests:
            cpu:    "100m"
            memory: "128Mi"
          limits:
            cpu:    "500m"
            memory: "256Mi"

      volumes:
      - name: shared-config
        emptyDir: {}              # Fresh per pod. Shared between init and main in same pod.
      - name: config-source
        configMap:
          name: app-config       # Must exist before applying this Deployment
```

> 💡 **Init container lifecycle — the 3 rules**
>
> 1. Init containers run **sequentially** — not in parallel. The second doesn't start until the first exits 0.
> 2. If an init container fails, Kubernetes restarts it per the pod's `restartPolicy`.
> 3. Main containers **never start** until ALL init containers have exited with code 0.
>
> This makes them perfect for dependency gates: "don't start the app until the database is accepting connections."

---

## The imperative cheat sheet — exam speed commands

In a 2-hour exam with 15–20 tasks, you cannot write YAML from scratch. These are the 16 commands that save you 3–5 minutes per task.

| Command | What it does |
|---|---|
| `kubectl create deploy NAME --image=IMG --replicas=N --dry-run=client -o yaml > f.yaml` | Generate YAML skeleton |
| `kubectl set image deploy/NAME container=IMG:TAG -n NS` | Update image |
| `kubectl scale deploy NAME --replicas=N -n NS` | Scale replicas |
| `kubectl rollout status deploy/NAME -n NS` | Watch rollout (blocks until complete) |
| `kubectl rollout history deploy/NAME -n NS` | Show revision history |
| `kubectl rollout undo deploy/NAME -n NS` | Rollback one step |
| `kubectl rollout undo deploy/NAME --to-revision=N -n NS` | Rollback to specific revision |
| `kubectl rollout pause deploy/NAME -n NS` | Pause a rollout |
| `kubectl rollout resume deploy/NAME -n NS` | Resume a paused rollout |
| `kubectl rollout restart deploy/NAME -n NS` | Restart all pods |
| `kubectl edit deploy NAME -n NS` | Edit live resource in-place |
| `kubectl patch deploy NAME --patch '{"spec":{...}}' -n NS` | Patch inline without editor |
| `kubectl get pods -l app=NAME --show-labels -n NS` | Get pods by label |
| `kubectl get endpoints SVC-NAME -n NS` | Check Service routing targets |
| `kubectl get events -n NS --sort-by=.metadata.creationTimestamp` | Debug events in order |
| `kubectl logs POD -c CONTAINER --previous -n NS` | Logs from crashed container |

---

## The 10 rules that separate passing from failing

These are not tips. They are the pattern-matching rules your brain needs to run automatically when reading an exam prompt.

### Rule 1 — Always generate, never write from scratch

Use `kubectl create deployment NAME --image=IMG --dry-run=client -o yaml > f.yaml` as your starting point. The generated YAML has correct structure and valid field names. You're just adding to it — not typing it.

### Rule 2 — Set your namespace on every command

Use `-n NAMESPACE` on every kubectl command, or set a default at the start of each task:

```bash
kubectl config set-context --current --namespace=NAMESPACE
```

Forgetting the namespace and operating in `default` is a silent failure — you complete the task in the wrong place and get zero marks.

### Rule 3 — Trap: `apiVersion: apps/v1`, not `v1`

Deployments, StatefulSets, DaemonSets, and ReplicaSets are all `apps/v1`. Only Pods, Services, ConfigMaps, Secrets, PersistentVolumeClaims, and Namespaces are core `v1`. Getting this wrong means the API server rejects the manifest immediately.

### Rule 4 — Trap: selectors are immutable

If a task asks you to change `spec.selector`, you must delete and recreate the Deployment. You cannot update the selector in place — kubectl will reject it. Use `--cascade=orphan` if you want to keep existing pods during the transition:

```bash
kubectl delete deploy NAME --cascade=orphan
```

### Rule 5 — `kubectl rollout status` is your verification command

It blocks until complete and prints either "successfully rolled out" or times out with an error. Use it as the built-in verification step every time you update a Deployment.

### Rule 6 — Trap: `livenessProbe.successThreshold` must be exactly 1

The API server validates this and will reject any liveness probe with `successThreshold` other than 1. For readiness probes, it can be any value ≥ 1. This asymmetry is one of the most common broken-YAML bugs in the exam.

### Rule 7 — `kubectl describe pod` first, not logs

Use `kubectl describe pod` first — the Events section shows exactly what Kubernetes is doing: image pulls, probe results, volume mount failures, scheduling decisions. Logs only help once the container is running. Describe tells you why it's not running.

### Rule 8 — Trap: memory units are `Mi` and `Gi`, never `GB`

- `Mi` = mebibytes (powers of 2) — use this
- `Gi` = gibibytes — use this
- `M` and `G` = powers of 10 — valid but avoid
- `GB` — **not a valid Kubernetes unit** — causes a validation error

Default to `Mi` and `Gi` for everything.

### Rule 9 — Canary requires a more specific selector on the canary Deployment

Add an extra label like `version: canary` to both the canary Deployment's `selector.matchLabels` AND the pod template labels. Without it, the stable Deployment adopts the canary pods. The Service only needs the shared label (`app: frontend`) — it routes to all pods regardless of the extra label.

### Rule 10 — Map exam phrases directly to YAML fields

| Exam phrase | YAML field | Value |
|---|---|---|
| "zero pods unavailable" | `maxUnavailable` | `0` |
| "at most N extra pods" | `maxSurge` | `N` |
| "healthy for N seconds before next" | `minReadySeconds` | `N` |
| "kill all then restart" | `strategy.type` | `Recreate` |
| "app takes 2 minutes to start" | `startupProbe` with `failureThreshold × periodSeconds` | `≥ 120` |

Train your brain to hear the phrase and see the YAML field instantly. That's the reflex that saves you two minutes per task.

---

## The bottom line

The Deployment is not just a resource type — it's the lens through which the CKAD tests whether you actually understand Kubernetes.

Own the YAML at the level where you understand what every field does and why. Not because you memorised it, but because you've run every pattern with your own hands and felt the difference between `maxUnavailable: 0` and `maxUnavailable: 1` under real traffic.

The exam is open-book in one sense: `kubectl explain deployment.spec.strategy.rollingUpdate` is always available and reflects the actual cluster version. Use it freely — it's faster than guessing and more reliable than memory when you're 90 minutes deep into a timed exam.

The difference between 72% and 91% is not knowing more YAML. It's moving through each task with a clear mental sequence: read the signal phrase, know the field, generate the skeleton, add what's missing, apply, verify, move on.

That's the whole game.

---

*All YAML and commands validated against Kubernetes 1.31. The CKAD exam currently runs on Kubernetes 1.31 as of 2025. Verify the current exam version at [training.linuxfoundation.org](https://training.linuxfoundation.org) before your exam date. This is Part 2 of a 2-part series — if you haven't read Part 1 (The Blueprint & Core Patterns), start there.*
