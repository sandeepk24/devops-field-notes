# Docker Fundamentals for the Advanced DevOps Engineer
## Part 1 of 3 — The Execution Stack, Linux Namespaces & cgroups

> *Most Docker articles stop at `docker run`. This one starts there.*

**Series:** Docker Internals · Part 1 of 3  
**Tags:** `Docker` `Linux Internals` `containerd` `runc` `Namespaces` `cgroups`  

---

**What's in this series**

| Part | Topic | Focus |
|---|---|---|
| **Part 1 — Today** | Execution Stack, Namespaces, cgroups | How Docker works at the kernel level |
| Part 2 — Next | OverlayFS, OCI Standard, BuildKit | Storage, open standards, build engine |
| Part 3 — Following | Networking, Security, Signals, Production Debugging | Running and operating Docker safely |

---

## Preface: What "Knowing Docker" Actually Means

There are three levels at which engineers understand Docker:

**Level 1 — The CLI user.** They know `docker build`, `docker run`, `docker-compose up`. Docker is a black box that makes things work.

**Level 2 — The practitioner.** They understand images vs containers, layers, volumes, networks. They write clean Dockerfiles and can debug most problems.

**Level 3 — The operator.** They understand what happens between the moment you press Enter on `docker run` and the moment a process starts receiving syscalls inside an isolated environment. They know why containers are not VMs, what Linux kernel features make isolation possible, where things break under load, and exactly which layer to reach into when the abstraction leaks.

This series is for engineers moving from Level 2 to Level 3. It will not explain what a Dockerfile is. It will explain what happens inside the kernel when Docker processes one.

---

## Section 1 — The Full Execution Stack: What Really Runs When You Type `docker run`

The single most important mental model upgrade an advanced engineer can make is understanding that "Docker" is not one thing. It is a chain of five distinct components, each with a precise role, communicating over defined interfaces.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        YOUR TERMINAL                                 │
│                     docker run nginx:latest                          │
└──────────────────────────────┬──────────────────────────────────────┘
                                │ REST API over Unix socket
                                │ /var/run/docker.sock
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         DOCKERD (Docker Engine)                      │
│  • Accepts CLI/API commands                                          │
│  • Manages image pulls from registries                               │
│  • Sets up networking (libnetwork) — bridge, overlay, macvlan        │
│  • Creates volumes, allocates IPs, configures port mapping           │
│  • Builds the OCI runtime spec (config.json)                         │
│  • Delegates container lifecycle to containerd via gRPC              │
└──────────────────────────────┬──────────────────────────────────────┘
                                │ gRPC (containerd API)
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    CONTAINERD (container runtime)                    │
│  • Industry-standard, CNCF-graduated project                         │
│  • Manages image storage (content-addressable store)                 │
│  • Creates OCI bundle: rootfs snapshot + config.json                 │
│  • Manages container task lifecycle                                  │
│  • Spawns containerd-shim for each container                         │
│  • Invokes runc to create the actual container                       │
└──────────────────────────────┬──────────────────────────────────────┘
                                │ fork/exec
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│               CONTAINERD-SHIM (per-container process)                │
│  • Stays alive even if dockerd/containerd crash or restart           │
│  • Manages container stdio (stdin/stdout/stderr)                     │
│  • Reports container exit status to containerd                       │
│  • This is why your containers survive daemon restarts               │
└──────────────────────────────┬──────────────────────────────────────┘
                                │ executes
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     RUNC (OCI runtime, reference impl)               │
│  • Reads config.json (OCI Runtime Spec)                              │
│  • Calls clone() syscall to create namespaces                        │
│  • Writes limits to /sys/fs/cgroup/                                  │
│  • Mounts the rootfs via OverlayFS                                   │
│  • Applies seccomp, AppArmor, capabilities                           │
│  • exec()s your container process                                    │
│  • EXITS — it is ephemeral, not a daemon                             │
└──────────────────────────────┬──────────────────────────────────────┘
                                │ syscalls: clone, mount, setns, execve
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      LINUX KERNEL                                    │
│  • Namespaces: pid, net, mnt, ipc, uts, user, cgroup, time           │
│  • cgroups v2: CPU, memory, I/O, pids hierarchy                      │
│  • OverlayFS: union mount of image layers                            │
│  • seccomp: syscall filter                                           │
│  • Capabilities: fine-grained root privilege restrictions            │
└─────────────────────────────────────────────────────────────────────┘
```

### The critical insight most engineers miss: runc exits

`runc` is not a daemon. It is a short-lived binary that runs, sets up the container environment, starts your process, and then exits. The container process becomes an orphan — adopted by `containerd-shim`, which is the process that keeps the container alive. This is why:

1. You can restart `dockerd` or `containerd` without killing running containers
2. `docker ps` can still work after a daemon crash — the shim held the state
3. When you debug with `ps aux | grep containerd-shim` on a host, you see one shim per running container

```bash
# On a host with running containers — observe this hierarchy
ps -ef --forest | grep -E "containerd|docker|runc" | head -30

# Output (simplified):
# root  containerd                    ← always running
# root    containerd-shim-runc-v2     ← one per container
# root      nginx                     ← your container process
# root    containerd-shim-runc-v2     ← another container
# root      node                      ← another container process
```

### Why this matters in production

When a container crashes and you see `dockerd` is also unhealthy, these are independent events. A hung `dockerd` will not kill running containers — the shims hold them. This is also why `docker stats` may fail while containers are still serving traffic perfectly fine.

---

## Section 2 — Linux Namespaces: The Isolation Machinery

A container is not a VM. It is a Linux process with restricted visibility. Namespaces are the kernel mechanism that creates that restriction. Understanding each namespace tells you exactly what a container can and cannot see.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           LINUX HOST KERNEL                               │
│                                                                            │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │                    CONTAINER NAMESPACE BUNDLE                      │   │
│  │                                                                    │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐    │   │
│  │  │  PID NS      │  │  NET NS      │  │   MNT NS             │    │   │
│  │  │ PID 1 = your │  │ eth0 = veth  │  │ / = overlayfs mount  │    │   │
│  │  │ process      │  │ 172.17.0.2   │  │ /proc = procfs       │    │   │
│  │  │              │  │ lo = 127.0.0.│  │ /sys  = sysfs        │    │   │
│  │  └──────────────┘  └──────────────┘  └──────────────────────┘    │   │
│  │                                                                    │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐    │   │
│  │  │  UTS NS      │  │  IPC NS      │  │   USER NS            │    │   │
│  │  │ hostname =   │  │ private      │  │  UID 0 (root) inside │    │   │
│  │  │ "my-app"     │  │ message      │  │  maps to UID 1000    │    │   │
│  │  │              │  │ queues,      │  │  outside (rootless)  │    │   │
│  │  │              │  │ semaphores   │  │                      │    │   │
│  │  └──────────────┘  └──────────────┘  └──────────────────────┘    │   │
│  │                                                                    │   │
│  │  ┌──────────────┐  ┌──────────────────────────────────────────┐  │   │
│  │  │  CGROUP NS   │  │   TIME NS (kernel 5.6+)                  │  │   │
│  │  │ container    │  │  clock_gettime() returns container-local  │  │   │
│  │  │ sees its own │  │  time — container can have different      │  │   │
│  │  │ cgroup root  │  │  clock offset from host                   │  │   │
│  │  └──────────────┘  └──────────────────────────────────────────┘  │   │
│  └────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

### The seven namespaces decoded

**PID namespace** — The container has its own PID tree starting at PID 1. Your container process is PID 1 inside the namespace, but has a completely different (much higher) PID on the host. This is why `docker exec nginx kill -9 1` is a common footgun — you're sending SIGKILL to PID 1 in the container's namespace, which terminates the container.

**NET namespace** — Each container gets its own network stack: interfaces, routing tables, iptables rules, sockets. The `eth0` inside a container is a virtual ethernet pair (veth) — one end inside the container namespace, the other on the host's `docker0` bridge. Traffic flows: `container eth0 → host veth pair → docker0 bridge → iptables → physical NIC`.

**MNT namespace** — The container has its own mount table. The host's filesystem is not visible unless you explicitly mount it with `-v`. The container's root filesystem is an OverlayFS mount — read-only image layers below, a thin read-write layer on top.

**UTS namespace** — Controls the `hostname` and `domainname` syscalls. UTS stands for UNIX Time Sharing. This is why `hostname` inside a container returns the container ID, not the host's hostname.

**IPC namespace** — Isolates System V IPC objects (message queues, semaphores, shared memory) and POSIX message queues. Two containers cannot communicate via shared memory unless they share an IPC namespace — which is what `--ipc=host` or Kubernetes `shareProcessNamespace` enables.

**USER namespace** — Maps UIDs/GIDs between the container and host. Rootless Docker uses this: UID 0 inside the container maps to a non-privileged UID on the host. Without user namespaces, root in a container is root on the host — a significant security risk.

**CGROUP namespace** — Controls what the container sees when it reads `/sys/fs/cgroup/`. Without this, a container could see the host's entire cgroup hierarchy, leaking information about other workloads.

### Verifying namespaces from the host

```bash
# Get a container's PID on the host
CONTAINER_PID=$(docker inspect --format '{{.State.Pid}}' my-container)

# See all namespaces for that process
ls -la /proc/${CONTAINER_PID}/ns/

# Different inode numbers = different namespaces from host init (PID 1)
ls -lai /proc/1/ns/net          # host network namespace inode
ls -lai /proc/${CONTAINER_PID}/ns/net  # container's — should differ

# Enter a specific namespace manually (what docker exec does under the hood)
nsenter --target ${CONTAINER_PID} \
  --mount --uts --ipc --net --pid \
  -- /bin/bash
# You are now inside the container's namespaces, running as root on the host
```

---

## Section 3 — cgroups v2: Resource Isolation Is a Filesystem

Control groups are a Linux kernel feature that exposes resource limits as a hierarchical filesystem under `/sys/fs/cgroup/`. This is not an abstraction — it is literally a filesystem. Writing a number to a file sets a resource limit. Reading a file reads current usage.

```
/sys/fs/cgroup/
└── system.slice/
    └── docker-<container_id>.scope/
        ├── memory.max          ← write to set hard memory limit (bytes)
        ├── memory.current      ← read to see current usage
        ├── memory.stat         ← detailed memory breakdown
        ├── memory.events       ← OOM event counters
        │
        ├── cpu.max             ← "quota period" format: "50000 100000" = 50% CPU
        ├── cpu.stat            ← usage_usec, throttled_usec  ← KEY SIGNAL
        │
        ├── io.max              ← "major:minor rbps=N wbps=N" per device
        │
        ├── pids.max            ← max number of processes (fork bomb protection)
        └── pids.current        ← current process count
```

```bash
# Run a container with explicit limits
docker run -d \
  --name stress-test \
  --cpus="1.5" \
  --memory="512m" \
  --pids-limit=100 \
  nginx:alpine

# Find the container ID
CONTAINER_ID=$(docker inspect --format '{{.Id}}' stress-test)

# Read kernel-enforced limits directly — no Docker abstraction layer
cat /sys/fs/cgroup/system.slice/docker-${CONTAINER_ID}.scope/memory.max
# 536870912  (512 × 1024 × 1024 = 512 MiB in bytes)

cat /sys/fs/cgroup/system.slice/docker-${CONTAINER_ID}.scope/cpu.max
# 150000 100000  (1.5 CPUs = 150ms per 100ms scheduling period)

cat /sys/fs/cgroup/system.slice/docker-${CONTAINER_ID}.scope/pids.max
# 100
```

### The OOM killer and what engineers get wrong

When a container's memory exceeds `memory.max`, the kernel OOM killer fires. It doesn't kill the process that exceeded the limit first — it kills whatever the kernel's heuristic picks as the best candidate, which is often PID 1 in the container. Your container exits with `OOMKilled: true` in `docker inspect`.

```bash
# Identify OOM events
dmesg | grep -i "oom\|out of memory\|killed process"

# Docker-level view
docker inspect my-container --format '{{.State.OOMKilled}}'
# true

# Kernel-level OOM stats for the container
CONTAINER_ID=$(docker inspect --format '{{.Id}}' my-container)
cat /sys/fs/cgroup/system.slice/docker-${CONTAINER_ID}.scope/memory.events
# anon   12345
# file   67890
# oom    1     ← number of times OOM was triggered
# oom_kill 1  ← number of processes killed by OOM
```

### CPU throttling: the silent production killer

CPU limits use the kernel's Completely Fair Scheduler. When a container exceeds its CPU quota, the kernel throttles it — the process doesn't error, it just stalls. This produces mysterious latency spikes with no errors in logs.

```bash
# Check if your container is being CPU throttled right now
CONTAINER_ID=$(docker inspect --format '{{.Id}}' my-container)
cat /sys/fs/cgroup/system.slice/docker-${CONTAINER_ID}.scope/cpu.stat

# Key fields to watch:
# usage_usec          ← total CPU time used (microseconds)
# user_usec           ← time in user space
# system_usec         ← time in kernel space
# throttled_usec      ← time the container was throttled ← YOUR SIGNAL
# throttled_periods   ← number of periods where throttling occurred

# Rule of thumb: if throttled_usec / usage_usec > 0.05 (5%),
# you have a CPU limit problem causing latency
```

> **Production insight:** CPU throttling never appears in application logs. A service with `--cpus=0.5` handling burst traffic will show rising p99 latency with zero errors in its logs. Engineers spend hours blaming the application before checking `cpu.stat`. Always check this first when latency is anomalous.

---

## Coming in Part 2

Part 2 covers how Docker actually stores images and what that means for security and performance:

- **OverlayFS mechanics** — how lowerdir, upperdir, workdir, and mergeddir create the container's filesystem
- **The whiteout problem** — why `RUN rm` doesn't delete secrets from your image layers and what does
- **Copy-on-write performance costs** — when OverlayFS is the bottleneck and when to bypass it with volumes
- **The OCI Standard** — why Docker images run in Podman, Kubernetes, and Firecracker without modification
- **BuildKit's DAG execution model** — parallel stage builds, cache mounts, secret mounts, SSH forwarding
- **Multi-stage production Dockerfiles** — the pattern that takes an image from 900MB to 150MB

---

*All commands validated against Docker Engine 26.x, containerd 1.7.x, runc 1.1.x, Linux kernel 6.x with cgroups v2. Cgroup paths shown use the systemd cgroup driver (default on modern Linux distributions).*
