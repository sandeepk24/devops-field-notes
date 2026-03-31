# The Real Difference Between `docker build` and `docker buildx`

> It's not a version upgrade. It's a different mental model for where your builds live in the world.

**Tags:** DevOps · Docker · Container Builds · CI/CD  
**Level:** Intermediate → Advanced  
**Read time:** ~18 min

---

Here is a trap I see engineers fall into constantly: they encounter `docker buildx` for the first time, treat it as the "new version" of `docker build`, and move on. That framing creates real confusion later — especially the moment an image disappears after a successful build, or a CI pipeline needs to serve ARM and x86 simultaneously.

This article is a field guide. We will go deep on what each command actually does, when to use which, why the output behavior differs, and how to use `buildx` correctly in a real delivery pipeline. No hand-waving — just the mechanics.

---

## Two Commands, Two Philosophies

Let's start with the surface-level comparison before going deeper.

| | `docker build` (classic) | `docker buildx build` (extended) |
|---|---|---|
| **Output** | Always loads into local Docker image store | Explicit — local store, registry, OCI archive, or nowhere |
| **Builder** | Local Docker daemon | Configurable backend (local, container, Kubernetes, remote) |
| **Platforms** | Single (host architecture) | Multi-platform via `--platform` |
| **Cache** | Layer cache on local disk | Registry-backed, GHA, inline, S3 |
| **Use case** | Local development | CI/CD pipelines, platform delivery |

Looking at those two commands side by side, the syntactic difference seems trivial:

```bash
# Classic
docker build -t my-app:latest .

# Extended
docker buildx build -t my-app:latest .
```

The architectural difference is not trivial.

`docker build` is designed around a local workflow. It assumes you want an image, you want it now, and you want it available on your machine. That contract — trigger → build → image in local store — is its entire design surface.

`docker buildx` treats the build as a more general compute operation. The builder is configurable. The output is configurable. The platform targets are configurable. The cache is configurable. You are not just building an image — you are running a build job that produces some artifact.

> **Core mental model:** `docker build` is a tool for building images. `docker buildx` is a framework for running configurable build pipelines. Most engineers only need the former. Platform teams and CI/CD pipelines need the latter.

---

## The BuildKit Difference

To understand `buildx`, you first have to understand BuildKit — because `buildx` is essentially a first-class interface to BuildKit's full capability surface.

The original `docker build` used the legacy build engine built into the Docker daemon. It was sequential: parse the Dockerfile, execute instructions one by one, snapshot each layer. It worked, but had real limitations.

BuildKit, which became default in Docker 23.0+, is a complete rewrite of the build subsystem. It introduces:

- **Parallel execution** — independent stages build concurrently
- **Advanced cache semantics** — cache import/export, registry-backed cache, inline cache
- **Secret mounting** — `--mount=type=secret` for credentials that never enter a layer
- **SSH agent forwarding** — forward your SSH socket into a build step without embedding keys
- **Multi-platform targeting** — cross-compilation and QEMU emulation for non-native architectures
- **Pluggable frontends** — Dockerfile is just one option; HLB, Buildpacks, and others are possible

Modern `docker build` also uses BuildKit now (since Docker 23.0). So you get many of these benefits with either command. The distinction with `buildx` is that it exposes the full BuildKit API — including builder management, output routing, and multi-platform targeting — through a clean CLI interface.

```bash
# Check which builder is currently active
docker buildx ls

# Output example:
# NAME/NODE       DRIVER/ENDPOINT    STATUS    PLATFORMS
# default *       docker             running   linux/amd64, linux/arm64/v8
# my-builder      docker-container   running   linux/amd64, linux/arm64, linux/s390x
```

---

## The Output Behavior That Surprises Everyone

This is the single most common source of confusion. Memorize it.

With `docker build`, output goes to the local Docker image store. Always. That's where the image ends up. You can `docker run` it immediately after the build completes.

With `docker buildx build`, the output destination is not predetermined. If you do not specify where the result should go, it may build successfully — with exit code 0 — and produce nothing you can immediately use.

> ⚠️ **The "successful build, no image" trap**
>
> Running `docker buildx build -t my-app:latest .` will often succeed silently without loading the image into your local store. This is not a bug. It's intentional design — the builder doesn't assume where you want the result. You must be explicit.

Here are the three most common output scenarios and their explicit flags:

```bash
# Load into local Docker image store (replaces docker build behavior)
docker buildx build --load -t my-app:latest .

# Push directly to a registry (no local copy)
docker buildx build --push -t myrepo/my-app:latest .

# Export as OCI tarball to disk
docker buildx build --output type=oci,dest=./my-app.tar .

# Build and discard (useful for cache warming or validation)
docker buildx build --output type=cacheonly .
```

The mental model that helps: `docker buildx build` is like a compiler that has been separated from the linker. The compilation step (turning your Dockerfile into layers) and the output step (where does the artifact go?) are distinct operations. You have to specify both.

---

## Multi-Platform Builds: Where buildx Earns Its Keep

This is the capability that makes `buildx` genuinely irreplaceable for platform teams. Not possible at all with classic `docker build`.

```bash
# Build a single image manifest supporting three architectures
docker buildx build \
  --platform linux/amd64,linux/arm64,linux/arm/v7 \
  -t myrepo/my-app:latest \
  --push .
```

This pushes a single OCI Image Index to the registry. When someone pulls `myrepo/my-app:latest`, Docker automatically selects the variant that matches their host architecture. The consumer never has to think about architecture — it's handled at the manifest level.

```
Dockerfile ──► docker buildx ──► linux/amd64  ──┐
               (BuildKit)     ► linux/arm64  ──┼──► OCI Image Index
                              ► linux/arm/v7 ──┘    (single tag → all arch)
```

The build process works via two mechanisms depending on your setup:

**Cross-compilation** is the fast path. If your Dockerfile uses multi-stage builds and your base images have multi-arch variants, BuildKit can compile native binaries for each target architecture on your host machine — no emulation overhead.

**QEMU emulation** is the fallback. When cross-compilation isn't viable (e.g. the build process itself runs architecture-specific binaries), BuildKit registers QEMU as a binary format handler via `binfmt_misc` and runs the build inside an emulated environment.

```bash
# Register QEMU emulators (needed for cross-arch emulation)
docker run --privileged --rm \
  tonistiigi/binfmt --install all

# Create a builder with multi-platform support
docker buildx create --name multiplatform --driver docker-container --use
docker buildx inspect --bootstrap
```

---

## Cache Control: The CI/CD Superpower

This is where `buildx` separates itself most dramatically for production pipeline work. The cache API available via `buildx` is qualitatively different from what classic `docker build` offers.

### Registry-Backed Cache

```bash
# Push cache to registry; subsequent builds pull from it
docker buildx build \
  --cache-from type=registry,ref=myrepo/my-app:buildcache \
  --cache-to   type=registry,ref=myrepo/my-app:buildcache,mode=max \
  -t myrepo/my-app:latest \
  --push .
```

With `mode=max`, every intermediate layer — not just the final image — gets pushed to the cache. A fresh CI runner that has never seen your codebase before can warm itself from the registry and skip layers that haven't changed. No local volume mounts needed. No persistent runner state required.

### GitHub Actions Cache Backend

```bash
# Use GitHub Actions cache API (requires actions/cache setup)
docker buildx build \
  --cache-from type=gha \
  --cache-to   type=gha,mode=max \
  -t myrepo/my-app:latest \
  --push .
```

### Inline Cache (Simpler, Good for Single-Platform)

```bash
# Embed cache metadata in the image itself
docker buildx build \
  --cache-from myrepo/my-app:latest \
  --build-arg  BUILDKIT_INLINE_CACHE=1 \
  -t myrepo/my-app:latest \
  --push .
```

---

## Builder Backends

One of the less-discussed features of `buildx` is that the builder itself is a first-class concept. You can have multiple builders configured simultaneously and switch between them.

```bash
# docker driver — uses the local daemon (default, like docker build)
docker buildx create --driver docker

# docker-container — runs BuildKit in a container, unlocks multi-platform
docker buildx create --driver docker-container --use

# kubernetes — spins up BuildKit pods in your cluster
docker buildx create --driver kubernetes \
  --driver-opt replicas=4 \
  --driver-opt namespace=build-system

# remote — connect to a BuildKit instance you manage yourself
docker buildx create --driver remote \
  tcp://buildkit.internal:1234
```

The Kubernetes driver is particularly powerful for large teams: you can run distributed, autoscaling build infrastructure inside your cluster with fine-grained resource controls, and engineers interact with it as though it's a local builder.

---

## Where Each Command Belongs in Your Workflow

| Scenario | Use this | Why |
|---|---|---|
| Local dev iteration | `docker build` | Automatic local store output. Fastest feedback loop for a single platform. |
| Quick local validation | `docker build` | No builder config needed. Works immediately, output is where you expect it. |
| Multi-platform production image | `buildx --push` | Only command that supports `--platform linux/amd64,linux/arm64` with a single manifest. |
| CI/CD pipeline | `buildx` | Registry cache import/export, explicit output routing, ephemeral-runner-safe. |
| Push to registry (single platform) | Either | Both work. `buildx --push` is cleaner; `docker build + docker push` is more familiar. |
| Build + run immediately | `docker build` | Automatic local store load. With `buildx` you'd need `--load` explicitly. |
| Secret mounting in build | `buildx` | BuildKit `--mount=type=secret` is only cleanly surfaced via `buildx`. |
| Distributed/remote build fleet | `buildx` | Builder backend system supports kubernetes, remote, and docker-container drivers. |

---

## A Production CI Pattern Worth Copying

This is a complete, production-grade GitHub Actions workflow using `buildx` with GHA cache and multi-platform output. Every flag is deliberate.

```yaml
name: Build and push multi-platform image

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:

      # Required: sets up QEMU for cross-platform emulation
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      # Required: bootstraps buildx with docker-container driver
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
          # Cache import: pull warm cache from GHA cache API
          cache-from: type=gha
          # Cache export: push all layers (mode=max) for downstream runs
          cache-to: type=gha,mode=max
```

A few things worth calling out in this workflow:

The `setup-qemu-action` step is required whenever your target platforms include architectures not native to the runner. Without it, building for ARM will either fail or use a very slow software emulation path.

The `setup-buildx-action` step creates a `docker-container` builder, which is required for multi-platform builds. The default `docker` driver does not support the `--platform` flag with multiple values.

`mode=max` on the cache-to directive caches all intermediate layers. This is more storage — but for complex multi-stage Dockerfiles, it pays back in build time on subsequent runs, particularly when only a late stage has changed.

---

## The Secret Mounting You Should Be Using

A less-used but high-value `buildx` capability: BuildKit secret mounting. This gives you access to credentials inside a build step without the secret ever appearing in a committed image layer.

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim

# The secret is mounted at /run/secrets/pip-token
# It is NOT committed to any image layer
RUN --mount=type=secret,id=pip-token \
    pip install \
      --index-url https://$(cat /run/secrets/pip-token)@private.registry/simple \
      my-private-package
```

```bash
# Pass the secret at build time
docker buildx build \
  --secret id=pip-token,env=PIP_TOKEN \
  --load -t my-app:latest .
```

With classic `docker build`, the common workaround was `--build-arg`, which embeds the secret in the image history and is visible to anyone with image access. This is a meaningful security improvement for builds that need credentials.

---

## Common Mistakes and How to Avoid Them

### ⚠️ Mistake 1 — Forgetting `--load` in local development

You run `docker buildx build -t my-app .`, it succeeds, you run `docker run my-app` and get "image not found." You need `--load`. The builder completed the job — you just didn't tell it where to put the result.

```bash
# Wrong
docker buildx build -t my-app:latest .

# Right
docker buildx build --load -t my-app:latest .
```

### ⚠️ Mistake 2 — Using `--load` with multi-platform

`--load` only supports single-platform images. If you try to use it with `--platform linux/amd64,linux/arm64`, you'll get an error. Multi-platform output must go to a registry via `--push`, or to an OCI tar via `--output type=oci,dest=./image.tar`.

```bash
# This will error
docker buildx build --platform linux/amd64,linux/arm64 --load -t my-app .

# Correct for multi-platform
docker buildx build --platform linux/amd64,linux/arm64 --push -t myrepo/my-app .
```

### ⚠️ Mistake 3 — Not creating a `docker-container` builder

Multi-platform builds via `--platform` with multiple targets require the `docker-container` builder driver. The default `docker` driver won't work.

```bash
# Create and activate the right builder first
docker buildx create --driver docker-container --use
docker buildx inspect --bootstrap
```

### ✅ Tip — Inspect your builder capabilities

Run `docker buildx inspect --bootstrap` to see exactly which platforms your current builder supports, whether BuildKit is healthy, and what driver is in use. Always check this when a build behaves unexpectedly.

---

## The Bottom Line

**`docker build`** = a local tool for building images. It assumes you want the result on your machine, for your architecture, right now. That assumption makes it simple and fast for local work.

**`docker buildx`** = a build framework. The builder is configurable. The output is explicit. The platforms are multi-target. The cache is registry-backed. It's the right tool the moment your build escapes the laptop.

The switch from `docker build` to `docker buildx` is not a version upgrade. It's a change in scope: from "build an image" to "run a build job with configurable compute, output, and cache."

For most engineers working locally on a single-architecture service: `docker build` is still completely fine. Do not introduce complexity you don't need.

For platform engineers, CI/CD infrastructure, or any team shipping ARM-compatible images: `docker buildx` with a `docker-container` builder, registry cache, and explicit output flags is not optional. It's how modern container delivery actually works.

Learn both. Use each where it fits. The confusion goes away the moment you stop treating them as versions of the same thing.

---

*All examples tested against Docker Engine 25.0 / BuildKit 0.13+. The `--platform` flag with multiple targets requires the docker-container or kubernetes driver. The GHA cache backend requires the `docker/build-push-action` GitHub Action.*
