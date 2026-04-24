# Docker Compose → ECS: The Local-to-Production Playbook Every DevOps Engineer Needs 🐳

> **The workflow Senior Developers and Senior DevOps Engineers actually use** — build, wire, test, and validate your entire stack locally with Docker Compose, then ship confidently to AWS ECS.  
> No more "works on my machine." No more broken staging surprises.

---

## 📋 Table of Contents

1. [Why Docker Compose Exists — The Problem It Solves](#1-why-docker-compose-exists)
2. [How Docker Compose Actually Works — Internals](#2-how-docker-compose-actually-works)
3. [The `docker-compose.yml` File — Every Field Explained](#3-the-docker-composeyml-file)
4. [Real-World Project: Python API + PostgreSQL + Redis + Nginx](#4-real-world-project)
5. [The Developer Workflow — What the Sr. Dev Does](#5-the-developer-workflow)
6. [The DevOps Workflow — What You Do](#6-the-devops-workflow)
7. [Networking Deep Dive — How Containers Talk](#7-networking-deep-dive)
8. [Volumes & Data Persistence](#8-volumes--data-persistence)
9. [Environment Variables & Secrets Management](#9-environment-variables--secrets-management)
10. [Health Checks — The Bridge to ECS](#10-health-checks)
11. [Multi-Stage Dockerfile — Dev vs Prod Images](#11-multi-stage-dockerfile)
12. [Local → ECS: The Translation Map](#12-local--ecs-the-translation-map)
13. [Common Failures & How to Debug Them](#13-common-failures--debugging)
14. [Production Checklist Before Pushing to ECS](#14-production-checklist)
15. [Quick Reference Cheat Sheet](#15-quick-reference-cheat-sheet)

---

## 1. Why Docker Compose Exists

### The Problem Before Compose

Imagine you have a typical application stack:

```
Frontend  (React)     → port 3000
API       (Python/FastAPI) → port 8000
Database  (PostgreSQL) → port 5432
Cache     (Redis)      → port 6379
Proxy     (Nginx)      → port 80/443
```

Without Docker Compose, a developer would need to:

```bash
# Terminal 1 — start postgres
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=myapp \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  --network myapp-net \
  postgres:15

# Terminal 2 — start redis
docker run -d \
  --name redis \
  -p 6379:6379 \
  --network myapp-net \
  redis:7-alpine

# Terminal 3 — start API (must wait for postgres!)
docker run -d \
  --name api \
  -e DATABASE_URL=postgresql://postgres:secret@postgres:5432/myapp \
  -e REDIS_URL=redis://redis:6379 \
  -p 8000:8000 \
  --network myapp-net \
  myapp-api:latest

# ...and remember all these flags, order, dependencies, volumes
# And tear it ALL down when done
# And recreate the network first
# And remember to rebuild when code changes
```

**This is chaos.** Docker Compose solves this with a single declarative file and three commands.

### What Docker Compose Gives You

```
docker compose up    → Entire stack running
docker compose down  → Entire stack gone
docker compose logs  → All service logs in one view
```

---

## 2. How Docker Compose Actually Works — Internals

Understanding the internals makes debugging 10x faster.

### The Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Your Machine                                  │
│                                                                  │
│  docker-compose.yml                                             │
│         │                                                        │
│         ▼                                                        │
│  ┌─────────────────┐                                            │
│  │  Compose CLI    │  (parses YAML, validates, plans)           │
│  │  (docker compose│                                            │
│  │   command)      │                                            │
│  └────────┬────────┘                                            │
│           │  Docker API calls (unix socket /var/run/docker.sock)│
│           ▼                                                      │
│  ┌─────────────────┐                                            │
│  │   Docker Daemon │  (containerd + runc)                       │
│  │   (dockerd)     │                                            │
│  └────────┬────────┘                                            │
│           │                                                      │
│   ┌───────┴───────────────────────────────┐                    │
│   │         PROJECT NETWORK               │                    │
│   │         (myapp_default)               │                    │
│   │                                        │                    │
│   │  ┌──────────┐  ┌──────────┐           │                    │
│   │  │ api      │  │ postgres │           │                    │
│   │  │container │  │container │           │                    │
│   │  └──────────┘  └──────────┘           │                    │
│   │  ┌──────────┐  ┌──────────┐           │                    │
│   │  │ redis    │  │ nginx    │           │                    │
│   │  │container │  │container │           │                    │
│   │  └──────────┘  └──────────┘           │                    │
│   └───────────────────────────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘
```

### What Happens When You Run `docker compose up`

```
Step 1: Parse & Validate
  └── Read docker-compose.yml
  └── Resolve variables from .env
  └── Validate syntax and references

Step 2: Create Network(s)
  └── Creates bridge network: <project_name>_default
  └── (or named networks you defined)

Step 3: Create Volumes
  └── Named volumes are created if they don't exist
  └── Bind mounts are validated

Step 4: Pull/Build Images
  └── build: → runs docker build
  └── image: → pulls if not locally cached

Step 5: Create & Start Containers (respecting depends_on)
  └── Containers connected to network automatically
  └── DNS entries registered for each service name

Step 6: Attach to logs (if not -d/--detach)
```

### Project Naming — Critical to Understand

Every Compose project gets a **project name** (defaults to the directory name). This prefixes ALL resources:

```bash
# If your directory is "myapp" and services are "api", "postgres":
# Network:   myapp_default
# Container: myapp-api-1
# Volume:    myapp_pgdata
# Container: myapp-postgres-1

# Set it explicitly to avoid surprises:
docker compose -p myapp up
# OR in docker-compose.yml:
# name: myapp
```

---

## 3. The `docker-compose.yml` File — Every Field Explained

```yaml
# docker-compose.yml
# SCHEMA VERSION: Compose Specification (no more "version: 3" needed in modern Docker)

name: myapp  # Project name — prefixes all containers, networks, volumes

# ══════════════════════════════════════════════════════════════
# SERVICES — Each is one containerized process
# ══════════════════════════════════════════════════════════════
services:

  # ── SERVICE: API ──────────────────────────────────────────
  api:
    # --- HOW TO GET THE IMAGE ---
    build:
      context: ./api          # Path to Dockerfile directory
      dockerfile: Dockerfile  # Default is "Dockerfile"
      target: development     # Multi-stage: which stage to build to
      args:                   # Build-time ARGs (not available at runtime)
        BUILD_VERSION: "1.0.0"
        NODE_ENV: development

    # OR use a pre-built image (never both build + image):
    # image: my-ecr-account.dkr.ecr.us-east-1.amazonaws.com/api:latest

    # --- CONTAINER SETTINGS ---
    container_name: myapp-api  # Override default naming (careful: breaks scaling)

    # --- PORT MAPPING ---
    ports:
      - "8000:8000"   # host_port:container_port
      # - "127.0.0.1:8000:8000"  # Bind to localhost only (more secure)
      # - "8000"      # Random host port — use for scaled services

    # --- ENVIRONMENT VARIABLES ---
    environment:
      # Direct values (visible in docker inspect — NOT for secrets)
      APP_ENV: development
      LOG_LEVEL: debug
      # Reference from shell or .env file
      DATABASE_URL: "postgresql://${DB_USER}:${DB_PASSWORD}@postgres:5432/${DB_NAME}"
      REDIS_URL: "redis://redis:6379/0"

    # OR load from a file:
    env_file:
      - .env           # Default env file
      - .env.local     # Override (later files override earlier)

    # --- STARTUP ORDER ---
    depends_on:
      postgres:
        condition: service_healthy   # Wait for health check to pass
      redis:
        condition: service_started   # Just wait for container to start
        # condition: service_completed_successfully  # Wait for one-shot jobs

    # --- VOLUMES ---
    volumes:
      - ./api:/app                    # Bind mount: local code → container
                                      # Hot reload during development!
      - /app/node_modules             # Anonymous volume: don't override node_modules
      - api_cache:/tmp/cache          # Named volume: shared cache

    # --- NETWORKING ---
    networks:
      - backend          # Can be on multiple networks
      - frontend

    # --- HEALTH CHECK ---
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s      # Check every 10 seconds
      timeout: 5s        # Fail if no response in 5 seconds
      retries: 3         # Mark unhealthy after 3 failures
      start_period: 30s  # Grace period before checks count

    # --- RESTART POLICY ---
    restart: unless-stopped
    # Options: no | always | on-failure | unless-stopped

    # --- RESOURCE LIMITS ---
    deploy:
      resources:
        limits:
          cpus: "0.5"     # Max 50% of one CPU
          memory: 512M    # Max 512MB RAM
        reservations:
          cpus: "0.25"    # Guaranteed 25% CPU
          memory: 256M    # Guaranteed 256MB RAM

    # --- WORKING DIRECTORY ---
    working_dir: /app

    # --- OVERRIDE THE DEFAULT COMMAND ---
    command: uvicorn main:app --host 0.0.0.0 --port 8000 --reload

    # OR override entrypoint entirely:
    # entrypoint: ["/bin/sh", "-c"]

    # --- LABELS ---
    labels:
      com.mycompany.version: "1.0.0"
      com.mycompany.team: "backend"

    # --- USER ---
    user: "1000:1000"   # Run as non-root user (security best practice)

    # --- LOGGING ---
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # ── SERVICE: POSTGRES ────────────────────────────────────
  postgres:
    image: postgres:15-alpine
    container_name: myapp-postgres

    environment:
      POSTGRES_USER: ${DB_USER:-postgres}      # default value after :-
      POSTGRES_PASSWORD: ${DB_PASSWORD:-secret}
      POSTGRES_DB: ${DB_NAME:-myapp}
      PGDATA: /var/lib/postgresql/data/pgdata  # Avoid permission issues

    volumes:
      - pgdata:/var/lib/postgresql/data        # Persist DB across restarts
      - ./init-scripts:/docker-entrypoint-initdb.d  # Auto-run on first start
        # Files here run in alphabetical order: 01_schema.sql, 02_seed.sql

    ports:
      - "5432:5432"   # Expose for local DB clients (TablePlus, DBeaver)
                      # REMOVE this in production ECS!

    networks:
      - backend

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-postgres} -d ${DB_NAME:-myapp}"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s

    restart: unless-stopped

  # ── SERVICE: REDIS ────────────────────────────────────────
  redis:
    image: redis:7-alpine
    container_name: myapp-redis

    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru

    volumes:
      - redisdata:/data

    ports:
      - "6379:6379"   # Remove in production

    networks:
      - backend

    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

    restart: unless-stopped

  # ── SERVICE: NGINX ────────────────────────────────────────
  nginx:
    image: nginx:alpine
    container_name: myapp-nginx

    ports:
      - "80:80"
      - "443:443"

    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro   # :ro = read-only
      - ./nginx/certs:/etc/nginx/certs:ro
      - ./frontend/dist:/usr/share/nginx/html:ro      # Static frontend files

    depends_on:
      api:
        condition: service_healthy

    networks:
      - frontend

    restart: unless-stopped

# ══════════════════════════════════════════════════════════════
# NETWORKS
# ══════════════════════════════════════════════════════════════
networks:
  frontend:
    driver: bridge
    # Only nginx and frontend-facing services

  backend:
    driver: bridge
    # Only internal services — postgres, redis, api
    # Postgres is NOT on frontend network — cannot be reached from nginx

  # Use an EXISTING external network (e.g., shared across compose files):
  # monitoring:
  #   external: true
  #   name: monitoring_default

# ══════════════════════════════════════════════════════════════
# VOLUMES
# ══════════════════════════════════════════════════════════════
volumes:
  pgdata:
    driver: local    # Default — stores on host filesystem
    # driver_opts:   # Custom location:
    #   type: none
    #   o: bind
    #   device: /mnt/fast-disk/pgdata

  redisdata:
    driver: local

  api_cache:
    driver: local

  # External volume (pre-existing, not managed by compose):
  # shared_uploads:
  #   external: true
  #   name: myapp_uploads
```

---

## 4. Real-World Project — Python API + PostgreSQL + Redis + Nginx

This is a complete, working example of what a Senior Dev and Senior DevOps engineer set up together.

### Project Structure

```
myapp/
├── docker-compose.yml          ← Main compose file
├── docker-compose.override.yml ← Dev overrides (auto-loaded)
├── docker-compose.prod.yml     ← Prod overrides (explicit -f flag)
├── .env                        ← Default env vars (commit to git, no secrets)
├── .env.local                  ← Local secrets (NEVER commit — in .gitignore)
├── .gitignore
│
├── api/                        ← Python FastAPI application
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── requirements-dev.txt
│   └── app/
│       ├── main.py
│       ├── models.py
│       └── routers/
│
├── nginx/
│   ├── nginx.conf
│   └── certs/                  ← Self-signed certs for local HTTPS
│
└── init-scripts/               ← DB initialization
    ├── 01_schema.sql
    └── 02_seed_dev.sql
```

### `docker-compose.yml` (Base — Shared Dev/Prod Config)

```yaml
# docker-compose.yml
name: myapp

services:
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    environment:
      DATABASE_URL: "postgresql://${DB_USER}:${DB_PASSWORD}@postgres:5432/${DB_NAME}"
      REDIS_URL: "redis://redis:6379/0"
      APP_ENV: ${APP_ENV:-development}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - backend
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s
    restart: unless-stopped

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redisdata:/data
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      api:
        condition: service_healthy
    networks:
      - frontend
      - backend
    restart: unless-stopped

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge

volumes:
  pgdata:
  redisdata:
```

### `docker-compose.override.yml` (Dev Only — Auto-Loaded)

```yaml
# docker-compose.override.yml
# This file is AUTOMATICALLY merged when you run `docker compose up`
# Use for developer-specific settings: hot reload, debug ports, bind mounts

services:
  api:
    build:
      target: development         # Use dev stage of multi-stage Dockerfile
    ports:
      - "8000:8000"               # Expose API directly for dev tools
    volumes:
      - ./api:/app                # Bind mount: code changes reflect instantly
      - /app/__pycache__          # Don't override cache dir
    environment:
      LOG_LEVEL: debug
      RELOAD: "true"
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

  postgres:
    ports:
      - "5432:5432"               # Expose for local DB clients

  redis:
    ports:
      - "6379:6379"               # Expose for Redis Insight / redis-cli
```

### `docker-compose.prod.yml` (Prod/ECS Testing — Explicit)

```yaml
# docker-compose.prod.yml
# Run with: docker compose -f docker-compose.yml -f docker-compose.prod.yml up

services:
  api:
    build:
      target: production          # Production stage — no dev deps
    environment:
      LOG_LEVEL: info
      APP_ENV: production
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 1G

  # In prod, no exposed ports for postgres/redis — only internal network
  postgres:
    ports: []                     # Override: remove port exposure

  redis:
    ports: []
```

### `.env` (Committed — Defaults Only)

```bash
# .env
APP_ENV=development
DB_USER=postgres
DB_NAME=myapp
# DB_PASSWORD intentionally left out — must come from .env.local
```

### `.env.local` (Never Committed — Secrets)

```bash
# .env.local
DB_PASSWORD=supersecretlocal123
SECRET_KEY=dev-secret-key-change-in-prod
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
```

### `.gitignore`

```
.env.local
.env.*.local
*.pem
*.key
certs/
```

---

## 5. The Developer Workflow — What the Sr. Dev Does

This is the daily workflow a Senior Developer uses when building features.

```
MORNING STARTUP:
─────────────────────────────────────────────────────
$ git pull origin main
$ docker compose up --build

# --build: rebuilds images if Dockerfile or requirements changed
# Without --build: uses cached images

MAKE CODE CHANGES:
─────────────────────────────────────────────────────
# Because of the bind mount (./api:/app) + --reload flag,
# code changes in ./api/ are INSTANTLY reflected in the container.
# No rebuild needed for Python file changes!

# If requirements.txt changes → need to rebuild:
$ docker compose up --build api

RUNNING TESTS:
─────────────────────────────────────────────────────
# Run tests INSIDE the container (same environment as prod)
$ docker compose exec api pytest tests/ -v

# Or one-off command:
$ docker compose run --rm api pytest tests/unit/ -v
# --rm: removes container after command completes

DATABASE MIGRATIONS:
─────────────────────────────────────────────────────
$ docker compose exec api alembic upgrade head
$ docker compose exec api alembic revision --autogenerate -m "add users table"

CHECKING LOGS:
─────────────────────────────────────────────────────
$ docker compose logs -f api        # Follow API logs only
$ docker compose logs -f            # Follow ALL service logs
$ docker compose logs --tail=50 api # Last 50 lines

END OF DAY:
─────────────────────────────────────────────────────
$ docker compose down               # Stop and remove containers
                                    # Volumes PRESERVED (data safe)

$ docker compose down -v            # Stop AND delete volumes (fresh start)
```

---

## 6. The DevOps Workflow — What You Do

As a DevOps engineer, your focus is on **image quality, build reproducibility, environment parity, and ECS readiness**.

### Your Core Tasks

```
1. Own the Dockerfile — multi-stage, optimized, secure
2. Own the docker-compose.yml structure and file organization
3. Validate images work identically locally and in ECS
4. Manage secrets strategy (local .env → ECS Secrets Manager)
5. Set up CI/CD pipeline that mirrors the compose workflow
6. Define resource limits that match ECS task definition values
```

### Inspecting and Validating

```bash
# ── VALIDATE COMPOSE FILE ─────────────────────────────────
docker compose config
# Outputs the fully resolved compose file after variable substitution
# Use this to catch issues before running

docker compose config --quiet  # Just validate, exit 0 if OK

# ── SEE WHAT WILL BE CREATED ──────────────────────────────
docker compose ps              # Running services
docker compose images          # Images used by services

# ── INSPECT A RUNNING CONTAINER ───────────────────────────
docker compose exec api env    # Show all environment variables
docker inspect myapp-api-1     # Full container config (JSON)
docker stats                   # Live CPU/memory per container

# ── INSPECT THE NETWORK ───────────────────────────────────
docker network ls
docker network inspect myapp_backend
# Shows which containers are connected and their IPs

# ── INSPECT VOLUMES ───────────────────────────────────────
docker volume ls
docker volume inspect myapp_pgdata
# Shows where data is stored on the host
```

### Scaling Services (Testing ECS-Like Behavior)

```bash
# Run 3 instances of the API (simulates ECS desired count = 3)
docker compose up --scale api=3

# BUT: container_name prevents scaling (only 1 container can have a name)
# Remove container_name from services you want to scale
# Nginx handles load balancing across all 3 instances automatically
```

### Rebuilding Cleanly

```bash
# Rebuild ONE service without touching others
docker compose up --build api

# Rebuild ALL, no cache (use when Dockerfile changes aren't picked up)
docker compose build --no-cache
docker compose up

# Pull latest base images first
docker compose build --pull
```

### Simulating Production Locally

```bash
# Use the prod compose file to test EXACTLY what ECS will run
docker compose \
  -f docker-compose.yml \
  -f docker-compose.prod.yml \
  --env-file .env.staging \
  up --build

# This catches:
# - Missing env vars in prod config
# - Production image size/startup issues
# - Health check failures with prod settings
```

---

## 7. Networking Deep Dive — How Containers Talk

This is the most important concept to understand for debugging.

### DNS-Based Service Discovery

```
When postgres is defined as a service in compose,
EVERY container on the SAME network can reach it at:

  hostname: postgres   ← the service name IS the hostname
  port:     5432

This is why your API's DATABASE_URL uses "postgres" not "localhost":
  DATABASE_URL=postgresql://user:pass@postgres:5432/myapp
                                       ^^^^^^^^
                                       service name = hostname

How it works:
  Docker's embedded DNS server (127.0.0.11) resolves service names
  to container IPs dynamically.
```

### Network Isolation — Why It Matters

```
Your compose setup has TWO networks:

  frontend network:  nginx ←→ api
  backend network:   api ←→ postgres ←→ redis

  ┌──────────────────────────────────────────────────────┐
  │  INTERNET                                            │
  │     │                                                │
  │     ▼ port 80                                        │
  │  ┌────────┐                                          │
  │  │ nginx  │  ← on BOTH networks                     │
  │  └────┬───┘                                          │
  │       │   frontend network                           │
  │  ┌────▼───┐                                          │
  │  │  api   │  ← on BOTH networks                     │
  │  └────┬───┘                                          │
  │       │   backend network                            │
  │  ┌────▼───┐  ┌─────────┐                             │
  │  │postgres│  │  redis  │  ← backend ONLY             │
  │  └────────┘  └─────────┘                             │
  │                                                      │
  │  postgres CANNOT be reached from nginx               │
  │  (different networks = network isolation)            │
  └──────────────────────────────────────────────────────┘
```

### Port Mapping vs Internal Communication

```
INTERNAL (container-to-container, uses container port):
  api → postgres:5432       ✅ uses container port directly
  api → redis:6379          ✅ no host port needed

EXTERNAL (host → container, uses host port):
  browser → localhost:80    ✅ mapped to nginx:80
  DBeaver → localhost:5432  ✅ mapped to postgres:5432

RULE: ports: in compose.yml is ONLY for host ↔ container
      Container-to-container ALWAYS uses the internal port
```

### Debugging Network Issues

```bash
# Can the API container reach postgres?
docker compose exec api ping postgres
docker compose exec api nc -zv postgres 5432

# What DNS does the container see?
docker compose exec api nslookup postgres
docker compose exec api cat /etc/resolv.conf

# Is the container on the right networks?
docker inspect myapp-api-1 | grep -A 30 '"Networks"'

# Trace the full network path
docker compose exec api curl -v http://postgres:5432
```

---

## 8. Volumes & Data Persistence

```
TYPE 1: Named Volumes (Managed by Docker)
─────────────────────────────────────────
volumes:
  pgdata:              ← Docker manages location
                         Lives in /var/lib/docker/volumes/myapp_pgdata/

USE FOR: Databases, persistent data that outlives containers
SURVIVES: docker compose down ✅
DELETED BY: docker compose down -v ❌

TYPE 2: Bind Mounts (Host Path)
────────────────────────────────
volumes:
  - ./api:/app         ← Exact host path mounted into container

USE FOR: Development code sync (hot reload)
         Nginx config files
         Init SQL scripts
SURVIVES: Always (it's just a folder on your machine)
RISK: Permissions issues, platform-specific paths

TYPE 3: Anonymous Volumes
──────────────────────────
volumes:
  - /app/node_modules  ← No host path = anonymous

USE FOR: Preventing bind mount from overriding specific dirs
         (node_modules, __pycache__ should NOT come from host)
SURVIVES: docker compose down ✅ (but no name → harder to manage)

TYPE 4: tmpfs (Memory)
───────────────────────
tmpfs:
  - /tmp               ← In-memory, not persisted

USE FOR: Temp files, test data that must not persist
SURVIVES: Nothing — gone when container stops
```

### Viewing and Managing Volumes

```bash
# List all volumes for this project
docker volume ls | grep myapp

# Inspect where data lives
docker volume inspect myapp_pgdata

# Backup a volume
docker run --rm \
  -v myapp_pgdata:/source:ro \
  -v $(pwd)/backup:/backup \
  alpine tar czf /backup/pgdata_backup.tar.gz -C /source .

# Restore
docker run --rm \
  -v myapp_pgdata:/target \
  -v $(pwd)/backup:/backup:ro \
  alpine tar xzf /backup/pgdata_backup.tar.gz -C /target
```

---

## 9. Environment Variables & Secrets Management

### The 4 Ways to Set Env Vars (Priority Order — Highest to Lowest)

```
1. Command line:      docker compose run -e KEY=value api
2. Shell env:         export KEY=value && docker compose up
3. compose.yml:       environment: KEY: value
4. env_file:          env_file: .env
```

### Variable Substitution in docker-compose.yml

```yaml
environment:
  # Simple substitution (fails if not set):
  DB_PASSWORD: ${DB_PASSWORD}

  # Default value (used if DB_PASSWORD not set):
  DB_PASSWORD: ${DB_PASSWORD:-defaultvalue}

  # Required — error if not set (no default):
  DB_PASSWORD: ${DB_PASSWORD:?DB_PASSWORD must be set!}

  # Literal $ sign (escape with $$):
  GREETING: "Hello $$USER"    # → "Hello $USER" (not substituted)
```

### The .env File Hierarchy

```bash
# 1. .env (committed to git — safe defaults, no secrets)
APP_ENV=development
DB_NAME=myapp
DB_USER=appuser
LOG_LEVEL=info

# 2. .env.local (gitignored — dev machine secrets)
DB_PASSWORD=my_local_password
SECRET_KEY=dev-only-secret

# 3. .env.staging (gitignored — used by DevOps for staging tests)
APP_ENV=staging
DB_PASSWORD=staging_password_from_vault
```

```bash
# Load specific env file:
docker compose --env-file .env.staging up
```

### Secrets Strategy: Local → ECS

```
LOCAL COMPOSE                        AWS ECS
─────────────────                    ──────────────────────────────
.env.local                      →    AWS Secrets Manager
  DB_PASSWORD=local123          →    /myapp/production/DB_PASSWORD
  SECRET_KEY=dev-key            →    /myapp/production/SECRET_KEY

In ECS Task Definition:
{
  "secrets": [
    {
      "name": "DB_PASSWORD",
      "valueFrom": "arn:aws:secretsmanager:us-east-1:123:secret:/myapp/prod/DB_PASSWORD"
    }
  ]
}

The container sees DB_PASSWORD as a regular environment variable.
Same code, same app config — only the source changes.
```

---

## 10. Health Checks — The Bridge to ECS

Health checks are the **most critical thing** to get right before moving to ECS. ECS uses the exact same health check logic to route traffic and replace unhealthy containers.

### Anatomy of a Health Check

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
  #      ^^^    ^^^     ^^^
  #      |      |       |
  #      |      |       curl flags: -f = fail on HTTP error codes
  #      |      exec directly (CMD) vs shell (CMD-SHELL)
  #      key field

  interval: 10s      # How often to check (ECS default: 30s)
  timeout: 5s        # Max wait for response (ECS default: 5s)
  retries: 3         # Failures before UNHEALTHY (ECS default: 3)
  start_period: 30s  # Grace period on startup (ECS default: 0s)
```

### CMD vs CMD-SHELL

```yaml
# CMD: exec the command directly (preferred — no shell injection risk)
test: ["CMD", "curl", "-f", "http://localhost:8000/health"]

# CMD-SHELL: run through /bin/sh (needed for shell features like &&, pipes)
test: ["CMD-SHELL", "pg_isready -U $$DB_USER && psql -U $$DB_USER -c 'SELECT 1'"]
```

### Health Check Examples by Service Type

```yaml
# Python FastAPI
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
  # Your /health endpoint should return 200 OK with {"status": "ok"}

# PostgreSQL
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]

# Redis
healthcheck:
  test: ["CMD", "redis-cli", "ping"]

# Node.js
healthcheck:
  test: ["CMD", "node", "-e", "require('http').get('http://localhost:3000/health', r => process.exit(r.statusCode === 200 ? 0 : 1))"]

# Custom script (most robust)
healthcheck:
  test: ["CMD", "/app/scripts/healthcheck.sh"]
```

### What a Good `/health` Endpoint Returns

```python
# app/routers/health.py
from fastapi import APIRouter
from sqlalchemy import text
from app.database import get_db

router = APIRouter()

@router.get("/health")
async def health_check(db=Depends(get_db)):
    try:
        # Check DB connectivity
        await db.execute(text("SELECT 1"))
        db_status = "ok"
    except Exception as e:
        db_status = f"error: {str(e)}"

    return {
        "status": "ok" if db_status == "ok" else "degraded",
        "database": db_status,
        "version": os.getenv("APP_VERSION", "unknown")
    }
    # HTTP 200 = healthy, anything else = unhealthy
```

### Monitoring Health Status

```bash
# Check health status of all containers
docker compose ps
# Shows: Name | Image | Command | State | Ports
# State will be: Up, Up (healthy), Up (unhealthy), Exit

# Detailed health check history
docker inspect --format='{{json .State.Health}}' myapp-api-1 | python3 -m json.tool

# Watch health changes live
watch -n2 'docker compose ps'
```

---

## 11. Multi-Stage Dockerfile — Dev vs Prod Images

This is how Sr. Developers and DevOps engineers ensure local and ECS images are equivalent.

```dockerfile
# api/Dockerfile
# Multi-stage build: dev image has hot-reload tools, prod is minimal

# ═══════════════════════════════════
# Stage 1: BASE — shared foundation
# ═══════════════════════════════════
FROM python:3.12-slim AS base

# Prevents Python from writing pyc files and buffering stdout
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Create non-root user (security best practice)
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# Install system dependencies (cached separately from Python deps)
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    libpq-dev \
  && rm -rf /var/lib/apt/lists/*

# ═══════════════════════════════════
# Stage 2: DEPS — Python dependencies
# ═══════════════════════════════════
FROM base AS deps

# Copy only requirements first (layer caching!)
# This layer only rebuilds when requirements.txt changes
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# ═══════════════════════════════════
# Stage 3: DEVELOPMENT
# ═══════════════════════════════════
FROM deps AS development

# Install dev-only tools
COPY requirements-dev.txt .
RUN pip install --no-cache-dir -r requirements-dev.txt

# Copy source (but will be overridden by bind mount in compose)
COPY --chown=appuser:appuser . .

USER appuser

EXPOSE 8000

# Hot reload via uvicorn --reload
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]

# ═══════════════════════════════════
# Stage 4: PRODUCTION
# ═══════════════════════════════════
FROM deps AS production

# Copy application code
COPY --chown=appuser:appuser . .

# Remove any dev artifacts
RUN find . -type f -name "*.pyc" -delete \
  && find . -type d -name "__pycache__" -delete

# Switch to non-root user
USER appuser

EXPOSE 8000

# Production: no --reload, use gunicorn for worker management
CMD ["gunicorn", "app.main:app", \
     "-w", "4", \
     "-k", "uvicorn.workers.UvicornWorker", \
     "--bind", "0.0.0.0:8000", \
     "--access-logfile", "-", \
     "--error-logfile", "-"]
```

### Image Size Comparison

```bash
# Build both stages
docker build --target development -t myapp-api:dev ./api
docker build --target production -t myapp-api:prod ./api

docker images myapp-api
# REPOSITORY    TAG    SIZE
# myapp-api     dev    890MB   ← includes dev tools, debugger
# myapp-api     prod   185MB   ← lean, only what's needed
```

---

## 12. Local → ECS: The Translation Map

This is the exact mapping between your `docker-compose.yml` and an **ECS Task Definition**.

```
docker-compose.yml                 ECS Task Definition
──────────────────────────────     ──────────────────────────────────────
services:                      →   containerDefinitions:
  api:                         →     - name: api

  image: my-ecr.../api:latest  →       image: my-ecr.../api:latest

  ports:                       →       portMappings:
    - "8000:8000"              →         - containerPort: 8000
                                           hostPort: 8000

  environment:                 →       environment:
    APP_ENV: production        →         - name: APP_ENV
                                           value: production

  env secrets → .env.local     →       secrets:
                                         - name: DB_PASSWORD
                                           valueFrom: arn:aws:secretsmanager:...

  healthcheck:                 →       healthCheck:
    test: [CMD, curl, ...]     →         command: [CMD, curl, -f, http://...]
    interval: 10s              →         interval: 10
    timeout: 5s                →         timeout: 5
    retries: 3                 →         retries: 3
    start_period: 30s          →         startPeriod: 30

  deploy:                      →       cpu: 512        (in task definition)
    resources:                 →       memory: 1024    (in task definition)
      limits:                  →
        cpus: "0.5"  (=512)    →
        memory: 1G   (=1024)   →

  volumes:                     →       mountPoints:
    - ./api:/app               →         (no bind mounts in ECS — use EFS)
    - pgdata:/data             →         - sourceVolume: pgdata
                                           containerPath: /data

  networks:                    →       (ECS uses VPC subnets + security groups)
    - backend                  →       (awsvpcConfiguration in service def)

  logging:                     →       logConfiguration:
    driver: json-file          →         logDriver: awslogs
                               →         options:
                               →           awslogs-group: /ecs/myapp
                               →           awslogs-region: us-east-1
```

### Generating an ECS Task Definition from Compose

```bash
# Install the ECS Compose extension
docker compose convert  # Shows resolved compose config

# Use AWS Copilot (recommended) — converts compose to ECS automatically
copilot init
copilot deploy

# OR manual conversion: use this as your template
```

### Key Differences: Compose vs ECS

```
┌────────────────────┬──────────────────────┬────────────────────────────┐
│ Concern            │ Docker Compose       │ AWS ECS                    │
├────────────────────┼──────────────────────┼────────────────────────────┤
│ Service discovery  │ DNS by service name  │ AWS Cloud Map / App Mesh   │
│ Networking         │ Bridge networks      │ VPC, subnets, SGs          │
│ Secrets            │ .env.local file      │ Secrets Manager / SSM      │
│ Volumes            │ Named volumes        │ EFS (persistent) / ephemeral│
│ Scaling            │ --scale api=3        │ Service desired count       │
│ Health checks      │ healthcheck:         │ Same format in task def     │
│ Load balancing     │ Nginx container      │ ALB Target Group            │
│ Logging            │ docker logs / files  │ CloudWatch Logs (awslogs)  │
│ Image registry     │ Local Docker cache   │ ECR                        │
│ DB (Postgres)      │ Service in compose   │ RDS (separate service)     │
│ Cache (Redis)      │ Service in compose   │ ElastiCache                │
└────────────────────┴──────────────────────┴────────────────────────────┘
```

> **Key insight:** In ECS, Postgres and Redis are NOT containers in your task — they're managed AWS services (RDS + ElastiCache). Docker Compose lets you simulate the full stack locally without paying for those services.

---

## 13. Common Failures & Debugging

### Failure 1: Container exits immediately

```bash
# Symptom: service exits with code 1 right after starting
docker compose ps
# api  Exit 1

# Debug:
docker compose logs api          # Check the error output
docker compose run --rm api sh   # Run a shell in the exact same image
```

**Common causes:**
- Missing environment variable → app crashes on startup
- Wrong command/entrypoint
- Port already in use on host
- Missing file that's expected at container start

### Failure 2: "Connection refused" between services

```bash
# Symptom:
# api can't connect to postgres:5432

# Check 1: Is postgres actually healthy?
docker compose ps postgres
# If not healthy — it's not ready yet

# Check 2: Are they on the same network?
docker inspect myapp-api-1 | grep -A5 '"Networks"'
docker inspect myapp-postgres-1 | grep -A5 '"Networks"'
# Must share at least one network name

# Check 3: DNS resolution
docker compose exec api nslookup postgres
# Should return an IP

# Check 4: Wait for depends_on (add health check to postgres)
```

### Failure 3: "depends_on doesn't wait long enough"

```bash
# Symptom: api starts before postgres is ready
# Root cause: depends_on: service_started only waits for container start,
#             NOT for the service inside to be ready

# FIX: Use condition: service_healthy + add healthcheck to postgres
depends_on:
  postgres:
    condition: service_healthy    # ← this waits for healthcheck to pass

# Also add a retry in your app:
# Use a library like tenacity (Python) to retry DB connection on startup
```

### Failure 4: Volume permission errors

```bash
# Symptom: Permission denied writing to /app or /data
# docker compose logs api → "Permission denied: '/app/logs/app.log'"

# Root cause: Container runs as UID 1000 but volume owned by root

# Fix 1: Set correct ownership in Dockerfile
RUN chown -R appuser:appuser /app

# Fix 2: Set user in compose
user: "1000:1000"

# Fix 3: Fix ownership on the host
sudo chown -R 1000:1000 ./api/logs
```

### Failure 5: "Port already in use"

```bash
# Symptom: Error: Bind for 0.0.0.0:5432 failed: port is already allocated

# Find what's using it:
lsof -i :5432          # macOS/Linux
netstat -ano | findstr :5432  # Windows

# Stop the local postgres service:
sudo service postgresql stop   # Linux
brew services stop postgresql  # macOS

# OR change the host port in compose:
ports:
  - "5433:5432"    # Use 5433 on host instead
```

### Failure 6: Build cache serving stale code

```bash
# Symptom: Code changes not reflected despite compose up --build

# Force no-cache rebuild:
docker compose build --no-cache api
docker compose up api

# Or nuclear option:
docker compose down
docker system prune -f
docker compose up --build
```

### The Ultimate Debug Toolkit

```bash
# Show full resolved config (after variable substitution):
docker compose config

# Show running processes INSIDE a container:
docker compose exec api ps aux

# Resource usage:
docker stats

# Full container details:
docker inspect myapp-api-1

# Network connectivity test:
docker compose exec api curl -v http://postgres:5432
docker compose exec api env | sort    # All env vars, sorted

# Interactive debug session:
docker compose exec api /bin/bash
docker compose exec api /bin/sh       # Alpine containers (no bash)

# Run one-off command in same environment as service:
docker compose run --rm api python -c "from app.database import engine; print(engine.url)"
```

---

## 14. Production Checklist Before Pushing to ECS

Before your Sr. DevOps engineer approves the ECS deployment, verify ALL of these locally:

```
IMAGE QUALITY
─────────────────────────────────────────────────────────────
[ ] docker compose build --no-cache --target production
    → Build succeeds with no errors
[ ] Image uses non-root user (USER directive in Dockerfile)
[ ] No secrets baked into the image (docker history myapp-api:prod)
[ ] Image size is reasonable (< 500MB for most apps)
[ ] Multi-stage build used (dev deps not in prod image)

HEALTH CHECKS
─────────────────────────────────────────────────────────────
[ ] Every service has a healthcheck: defined
[ ] docker compose ps shows "(healthy)" for all services after startup
[ ] Health endpoint returns 200 on happy path
[ ] Health endpoint returns 5xx when dependency (DB) is down

ENVIRONMENT & SECRETS
─────────────────────────────────────────────────────────────
[ ] No secrets hardcoded in docker-compose.yml or Dockerfile
[ ] All secrets reference ${VAR} — sourced from .env.local (local)
[ ] ECS task definition will use Secrets Manager for same vars
[ ] docker compose config shows no literal secret values in output

NETWORKING
─────────────────────────────────────────────────────────────
[ ] Service-to-service comms use service names (not localhost)
[ ] Postgres/Redis ports NOT exposed in prod compose file
[ ] Only nginx/API ports exposed to host

RESOURCE LIMITS
─────────────────────────────────────────────────────────────
[ ] deploy.resources.limits set in compose (matches ECS task def)
[ ] Memory limit tested — app doesn't OOM under load
[ ] CPU limit tested — app responds acceptably under limit

STARTUP & SHUTDOWN
─────────────────────────────────────────────────────────────
[ ] App handles SIGTERM gracefully (drains connections before exit)
[ ] depends_on uses condition: service_healthy for critical deps
[ ] App retries database connection on startup (not just fails fast)
[ ] Startup time from zero < ECS health check start_period

LOGGING
─────────────────────────────────────────────────────────────
[ ] App logs to stdout/stderr (not files — ECS captures these)
[ ] Log format is JSON (structured — searchable in CloudWatch)
[ ] No sensitive data in logs (passwords, tokens, PII)

TEST END-TO-END
─────────────────────────────────────────────────────────────
[ ] docker compose -f docker-compose.yml -f docker-compose.prod.yml up
    → Full stack starts with production image
[ ] API returns 200 on all critical endpoints
[ ] DB migrations run successfully
[ ] Auth flow works end-to-end
```

---

## 15. Quick Reference Cheat Sheet

### Essential Commands

```bash
# ── STARTING ──────────────────────────────────────────────────
docker compose up                 # Start all services
docker compose up -d              # Detached (background)
docker compose up --build         # Rebuild changed images first
docker compose up api             # Start only the 'api' service
docker compose up --scale api=3   # Run 3 api instances

# ── STOPPING ──────────────────────────────────────────────────
docker compose stop               # Stop containers (keep them)
docker compose down               # Stop & remove containers + networks
docker compose down -v            # + remove volumes (wipes DB data!)
docker compose down --rmi all     # + remove images

# ── LOGS ──────────────────────────────────────────────────────
docker compose logs               # All logs
docker compose logs -f            # Follow (stream)
docker compose logs -f api        # Follow single service
docker compose logs --tail=100    # Last 100 lines

# ── EXEC / RUN ────────────────────────────────────────────────
docker compose exec api bash      # Shell in RUNNING container
docker compose exec api env       # List env vars
docker compose run --rm api bash  # Start NEW container with shell
docker compose run --rm api pytest # Run tests in fresh container

# ── STATUS & INSPECT ──────────────────────────────────────────
docker compose ps                 # Service status + health
docker compose images             # Images used
docker compose config             # Show resolved config
docker stats                      # Live resource usage

# ── BUILD ─────────────────────────────────────────────────────
docker compose build              # Build all services
docker compose build api          # Build one service
docker compose build --no-cache   # Force full rebuild
docker compose build --pull       # Pull latest base images

# ── CLEANUP ───────────────────────────────────────────────────
docker system prune               # Remove unused containers/images
docker system prune -a            # + remove all unused images
docker volume prune               # Remove unused volumes
```

### The `docker-compose.yml` Skeleton

```yaml
name: myapp

services:
  myservice:
    build: ./myservice    # OR image: myimage:tag
    ports:
      - "HOST:CONTAINER"
    environment:
      KEY: ${VALUE:-default}
    env_file: .env
    volumes:
      - ./local:/container       # bind mount
      - namedvol:/data           # named volume
    depends_on:
      other:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:PORT/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s
    networks:
      - mynetwork
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M

networks:
  mynetwork:
    driver: bridge

volumes:
  namedvol:
```

### File Naming Convention for Your Repo

```
docker-compose.yml              ← Base config (always loaded)
docker-compose.override.yml     ← Dev overrides (auto-loaded with base)
docker-compose.prod.yml         ← Prod simulation (explicit -f flag)
docker-compose.test.yml         ← CI test environment

.env                            ← Default non-secret values (committed)
.env.local                      ← Dev secrets (gitignored)
.env.staging                    ← Staging secrets (gitignored)
```

---

## 📁 Where This Fits in the Repo

```
devops-field-guide/
├── python/
├── eks/
├── ecs/
│   └── what-devops-engineers-should-understand-about-ecs.md
├── docker/
│   └── docker-compose-local-to-ecs-the-complete-devops-playbook.md  ← THIS FILE
├── ckad/
├── aws/
├── README.md
└── LICENSE
```

> **Best home:** `docker/` — this is a Docker Compose deep dive, with the ECS translation as the destination. It bridges `docker/` and `ecs/` perfectly.

---

*Part of the [devops-field-guide](../) series · Built for working engineers, not textbooks.*  

⭐ Star the repo if this helped you ship confidently from localhost to ECS.
