# Docker Fundamentals for the Advanced DevOps Engineer
## Part 2 of 3 — OverlayFS, the OCI Standard & BuildKit

> *Understanding Docker's storage engine and build architecture changes how you design images.*

**Series:** Docker Internals · Part 2 of 3  
**Tags:** `Docker` `OverlayFS` `OCI` `BuildKit` `Multi-stage` `Image Security`  

---

**What's in this series**

| Part | Topic | Focus |
|---|---|---|
| Part 1 — Previous | Execution Stack, Namespaces, cgroups | How Docker works at the kernel level |
| **Part 2 — Today** | OverlayFS, OCI Standard, BuildKit | Storage, open standards, build engine |
| Part 3 — Next | Networking, Security, Signals, Production Debugging | Running and operating Docker safely |

---

**From Part 1 — foundations to carry forward**

- Docker is a 5-layer stack: CLI → dockerd → containerd → containerd-shim → runc
- `runc` is ephemeral — it exits after starting your container; the shim keeps it alive
- Namespaces isolate what a container sees; cgroups limit what it consumes
- Both are Linux kernel primitives — Docker is a user-space wrapper around them

---

## Section 4 — OverlayFS: The Layer Mechanism Exposed

Every Docker image is a stack of read-only layers. Every running container adds one writable layer on top. The Linux kernel presents all of these as a single unified filesystem using OverlayFS. This mechanism is behind image sharing, fast container startup, and several non-obvious production gotchas.

```
┌──────────────────────────────────────────────────────────────────────┐
│                    WHAT THE CONTAINER SEES                            │
│              MergedDir: / (unified filesystem view)                   │
└──────────────────────────────────────────────────────────────────────┘
                              ▲
                ┌─────────────┴──────────────────┐
                │                                │
┌───────────────┴──────────┐  ┌──────────────────┴─────────────────┐
│    UpperDir (Read/Write) │  │    LowerDir (Read-Only chain)       │
│                          │  │                                     │
│  /var/lib/docker/        │  │  Layer N:   App code                │
│  overlay2/<id>/upper/    │  │  Layer N-1: pip install             │
│                          │  │  Layer N-2: apt-get install         │
│  Container writes go     │  │  Layer N-3: base OS files           │
│  here. Deleted files     │  │                                     │
│  become "whiteout"       │  │  Shared across ALL containers       │
│  markers.                │  │  using the same base image.         │
│  Wiped on docker rm.     │  │  Read from disk once, then cached.  │
└──────────────────────────┘  └─────────────────────────────────────┘
                │
                └── WorkDir: temporary scratch space
                    for OverlayFS atomic operations
```

The key efficiency insight: 10 containers based on the same `python:3.11-slim` image share exactly the same read-only lower layers on disk. Only the per-container UpperDir differs. This is why containers start in milliseconds — there's nothing to copy.

### Inspecting the actual OverlayFS mount on a running container

```bash
# Start a container
docker run -d --name overlay-demo nginx:alpine
CONTAINER_ID=$(docker inspect --format '{{.Id}}' overlay-demo)

# See the actual LowerDir, UpperDir, MergedDir, WorkDir
docker inspect overlay-demo \
  --format '{{json .GraphDriver.Data}}' | python3 -m json.tool

# Output:
# {
#   "LowerDir": "/var/lib/docker/overlay2/abc.../diff:
#                /var/lib/docker/overlay2/def.../diff",
#   "MergedDir": "/var/lib/docker/overlay2/xyz.../merged",
#   "UpperDir":  "/var/lib/docker/overlay2/xyz.../diff",
#   "WorkDir":   "/var/lib/docker/overlay2/xyz.../work"
# }

# Verify the actual mount registered in the kernel
cat /proc/mounts | grep overlay
# overlay /var/lib/docker/overlay2/xyz.../merged overlay
# rw,relatime,lowerdir=abc/diff:def/diff,upperdir=xyz/diff,workdir=xyz/work 0 0

# See everything the container has written since it started
UPPER=$(docker inspect overlay-demo \
  --format '{{.GraphDriver.Data.UpperDir}}')
find ${UPPER} -type f | head -20
```

### The whiteout mechanism: the thing that bites you in security scans

When a Dockerfile `RUN rm` deletes a file from a lower (read-only) layer, OverlayFS does not actually delete it. It creates a *whiteout file* in the UpperDir — a special character device with major/minor number 0/0. The union mount presents the file as absent to the container, but the bytes remain intact in the lower layer tarball.

This has a critical security implication: **if you `COPY` a secret into a layer and then `RUN rm` it in a later instruction, the secret is still in the image.** Any tool that extracts individual layer tarballs — `docker save`, `skopeo`, a registry scanner — can read it.

```bash
# Demonstrate the whiteout problem
cat > Dockerfile.bad << 'EOF'
FROM alpine:3.18
RUN echo "my_secret_api_key=supersecret123" > /tmp/api.key
RUN rm /tmp/api.key        # THIS DOES NOT REMOVE THE SECRET FROM THE IMAGE
EOF

docker build -t secret-leak-demo -f Dockerfile.bad . --quiet

# Prove the secret is still in the image
# docker history shows every layer command in plaintext
docker history --no-trunc secret-leak-demo | grep "api.key"
# RUN echo "my_secret_api_key=supersecret123" > /tmp/api.key
# ← still visible in layer history, bytes intact in layer tarball

# Save the image as a tar and inspect the layer that contains the secret
docker save secret-leak-demo -o /tmp/leak.tar
tar -tf /tmp/leak.tar        # Lists all layer tarballs — each is extractable
```

```dockerfile
# THE CORRECT APPROACH: BuildKit secret mounts
# The secret is never written to any filesystem layer
# syntax=docker/dockerfile:1
FROM alpine:3.18

RUN --mount=type=secret,id=api_key \
    API_KEY=$(cat /run/secrets/api_key) && \
    configure-app --key="${API_KEY}"
    # /run/secrets/api_key exists ONLY during this RUN instruction
    # It is a tmpfs mount, not a layer — it cannot be extracted
```

```bash
# Build with the secret injected at build time
docker build \
  --secret id=api_key,src=./api_key.txt \
  -t secure-app \
  -f Dockerfile.secure .

# Verify: docker history shows no trace of the secret value
docker history --no-trunc secure-app
```

### Copy-on-write: the hidden I/O performance cost

When a container modifies a file that lives in a lower read-only layer, OverlayFS must first copy the **entire file** to the UpperDir before modifying it — regardless of how small the modification is. For small files this cost is negligible. For large files (database data files, compilation artifacts, log files), this initial copy causes:

- Latency spikes on the first write to any large file
- UpperDir growing to the full size of every large file the container touched
- Disk I/O patterns that look like random writes, confusing I/O profilers

```bash
# BAD: Database writing inside the container
# Every write goes through OverlayFS copy-on-write
docker run -d postgres:16

# GOOD: Database writing to a named volume
# Volume bypasses OverlayFS entirely — direct I/O to the host filesystem
docker run -d \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16

# Verify the mount type — volumes bypass OverlayFS, bind mounts do too
docker inspect pg-container \
  --format '{{range .Mounts}}{{.Type}}: {{.Source}} → {{.Destination}}{{"\n"}}{{end}}'
# volume: /var/lib/docker/volumes/pgdata/_data → /var/lib/postgresql/data

# Rule: any container doing significant I/O should use volumes or bind mounts
# Containers that only read from the filesystem (web servers, APIs) are fine with OverlayFS
```

---

## Section 5 — The OCI Standard: Why Docker Is Not a Walled Garden

Docker is built on open standards. Understanding these standards explains why you can use `podman`, `cri-o`, `nerdctl`, or `containerd` directly — and why images built with Docker run in Kubernetes, AWS Lambda, and Google Cloud Run without modification.

```
┌───────────────────────────────────────────────────────────────────────┐
│                    OCI SPECIFICATION STACK                             │
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │              OCI IMAGE SPEC (image format)                      │  │
│  │  • JSON manifest listing layer content-addressed SHA-256 hashes │  │
│  │  • JSON config: entrypoint, env, labels, exposed ports          │  │
│  │  • Layer tarballs: content-addressed by SHA-256                 │  │
│  │  • Image index: multi-architecture manifest list                │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                              ▲ image is pulled                         │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │              OCI RUNTIME SPEC (config.json)                     │  │
│  │  • Namespaces to create: pid, net, mnt, uts, ipc, user          │  │
│  │  • cgroup resource limits: cpu, memory, pids, blkio             │  │
│  │  • Mount points: rootfs + /proc + /sys + bind mounts            │  │
│  │  • Capabilities to grant or drop                                │  │
│  │  • seccomp profile (syscall whitelist)                          │  │
│  │  • Process: args, env, cwd, user                                │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                              ▲ runc reads this                         │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │              OCI DISTRIBUTION SPEC (registry protocol)          │  │
│  │  • HTTP API for push/pull (content-addressed storage)           │  │
│  │  • Authentication: Bearer tokens, OAuth2                        │  │
│  │  • Content negotiation for multi-arch image indexes             │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘
```

### Bypassing Docker entirely to understand the stack

One of the best exercises for building deep Docker intuition is bypassing it at each layer:

```bash
# ── Level 1: Use containerd's CLI directly (no dockerd) ──
ctr images pull docker.io/library/nginx:alpine
ctr run --rm docker.io/library/nginx:alpine nginx-direct /bin/sh
# You just ran a container without touching the Docker daemon at all

# ── Level 2: Use nerdctl (Docker-compatible CLI on containerd) ──
# nerdctl run is syntactically identical to docker run
nerdctl run --rm nginx:alpine echo "running via containerd directly"

# ── Level 3: Run a container with runc directly (no containerd, no dockerd) ──
mkdir -p /tmp/oci-demo/rootfs && cd /tmp/oci-demo

# Build the rootfs from a Docker image
docker export $(docker create alpine) | tar -C rootfs -xf -

# Generate the default OCI runtime spec — this is what runc reads
runc spec
# Produces config.json — the complete container configuration
# Inspect it to see exactly what Docker constructs for every container
cat config.json | python3 -m json.tool | head -60

# The key sections in config.json:
# "namespaces": [{"type": "pid"}, {"type": "network"}, ...]
# "cgroupsPath": "user.slice/user-1000.slice/session-1.scope"
# "capabilities": {"bounding": [...], "permitted": [...]}
# "seccomp": {"defaultAction": "SCMP_ACT_ERRNO", "syscalls": [...]}
# "mounts": [{"destination": "/proc", "type": "proc"}, ...]

# Run the container directly via runc — no Docker, no containerd
sudo runc run my-manual-container
# You are now inside a container created purely by runc + Linux kernel syscalls
```

Understanding that `config.json` is the lingua franca between all container runtimes explains why Kubernetes can swap Docker for containerd or cri-o without rebuilding images — the image format and runtime spec are standardised.

---

## Section 6 — BuildKit: The Build Engine Most Engineers Underuse

BuildKit is not just a faster `docker build`. It is a completely different build architecture with a DAG-based execution model, content-addressable caching, and a frontend plugin system. Understanding its internals changes how you write Dockerfiles and why certain patterns are dramatically faster than others.

```
┌───────────────────────────────────────────────────────────────────────┐
│                      BUILDKIT ARCHITECTURE                             │
│                                                                        │
│  Dockerfile ──► Frontend (dockerfile2llb) ──► LLB (Low-Level Builder) │
│                                                                        │
│  LLB is a Directed Acyclic Graph (DAG) of build operations.           │
│  Independent branches execute in PARALLEL.                             │
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │                                                                  │  │
│  │   [FROM node:20]        [FROM python:3.11]   ← independent      │  │
│  │        │                       │                                 │  │
│  │   [COPY package.json]  [COPY requirements]   ← parallel!        │  │
│  │        │                       │                                 │  │
│  │   [RUN npm ci]          [RUN pip install]    ← parallel!        │  │
│  │        │                       │                                 │  │
│  │         ───────────────────────                                  │  │
│  │                    │                                             │  │
│  │              [FROM final-base]                                   │  │
│  │              [COPY --from=node ...]                              │  │
│  │              [COPY --from=python ...]                            │  │
│  │                    │                                             │  │
│  │              [Final image]                                       │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  Classic docker build: ALL stages execute SEQUENTIALLY.                │
│  BuildKit: independent branches execute in PARALLEL.                   │
│  BuildKit also skips stages the target stage doesn't depend on.        │
└───────────────────────────────────────────────────────────────────────┘
```

### The cache mount: the most underused BuildKit feature

`--mount=type=cache` creates a persistent build cache directory that survives between builds. Package managers use it to avoid re-downloading dependencies that haven't changed — not a layer cache, not an image cache, a dedicated build-time cache that lives in BuildKit's content store.

```dockerfile
# syntax=docker/dockerfile:1

FROM node:20-alpine AS deps
WORKDIR /app

# --mount=type=cache persists /root/.npm between ALL builds of this image
# It is NOT a layer — it is a bind mount that exists only during this RUN
# The bytes never appear in the final image
RUN --mount=type=cache,target=/root/.npm \
    --mount=type=bind,source=package.json,target=package.json \
    --mount=type=bind,source=package-lock.json,target=package-lock.json \
    npm ci --prefer-offline

# Without cache mount: npm ci downloads all packages every build (~45s)
# With cache mount:    npm ci uses local cache on subsequent builds (~3s)
```

```dockerfile
# syntax=docker/dockerfile:1

FROM python:3.11-slim AS builder
WORKDIR /app

# pip download cache persists between builds
RUN --mount=type=cache,target=/root/.cache/pip \
    --mount=type=bind,source=requirements.txt,target=requirements.txt \
    pip install --no-deps -r requirements.txt

# SSH agent forwarding into the build — private git repos without baking keys
# The SSH socket is a tmpfs mount — it never touches a layer
RUN --mount=type=ssh \
    git clone git@github.com:your-org/private-lib.git /app/private-lib
```

```bash
# Build with SSH agent forwarded (key never appears in any layer)
docker build \
  --ssh default=$SSH_AUTH_SOCK \
  -t my-private-app .
```

### Multi-stage builds: the production image size discipline

The goal is a final image containing only what is needed at runtime. Multi-stage builds make this systematic and enforce it structurally — build tools, test frameworks, and source code cannot leak into the production image because the production stage only receives what you explicitly `COPY --from`.

```dockerfile
# syntax=docker/dockerfile:1

# ────────── Stage 1: Install production dependencies ──────────
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production

# ────────── Stage 2: Build the application ──────────
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build
# This stage has: node_modules + source + compiled output (~900MB)

# ────────── Stage 3: Test gate (CI only, never shipped) ──────────
FROM builder AS test
RUN npm run test && npm run lint
# BuildKit skips this stage entirely when --target production is specified

# ────────── Stage 4: Production image ──────────
FROM node:20-alpine AS production

# Non-root user: root inside the container without user namespaces
# is root on the host — always create a dedicated user
RUN addgroup -S app && adduser -S app -G app

WORKDIR /app

# Copy ONLY the compiled output and production dependencies
# Source code, devDependencies, test files — none of it crosses into here
COPY --from=builder --chown=app:app /app/dist       ./dist
COPY --from=deps    --chown=app:app /app/node_modules ./node_modules

USER app
EXPOSE 3000

# HEALTHCHECK enables orchestrators to detect unhealthy containers
# without relying on process-level signals alone
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

# Exec form (JSON array): node receives signals directly as PID 1
# Shell form ("node server.js"): /bin/sh receives signals, node does not
ENTRYPOINT ["node", "dist/server.js"]
```

```bash
# Build only the production stage — BuildKit skips stages test doesn't depend on
docker build --target production -t my-app:latest .

# Run the test stage explicitly in CI
docker build --target test -t my-app:test .
docker run --rm my-app:test   # exits 0 on pass, non-zero on fail

# The size difference this creates in practice:
docker images my-app
# my-app  latest   abc123   150MB   ← production: runtime only
# my-app  test     def456   900MB   ← test stage: full build environment

# Layer cache hit rate with cache mounts:
# First build:     ~60s  (downloading all dependencies)
# Subsequent code-only changes: ~8s  (deps cached, only app code rebuilt)
```

### Layer ordering as a cache strategy

```dockerfile
# WRONG: Application code invalidates all cache below it
FROM python:3.11-slim
COPY . /app               # ← ANY file change busts everything below
RUN pip install -r /app/requirements.txt    # re-runs on every code change
RUN python setup.py build

# RIGHT: Stable dependencies first, volatile code last
FROM python:3.11-slim
COPY requirements.txt /app/   # ← only changes when dependencies change
RUN pip install -r /app/requirements.txt    # cached unless requirements change
COPY . /app                   # ← code changes don't invalidate pip layer
RUN python setup.py build
```

> **The discipline:** every Dockerfile instruction is a cache invalidation boundary. Ask for each line: "how often does this change?" Instructions that change rarely (system packages, base dependencies) go early. Instructions that change frequently (application code, config files) go late. A properly ordered Dockerfile with BuildKit cache mounts can rebuild a production image from a code change in under 10 seconds.

---

## Part 3

Part 3 covers running and operating Docker safely in production:

- **Container networking internals** — veth pairs, the docker0 bridge, iptables rules, and embedded DNS
- **The security layer stack** — Linux capabilities, seccomp syscall filtering, AppArmor, and rootless Docker
- **The PID 1 signals problem** — why your containers take 10 seconds to shut down and how to fix it permanently
- **The production debugging toolkit** — nsenter, strace, cgroup stats, and commands that bypass the Docker abstraction entirely
- **Ten truths senior engineers know that articles skip** — the production hard-won insights that don't fit in tutorials

---

*All commands validated against Docker Engine 26.x, containerd 1.7.x, runc 1.1.x, Linux kernel 6.x with cgroups v2.*
