# GitLab CI/CD: End-to-End 101 Guide
### From First Pipeline to Production-Grade ECS & EKS Deployments

> **Author's Note:** This guide is written for all levels — whether you're copying code to a server today or architecting multi-environment EKS deployments tomorrow. Sections are labeled `[Beginner]`, `[Intermediate]`, and `[Advanced]` so you can jump to where you are and grow from there.

---

## Table of Contents

1. [What Is CI/CD and Why It Matters](#1-what-is-cicd-and-why-it-matters)
2. [GitLab CI/CD Core Concepts](#2-gitlab-cicd-core-concepts)
3. [Your First Pipeline — Getting Started](#3-your-first-pipeline--getting-started)
4. [Pipeline Anatomy Deep Dive](#4-pipeline-anatomy-deep-dive)
5. [Variables, Secrets & Environments](#5-variables-secrets--environments)
6. [Artifacts, Caching & Optimization](#6-artifacts-caching--optimization)
7. [Docker & Container Builds in GitLab](#7-docker--container-builds-in-gitlab)
8. [Deploying to AWS ECS — End to End](#8-deploying-to-aws-ecs--end-to-end)
9. [Deploying to AWS EKS — End to End](#9-deploying-to-aws-eks--end-to-end)
10. [Multi-Environment Pipelines (Dev → Staging → Prod)](#10-multi-environment-pipelines-dev--staging--prod)
11. [Security Best Practices](#11-security-best-practices)
12. [Pipeline Best Practices by Level](#12-pipeline-best-practices-by-level)
13. [Monitoring, Observability & Troubleshooting](#13-monitoring-observability--troubleshooting)
14. [GitLab Runners — Self-Hosted vs Shared](#14-gitlab-runners--self-hosted-vs-shared)
15. [Quick Reference Cheat Sheet](#15-quick-reference-cheat-sheet)

---

## 1. What Is CI/CD and Why It Matters

### The Problem CI/CD Solves

Without CI/CD, deploying software looks like this:

```
Developer writes code → Manually tests locally → SSHs into server →
Copies files manually → Hopes nothing breaks → Fixes things at 2 AM
```

With CI/CD, it looks like this:

```
Developer pushes code → Automated tests run → Docker image built →
Deployed to staging → Approved → Deployed to production → Sleep soundly
```

### CI vs CD

| Term | Full Name | What It Does |
|------|-----------|--------------|
| **CI** | Continuous Integration | Automatically builds and tests code on every push |
| **CD** | Continuous Delivery | Automatically deploys tested code to an environment (with approval gates) |
| **CD** | Continuous Deployment | Automatically deploys to production without manual approval |

### Why GitLab?

GitLab is a **complete DevOps platform** — not just a Git host. It includes:

- Source code management (like GitHub)
- Built-in CI/CD pipelines (no Jenkins needed)
- Container registry (push Docker images right inside GitLab)
- Kubernetes integration
- Security scanning (SAST, DAST, dependency scanning)
- Environment dashboards and deployment tracking

---

## 2. GitLab CI/CD Core Concepts

`[Beginner]`

Understanding these terms is essential before writing a single line of YAML.

### The `.gitlab-ci.yml` File

This is your entire pipeline. It lives in the **root of your repository**. GitLab automatically detects it and runs it on every push (by default).

```
your-repo/
├── .gitlab-ci.yml        ← Your entire pipeline lives here
├── app/
│   └── main.py
├── Dockerfile
└── requirements.txt
```

### Runners

A **Runner** is the machine that executes your pipeline jobs. Think of it as the worker bee.

- **Shared Runners**: GitLab's own machines (free tier available, limited minutes)
- **Self-Hosted Runners**: Your own EC2 instance or Ubuntu server running the GitLab Runner agent

> **Your situation**: You currently use GitLab to push code to an Ubuntu server. A self-hosted runner ON that Ubuntu server would let GitLab automatically deploy your code — no more manual copying.

### Stages

Stages define the **order** in which jobs run. All jobs in the same stage run **in parallel**.

```yaml
stages:
  - build      # First: compile/build
  - test       # Second: run tests
  - package    # Third: create Docker image
  - deploy     # Fourth: push to server/ECS/EKS
```

### Jobs

A **Job** is a single unit of work with a name, a stage, and a script to run.

```yaml
run-unit-tests:        # Job name (you pick this)
  stage: test          # Which stage it belongs to
  script:
    - pip install -r requirements.txt
    - pytest tests/
```

### Pipelines

A **Pipeline** is the complete run of all stages and jobs triggered by a push or merge request. You can see every pipeline's status in GitLab under `CI/CD → Pipelines`.

---

## 3. Your First Pipeline — Getting Started

`[Beginner]`

### Step 1: Create `.gitlab-ci.yml`

Create this file in your repository root. This is the simplest possible pipeline:

```yaml
# .gitlab-ci.yml

stages:
  - test
  - deploy

# Job 1: Run tests
run-tests:
  stage: test
  image: python:3.11-slim          # Use a Python Docker image as the environment
  script:
    - pip install -r requirements.txt
    - echo "Running tests..."
    - python -m pytest tests/ -v
  only:
    - main                          # Only run on pushes to main branch
    - merge_requests                # And on merge requests

# Job 2: Deploy to your Ubuntu server
deploy-to-server:
  stage: deploy
  script:
    - echo "Deploying application..."
    - rsync -avz --delete ./app/ user@your-server:/var/www/myapp/
  only:
    - main                          # Only deploy when code merges to main
  environment:
    name: production
```

### Step 2: Commit and Push

```bash
git add .gitlab-ci.yml
git commit -m "Add initial CI/CD pipeline"
git push origin main
```

### Step 3: Watch It Run

Go to your GitLab project → **CI/CD → Pipelines**. You'll see your pipeline running in real time.

### Step 4: Set Up a Self-Hosted Runner on Your Ubuntu Server

Run these commands on your Ubuntu server to install the GitLab Runner:

```bash
# Download and install GitLab Runner
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install gitlab-runner

# Register the runner with your GitLab project
# You'll need your project's registration token from:
# GitLab Project → Settings → CI/CD → Runners → New project runner
sudo gitlab-runner register
```

During registration, you'll be asked:
- **GitLab URL**: `https://gitlab.com` (or your self-hosted GitLab URL)
- **Token**: From your project settings
- **Description**: `ubuntu-production-runner`
- **Tags**: `production, ubuntu`
- **Executor**: `shell` (simplest for your current setup)

---

## 4. Pipeline Anatomy Deep Dive

`[Beginner → Intermediate]`

### Full YAML Structure

```yaml
# ── Global defaults applied to ALL jobs ──────────────────────────────
default:
  image: python:3.11-slim
  before_script:
    - pip install -r requirements.txt

# ── Variables available to all jobs ──────────────────────────────────
variables:
  APP_NAME: "my-application"
  DOCKER_DRIVER: overlay2

# ── Stage order definition ────────────────────────────────────────────
stages:
  - build
  - test
  - package
  - deploy

# ── Reusable template (anchors) ───────────────────────────────────────
.python-base: &python-base
  image: python:3.11-slim
  before_script:
    - pip install -r requirements.txt

# ── Job definitions ───────────────────────────────────────────────────
build-app:
  <<: *python-base                  # Merge the template above
  stage: build
  script:
    - python setup.py build
  artifacts:
    paths:
      - dist/                       # Save build output for next stages
    expire_in: 1 hour

unit-tests:
  <<: *python-base
  stage: test
  script:
    - pytest tests/unit/ --cov=app --cov-report=xml
  coverage: '/TOTAL.*\s+(\d+%)$/'   # Parse coverage % from output
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml

integration-tests:
  <<: *python-base
  stage: test
  script:
    - pytest tests/integration/
  allow_failure: false              # This job MUST pass

deploy-staging:
  stage: deploy
  script:
    - ./scripts/deploy.sh staging
  environment:
    name: staging
    url: https://staging.myapp.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

deploy-production:
  stage: deploy
  script:
    - ./scripts/deploy.sh production
  environment:
    name: production
    url: https://myapp.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  when: manual                      # Require a human to click "Deploy"
```

### Key Job Keywords

| Keyword | Purpose | Example |
|---------|---------|---------|
| `image` | Docker image to use as the job environment | `image: node:18` |
| `stage` | Which stage this job belongs to | `stage: test` |
| `script` | Shell commands to run | `script: - pytest` |
| `before_script` | Commands to run before `script` | Install dependencies |
| `after_script` | Commands to run even if job fails | Upload logs |
| `artifacts` | Files to save and pass to next stage | Build outputs |
| `cache` | Files to reuse across pipeline runs | `node_modules/` |
| `rules` | Conditions for when job runs | Run only on `main` |
| `when` | `always`, `on_success`, `on_failure`, `manual` | Require approval |
| `needs` | Skip stage order — run job when dependency passes | DAG pipelines |
| `allow_failure` | Don't fail pipeline if this job fails | Linting warnings |
| `timeout` | Max job duration | `timeout: 30 minutes` |
| `tags` | Which runner to use | `tags: [production]` |

### Using `rules` vs `only`/`except`

`rules` is the modern, more powerful approach. Prefer it over `only`/`except`.

```yaml
# OLD way (still works, but limited)
deploy:
  only:
    - main
  except:
    - schedules

# NEW way (more powerful)
deploy:
  rules:
    - if: '$CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE != "schedule"'
      when: on_success
    - when: never                   # Default: don't run
```

### Pre-defined CI Variables

GitLab automatically injects these into every job:

```bash
$CI_COMMIT_BRANCH          # Branch name: "main", "develop"
$CI_COMMIT_SHA             # Full commit hash
$CI_COMMIT_SHORT_SHA       # Short hash: "a1b2c3d4"
$CI_COMMIT_TAG             # Tag name (if triggered by a tag)
$CI_PIPELINE_ID            # Unique pipeline ID
$CI_JOB_NAME               # Current job name
$CI_PROJECT_NAME           # Repository name
$CI_REGISTRY               # GitLab container registry URL
$CI_REGISTRY_IMAGE         # Full image path in registry
$CI_REGISTRY_USER          # Registry auth username
$CI_REGISTRY_PASSWORD      # Registry auth password
$CI_ENVIRONMENT_NAME       # Deployment environment name
```

---

## 5. Variables, Secrets & Environments

`[Intermediate]`

### Variable Types

```
GitLab Variable Hierarchy (highest precedence wins):
─────────────────────────────────────────────────
1. Job-level variables (in .gitlab-ci.yml)
2. Pipeline-level variables (triggered via API)
3. Project CI/CD variables (Settings → CI/CD → Variables)
4. Group CI/CD variables (inherited by all projects in group)
5. Instance-level variables (admin only)
```

### Setting Secrets in GitLab UI

**Never put secrets in `.gitlab-ci.yml`.** Use GitLab's built-in secret management:

1. Go to **Project → Settings → CI/CD → Variables**
2. Click **Add variable**
3. Set:
   - **Key**: `AWS_ACCESS_KEY_ID`
   - **Value**: Your actual key
   - **Type**: Variable
   - ✅ **Masked**: Hides value in logs
   - ✅ **Protected**: Only available in protected branches/tags

### Using Variables in Your Pipeline

```yaml
variables:
  # Non-secret variables — fine to define in YAML
  APP_PORT: "8080"
  DOCKER_REGISTRY: "registry.gitlab.com/mygroup/myapp"

deploy-to-ecs:
  stage: deploy
  script:
    # Secrets ($AWS_ACCESS_KEY_ID etc.) come from GitLab UI variables
    - aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
    - aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
    - aws configure set default.region $AWS_DEFAULT_REGION
    - aws ecs update-service --cluster $ECS_CLUSTER --service $ECS_SERVICE --force-new-deployment
```

### Environment-Scoped Variables

You can scope variables to specific environments:

```
Variable: DATABASE_URL
├── Scope: staging  → postgres://staging-db:5432/app
└── Scope: production → postgres://prod-db:5432/app
```

When your job sets `environment: name: staging`, GitLab automatically injects the staging-scoped `DATABASE_URL`.

---

## 6. Artifacts, Caching & Optimization

`[Intermediate]`

### Artifacts — Pass Data Between Stages

Artifacts are files that jobs upload to GitLab and can be downloaded by later jobs or humans.

```yaml
build:
  stage: build
  script:
    - python setup.py bdist_wheel
  artifacts:
    name: "$CI_COMMIT_SHORT_SHA-build"
    paths:
      - dist/*.whl                  # Save the wheel file
    expire_in: 7 days               # Auto-delete after 7 days
    when: on_success                # Only save if job succeeded

test:
  stage: test
  needs:
    - job: build
      artifacts: true               # Download artifacts from build job
  script:
    - pip install dist/*.whl
    - pytest
  artifacts:
    reports:
      junit: test-results.xml       # GitLab parses this and shows test results in MR
```

### Caching — Speed Up Repeated Runs

Cache reuses files across **different pipeline runs** to avoid re-downloading dependencies.

```yaml
# Cache Python dependencies between pipeline runs
default:
  cache:
    key:
      files:
        - requirements.txt          # Invalidate cache when this file changes
    paths:
      - .pip-cache/
    policy: pull-push               # Download at start, upload at end

variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.pip-cache"

install-deps:
  script:
    - pip install --cache-dir .pip-cache -r requirements.txt
```

### Cache vs Artifacts

| Feature | Cache | Artifacts |
|---------|-------|-----------|
| **Purpose** | Speed up pipelines | Pass data between jobs |
| **Scope** | Same runner, same branch (configurable) | Available to all runners |
| **Guaranteed?** | No (best-effort) | Yes |
| **Example** | `node_modules/`, pip cache | Built binaries, test reports |

### Pipeline Optimization Techniques

```yaml
# ── Technique 1: DAG Pipelines with `needs` ──────────────────────────
# Don't wait for entire stage — run as soon as dependencies are done
test-unit:
  stage: test
  needs: [build]          # Start as soon as build passes, not entire build stage

test-integration:
  stage: test
  needs: [build]          # Also starts immediately after build

deploy:
  stage: deploy
  needs:
    - test-unit
    - test-integration    # Only needs these two, not ALL test-stage jobs

# ── Technique 2: Parallel Jobs ───────────────────────────────────────
test-matrix:
  stage: test
  parallel:
    matrix:
      - PYTHON_VERSION: ["3.9", "3.10", "3.11"]
  image: python:$PYTHON_VERSION
  script:
    - pytest tests/

# ── Technique 3: Interruptible Pipelines ─────────────────────────────
# Cancel old pipeline when new commit is pushed to same branch
build:
  interruptible: true

# ── Technique 4: Conditional Jobs ────────────────────────────────────
# Skip expensive jobs unless relevant files changed
run-python-tests:
  rules:
    - changes:
        - "**/*.py"
        - requirements.txt
      when: on_success
    - when: never
```

---

## 7. Docker & Container Builds in GitLab

`[Intermediate]`

### Building Docker Images in CI

GitLab has a built-in **Container Registry** — you don't need Docker Hub.

```yaml
variables:
  # These are automatically set by GitLab
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  IMAGE_LATEST: $CI_REGISTRY_IMAGE:latest

stages:
  - build
  - test
  - push

build-docker-image:
  stage: build
  image: docker:24.0
  services:
    - docker:24.0-dind              # Docker-in-Docker daemon
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    # Build with commit SHA tag for traceability
    - docker build -t $IMAGE_TAG -t $IMAGE_LATEST .
    # Push both tags
    - docker push $IMAGE_TAG
    - docker push $IMAGE_LATEST
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

### Multi-Stage Dockerfile Best Practice

A well-structured Dockerfile for a Python app:

```dockerfile
# ── Stage 1: Build dependencies ───────────────────────────────────────
FROM python:3.11-slim AS builder

WORKDIR /build
COPY requirements.txt .

# Install dependencies into a virtual env for easy copying
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
RUN pip install --no-cache-dir -r requirements.txt

# ── Stage 2: Production image (small, no build tools) ─────────────────
FROM python:3.11-slim AS production

# Security: Don't run as root
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# Copy only the installed dependencies, not the build tools
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Copy application code
COPY --chown=appuser:appuser ./app .

USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

### Scanning Docker Images for Vulnerabilities

GitLab has built-in container scanning:

```yaml
include:
  - template: Security/Container-Scanning.gitlab-ci.yml   # Built-in template

container_scanning:
  variables:
    CS_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

---

## 8. Deploying to AWS ECS — End to End

`[Intermediate → Advanced]`

### Architecture Overview

```
GitLab Push
    │
    ▼
GitLab Pipeline
    ├── Build & Test Python App
    ├── Build Docker Image
    ├── Push to ECR (AWS Container Registry)
    └── Update ECS Service (rolling deployment)
                │
                ▼
        ECS Cluster
        ├── Service: my-app
        │   ├── Task Definition (points to new image)
        │   └── Tasks running on EC2 or Fargate
        └── Load Balancer (ALB)
```

### Prerequisites in AWS

Before writing the pipeline, set up in AWS:

1. **ECR Repository**: `aws ecr create-repository --repository-name my-app`
2. **ECS Cluster**: Via Console or Terraform
3. **Task Definition**: Defines CPU/memory, image, environment variables, port mappings
4. **ECS Service**: Running tasks with desired count
5. **IAM User for GitLab**: With these permissions:
   - `ecr:GetAuthorizationToken`
   - `ecr:BatchCheckLayerAvailability`
   - `ecr:PutImage`
   - `ecr:InitiateLayerUpload`
   - `ecr:UploadLayerPart`
   - `ecr:CompleteLayerUpload`
   - `ecs:UpdateService`
   - `ecs:DescribeServices`
   - `ecs:RegisterTaskDefinition`
   - `ecs:DescribeTaskDefinition`
   - `iam:PassRole`

### GitLab Variables to Set (Project → Settings → CI/CD → Variables)

```
AWS_ACCESS_KEY_ID          (masked, protected)
AWS_SECRET_ACCESS_KEY      (masked, protected)
AWS_DEFAULT_REGION         (e.g., us-east-1)
ECR_REGISTRY               (your-account-id.dkr.ecr.us-east-1.amazonaws.com)
ECR_REPOSITORY             (my-app)
ECS_CLUSTER                (my-cluster)
ECS_SERVICE                (my-app-service)
ECS_CONTAINER_NAME         (my-app)           # Must match name in task definition
```

### Complete ECS Pipeline

```yaml
# .gitlab-ci.yml — ECS Deployment Pipeline

stages:
  - test
  - build
  - push
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  IMAGE_URI: $ECR_REGISTRY/$ECR_REPOSITORY

# ── 1. Test ────────────────────────────────────────────────────────────
unit-tests:
  stage: test
  image: python:3.11-slim
  script:
    - pip install -r requirements.txt
    - pytest tests/ -v --junitxml=test-results.xml
  artifacts:
    reports:
      junit: test-results.xml
  coverage: '/TOTAL.*\s+(\d+%)$/'

# ── 2. Build Docker Image ──────────────────────────────────────────────
build-image:
  stage: build
  image: docker:24.0
  services:
    - docker:24.0-dind
  script:
    - docker build
        --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')
        --build-arg GIT_COMMIT=$CI_COMMIT_SHA
        -t $IMAGE_URI:$CI_COMMIT_SHORT_SHA
        -t $IMAGE_URI:latest
        .
    - docker save $IMAGE_URI:$CI_COMMIT_SHORT_SHA > image.tar
  artifacts:
    paths:
      - image.tar
    expire_in: 1 hour
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

# ── 3. Push to ECR ─────────────────────────────────────────────────────
push-to-ecr:
  stage: push
  image: docker:24.0
  services:
    - docker:24.0-dind
  needs:
    - job: build-image
      artifacts: true
  before_script:
    - apk add --no-cache aws-cli
  script:
    # Authenticate Docker to ECR
    - aws ecr get-login-password --region $AWS_DEFAULT_REGION |
        docker login --username AWS --password-stdin $ECR_REGISTRY
    # Load saved image and push
    - docker load < image.tar
    - docker push $IMAGE_URI:$CI_COMMIT_SHORT_SHA
    - docker push $IMAGE_URI:latest
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

# ── 4. Deploy to ECS Staging ───────────────────────────────────────────
deploy-staging:
  stage: deploy
  image:
    name: amazon/aws-cli:latest
    entrypoint: [""]
  needs: [push-to-ecr]
  before_script:
    - aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
    - aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
    - aws configure set default.region $AWS_DEFAULT_REGION
  script:
    # Get current task definition
    - TASK_DEF=$(aws ecs describe-task-definition
        --task-definition $ECS_SERVICE
        --query taskDefinition)

    # Update image in task definition
    - NEW_TASK_DEF=$(echo $TASK_DEF | python3 -c "
        import json, sys
        td = json.load(sys.stdin)
        for c in td['containerDefinitions']:
            if c['name'] == '$ECS_CONTAINER_NAME':
                c['image'] = '$IMAGE_URI:$CI_COMMIT_SHORT_SHA'
        # Remove fields that can't be re-registered
        for key in ['taskDefinitionArn','revision','status','requiresAttributes',
                    'compatibilities','registeredAt','registeredBy']:
            td.pop(key, None)
        print(json.dumps(td))
      ")

    # Register the new task definition revision
    - NEW_TASK_DEF_ARN=$(echo $NEW_TASK_DEF |
        aws ecs register-task-definition --cli-input-json file:///dev/stdin |
        python3 -c "import json,sys; print(json.load(sys.stdin)['taskDefinition']['taskDefinitionArn'])")

    # Update the ECS service with the new task definition
    - aws ecs update-service
        --cluster $ECS_CLUSTER
        --service $ECS_SERVICE-staging
        --task-definition $NEW_TASK_DEF_ARN
        --force-new-deployment

    # Wait for deployment to complete (up to 10 minutes)
    - aws ecs wait services-stable
        --cluster $ECS_CLUSTER
        --services $ECS_SERVICE-staging
  environment:
    name: staging
    url: https://staging.myapp.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

# ── 5. Deploy to ECS Production (Manual Approval) ─────────────────────
deploy-production:
  stage: deploy
  image:
    name: amazon/aws-cli:latest
    entrypoint: [""]
  needs: [deploy-staging]
  before_script:
    - aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
    - aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
    - aws configure set default.region $AWS_DEFAULT_REGION
  script:
    # Update production service with the SAME image that passed staging
    - aws ecs update-service
        --cluster $ECS_CLUSTER
        --service $ECS_SERVICE-production
        --task-definition $ECS_SERVICE
        --force-new-deployment
    - aws ecs wait services-stable
        --cluster $ECS_CLUSTER
        --services $ECS_SERVICE-production
  environment:
    name: production
    url: https://myapp.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  when: manual                      # Requires human approval in GitLab UI
```

### ECS Rollback Strategy

```yaml
rollback-production:
  stage: deploy
  image:
    name: amazon/aws-cli:latest
    entrypoint: [""]
  script:
    # Get the previous task definition revision
    - CURRENT_REVISION=$(aws ecs describe-services
        --cluster $ECS_CLUSTER
        --services $ECS_SERVICE-production
        --query 'services[0].taskDefinition' --output text | grep -oE '[0-9]+$')
    - PREVIOUS_REVISION=$((CURRENT_REVISION - 1))
    - aws ecs update-service
        --cluster $ECS_CLUSTER
        --service $ECS_SERVICE-production
        --task-definition "$ECS_SERVICE:$PREVIOUS_REVISION"
        --force-new-deployment
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  when: manual
  environment:
    name: production
    action: prepare
```

---

## 9. Deploying to AWS EKS — End to End

`[Advanced]`

### Architecture Overview

```
GitLab Push
    │
    ▼
GitLab Pipeline
    ├── Test
    ├── Build & Push Docker image to ECR
    └── Deploy to EKS
            │  (kubectl apply or helm upgrade)
            ▼
    EKS Cluster
    ├── Namespace: staging
    │   └── Deployment → Pods → Service → Ingress
    └── Namespace: production
        └── Deployment → Pods → Service → Ingress
```

### Prerequisites

1. **EKS Cluster** running (via `eksctl` or Terraform)
2. **ECR Repository** for your images
3. **kubectl** and **helm** available in your pipeline
4. **IAM Role for GitLab** (or OIDC-based auth — preferred for security)
5. **kubeconfig** stored as a GitLab variable (base64 encoded)

### Kubernetes Manifests Structure

```
k8s/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── hpa.yaml              # Horizontal Pod Autoscaler
├── overlays/
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   └── patch-replicas.yaml
│   └── production/
│       ├── kustomization.yaml
│       └── patch-replicas.yaml
└── helm/
    └── my-app/
        ├── Chart.yaml
        ├── values.yaml
        ├── values-staging.yaml
        ├── values-production.yaml
        └── templates/
```

### Kubernetes Deployment Manifest

```yaml
# k8s/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
    version: "IMAGE_TAG"            # Replaced by pipeline
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0             # Zero-downtime deployment
  template:
    metadata:
      labels:
        app: my-app
        version: "IMAGE_TAG"
    spec:
      # Security context — don't run as root
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
        - name: my-app
          image: IMAGE_PLACEHOLDER   # Replaced by pipeline sed command
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          env:
            - name: APP_ENV
              valueFrom:
                configMapKeyRef:
                  name: my-app-config
                  key: APP_ENV
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: my-app-secrets
                  key: DATABASE_URL
```

### GitLab Variables for EKS

```
AWS_ACCESS_KEY_ID           (masked, protected)
AWS_SECRET_ACCESS_KEY       (masked, protected)
AWS_DEFAULT_REGION          us-east-1
ECR_REGISTRY                <account>.dkr.ecr.us-east-1.amazonaws.com
ECR_REPOSITORY              my-app
EKS_CLUSTER_NAME            my-eks-cluster
KUBECONFIG_BASE64           (base64 of your kubeconfig — masked, protected)
```

### Complete EKS Pipeline

```yaml
# .gitlab-ci.yml — EKS Deployment Pipeline

stages:
  - test
  - build
  - push
  - security-scan
  - deploy-staging
  - integration-test
  - deploy-production

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  IMAGE_URI: $ECR_REGISTRY/$ECR_REPOSITORY
  K8S_STAGING_NS: staging
  K8S_PROD_NS: production

# ── Reusable AWS auth setup ────────────────────────────────────────────
.aws-setup: &aws-setup
  before_script:
    - aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
    - aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
    - aws configure set default.region $AWS_DEFAULT_REGION

.kubectl-setup: &kubectl-setup
  before_script:
    - aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
    - aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
    - aws configure set default.region $AWS_DEFAULT_REGION
    # Decode and configure kubeconfig
    - echo $KUBECONFIG_BASE64 | base64 -d > /tmp/kubeconfig
    - export KUBECONFIG=/tmp/kubeconfig
    # Update kubeconfig with EKS cluster credentials
    - aws eks update-kubeconfig --region $AWS_DEFAULT_REGION --name $EKS_CLUSTER_NAME

# ── 1. Test ────────────────────────────────────────────────────────────
unit-tests:
  stage: test
  image: python:3.11-slim
  script:
    - pip install -r requirements.txt
    - pytest tests/unit/ -v --cov=app --cov-report=xml --junitxml=results.xml
  artifacts:
    reports:
      junit: results.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
  coverage: '/TOTAL.*\s+(\d+%)$/'

# ── 2. Build Docker Image ──────────────────────────────────────────────
build-image:
  stage: build
  image: docker:24.0
  services:
    - docker:24.0-dind
  script:
    - docker build
        --label "git-commit=$CI_COMMIT_SHA"
        --label "built-by=gitlab-ci"
        --label "pipeline-id=$CI_PIPELINE_ID"
        -t $IMAGE_URI:$CI_COMMIT_SHORT_SHA
        -t $IMAGE_URI:latest
        .
    - docker save $IMAGE_URI:$CI_COMMIT_SHORT_SHA > image.tar
  artifacts:
    paths:
      - image.tar
    expire_in: 2 hours
  rules:
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'

# ── 3. Push to ECR ─────────────────────────────────────────────────────
push-ecr:
  stage: push
  image: docker:24.0
  services:
    - docker:24.0-dind
  <<: *aws-setup
  needs:
    - job: build-image
      artifacts: true
  script:
    - aws ecr get-login-password --region $AWS_DEFAULT_REGION |
        docker login --username AWS --password-stdin $ECR_REGISTRY
    - docker load < image.tar
    - docker push $IMAGE_URI:$CI_COMMIT_SHORT_SHA
    - docker push $IMAGE_URI:latest
  rules:
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'

# ── 4. Security Scan ───────────────────────────────────────────────────
trivy-scan:
  stage: security-scan
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  needs: [push-ecr]
  <<: *aws-setup
  script:
    - aws ecr get-login-password --region $AWS_DEFAULT_REGION |
        docker login --username AWS --password-stdin $ECR_REGISTRY
    # Fail pipeline if CRITICAL vulnerabilities found
    - trivy image
        --exit-code 1
        --severity CRITICAL
        --no-progress
        $IMAGE_URI:$CI_COMMIT_SHORT_SHA
  allow_failure: false
  rules:
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'

# ── 5. Deploy to Staging ───────────────────────────────────────────────
deploy-staging:
  stage: deploy-staging
  image:
    name: alpine/k8s:1.29.2
    entrypoint: [""]
  <<: *kubectl-setup
  needs: [trivy-scan]
  script:
    # Replace image placeholder in manifest
    - sed -i "s|IMAGE_PLACEHOLDER|$IMAGE_URI:$CI_COMMIT_SHORT_SHA|g" k8s/base/deployment.yaml
    - sed -i "s|IMAGE_TAG|$CI_COMMIT_SHORT_SHA|g" k8s/base/deployment.yaml

    # Apply manifests to staging namespace
    - kubectl apply -f k8s/base/ -n $K8S_STAGING_NS

    # Wait for rollout to complete (5 minute timeout)
    - kubectl rollout status deployment/my-app -n $K8S_STAGING_NS --timeout=300s
  environment:
    name: staging
    url: https://staging.myapp.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'
    - if: '$CI_COMMIT_BRANCH == "main"'

# ── 6. Helm-Based Deployment (Alternative to kubectl apply) ────────────
deploy-staging-helm:
  stage: deploy-staging
  image:
    name: alpine/helm:3.14.0
    entrypoint: [""]
  <<: *kubectl-setup
  needs: [trivy-scan]
  script:
    - helm upgrade --install my-app ./helm/my-app
        --namespace $K8S_STAGING_NS
        --create-namespace
        --values helm/my-app/values.yaml
        --values helm/my-app/values-staging.yaml
        --set image.repository=$IMAGE_URI
        --set image.tag=$CI_COMMIT_SHORT_SHA
        --set global.environment=staging
        --wait
        --timeout 5m
        --atomic                    # Roll back automatically if deployment fails
  environment:
    name: staging
    url: https://staging.myapp.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: never                   # Use this job OR deploy-staging, not both
    - when: never

# ── 7. Integration Tests Against Staging ──────────────────────────────
integration-tests:
  stage: integration-test
  image: python:3.11-slim
  needs: [deploy-staging]
  script:
    - pip install requests pytest
    - STAGING_URL=https://staging.myapp.com pytest tests/integration/ -v
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

# ── 8. Deploy to Production (Manual) ──────────────────────────────────
deploy-production:
  stage: deploy-production
  image:
    name: alpine/k8s:1.29.2
    entrypoint: [""]
  <<: *kubectl-setup
  needs: [integration-tests]
  script:
    - sed -i "s|IMAGE_PLACEHOLDER|$IMAGE_URI:$CI_COMMIT_SHORT_SHA|g" k8s/base/deployment.yaml
    - kubectl apply -f k8s/base/ -n $K8S_PROD_NS
    - kubectl rollout status deployment/my-app -n $K8S_PROD_NS --timeout=600s

    # Post-deploy health check
    - sleep 30
    - kubectl get pods -n $K8S_PROD_NS -l app=my-app
  environment:
    name: production
    url: https://myapp.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  when: manual

# ── 9. Rollback Production ─────────────────────────────────────────────
rollback-production:
  stage: deploy-production
  image:
    name: alpine/k8s:1.29.2
    entrypoint: [""]
  <<: *kubectl-setup
  script:
    - kubectl rollout undo deployment/my-app -n $K8S_PROD_NS
    - kubectl rollout status deployment/my-app -n $K8S_PROD_NS --timeout=300s
  environment:
    name: production
    action: prepare
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  when: manual
```

---

## 10. Multi-Environment Pipelines (Dev → Staging → Prod)

`[Intermediate → Advanced]`

### Git Branching Strategy

```
feature/new-feature
        │
        │ Merge Request
        ▼
    develop ──────────────── Auto-deploy to: Staging
        │
        │ Merge Request + Review
        ▼
      main ─────────────────── Auto-deploy to: Pre-prod
        │
        │ Manual Approval (Product Owner)
        ▼
    [tag: v1.2.3] ────────── Deploy to: Production
```

### Environment-Aware Pipeline Template

```yaml
# .gitlab-ci.yml — Multi-Environment Pipeline

stages:
  - test
  - build
  - deploy

# ── Deploy job template ────────────────────────────────────────────────
.deploy-template:
  stage: deploy
  image: amazon/aws-cli:latest
  script:
    - echo "Deploying to $DEPLOY_ENV with image $IMAGE_URI:$CI_COMMIT_SHORT_SHA"
    - aws ecs update-service
        --cluster $ECS_CLUSTER_NAME
        --service my-app-$DEPLOY_ENV
        --force-new-deployment
    - aws ecs wait services-stable
        --cluster $ECS_CLUSTER_NAME
        --services my-app-$DEPLOY_ENV

# ── Concrete jobs using the template ──────────────────────────────────
deploy-dev:
  extends: .deploy-template
  variables:
    DEPLOY_ENV: dev
    ECS_CLUSTER_NAME: dev-cluster
  environment:
    name: development
    url: https://dev.myapp.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

deploy-staging:
  extends: .deploy-template
  variables:
    DEPLOY_ENV: staging
    ECS_CLUSTER_NAME: staging-cluster
  environment:
    name: staging
    url: https://staging.myapp.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy-production:
  extends: .deploy-template
  variables:
    DEPLOY_ENV: production
    ECS_CLUSTER_NAME: prod-cluster
  environment:
    name: production
    url: https://myapp.com
  rules:
    - if: '$CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/'  # Only on version tags
  when: manual
```

### Using GitLab Environments for Deployment Tracking

GitLab tracks every deployment per environment. You get:

- History of every production deployment
- One-click rollback to previous version
- Link to the live deployment URL
- Who approved and deployed each release

```yaml
environment:
  name: production
  url: https://myapp.com
  on_stop: stop-production         # Job to run when environment is stopped
  auto_stop_in: 2 days             # Auto-stop after 2 days (great for review apps)
```

---

## 11. Security Best Practices

`[All Levels — Critical]`

### The Non-Negotiables

```yaml
# ── 1. Never hardcode secrets ─────────────────────────────────────────
# BAD:
variables:
  AWS_SECRET: "AKIAIOSFODNN7EXAMPLE"  # NEVER do this

# GOOD: Set in GitLab UI → Settings → CI/CD → Variables (masked + protected)
script:
  - aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY

# ── 2. Use protected branches + protected variables ───────────────────
# In GitLab: Settings → Repository → Protected Branches
# Set main and release/* as protected
# Only mark production secrets as "Protected" — they won't leak to feature branches

# ── 3. Use least-privilege IAM permissions ────────────────────────────
# Create a dedicated IAM user/role for CI with ONLY the permissions needed
# Never use admin credentials in pipelines

# ── 4. Pin image versions ─────────────────────────────────────────────
# BAD:
image: python:latest                # Can change unexpectedly

# GOOD:
image: python:3.11.9-slim-bookworm  # Pinned, reproducible, auditable

# ── 5. Scan dependencies ──────────────────────────────────────────────
dependency-scan:
  stage: test
  image: python:3.11-slim
  script:
    - pip install safety
    - safety check -r requirements.txt

# ── 6. Scan code (SAST) ───────────────────────────────────────────────
include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml  # Finds secrets in code history
```

### GitLab Security Templates (Free Tier)

```yaml
# Add these to your pipeline — they run automatically
include:
  - template: Security/SAST.gitlab-ci.yml              # Static code analysis
  - template: Security/Secret-Detection.gitlab-ci.yml  # Find leaked secrets
  - template: Security/Dependency-Scanning.gitlab-ci.yml
  - template: Security/Container-Scanning.gitlab-ci.yml
```

### IAM Best Practices for ECS/EKS

```
Instead of AWS Access Keys (long-lived credentials), prefer:
├── ECS: Use ECS Task Roles (IAM role attached to the task)
├── EKS: Use IRSA (IAM Roles for Service Accounts)
└── GitLab CI: Use OIDC/Web Identity Token (if self-hosted on AWS)
    └── Eliminates need for AWS_ACCESS_KEY_ID entirely
```

### OIDC-Based AWS Auth (Advanced — Most Secure)

```yaml
# Zero long-lived credentials — GitLab gets a short-lived token from AWS
assume-role:
  stage: .pre
  image: amazon/aws-cli:latest
  id_tokens:
    AWS_TOKEN:
      aud: https://gitlab.com
  script:
    - |
      export $(printf "AWS_ACCESS_KEY_ID=%s AWS_SECRET_ACCESS_KEY=%s AWS_SESSION_TOKEN=%s" \
        $(aws sts assume-role-with-web-identity \
          --role-arn $AWS_ROLE_ARN \
          --role-session-name "GitLabCI-$CI_PIPELINE_ID" \
          --web-identity-token "$AWS_TOKEN" \
          --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
          --output text))
```

---

## 12. Pipeline Best Practices by Level

`[All Levels]`

### Beginner Best Practices

```
✅ DO:
- Start with a simple pipeline (build → test → deploy)
- Use GitLab's built-in Container Registry
- Set ALL secrets in CI/CD Variables, never in YAML
- Add a test stage before every deploy
- Use `when: manual` for production deployments

❌ DON'T:
- Copy-paste credentials into .gitlab-ci.yml
- Deploy directly to production without testing
- Use `latest` tag for Docker images in production
- Skip the test stage to "save time"
```

### Intermediate Best Practices

```yaml
# ── Use includes to keep pipelines DRY ───────────────────────────────
# .gitlab/ci/test.yml
unit-tests:
  stage: test
  script: [pytest]

# .gitlab-ci.yml
include:
  - local: '.gitlab/ci/test.yml'
  - local: '.gitlab/ci/build.yml'
  - local: '.gitlab/ci/deploy.yml'
  - project: 'company/shared-pipelines'    # Share templates across projects
    ref: main
    file: '/templates/docker-build.yml'

# ── Use environments properly ─────────────────────────────────────────
environment:
  name: production
  url: https://myapp.com

# ── Always define resource limits ────────────────────────────────────
job:
  timeout: 20 minutes
  resource_group: production       # Prevents concurrent production deployments
```

### Advanced Best Practices

```yaml
# ── Parallel test execution ───────────────────────────────────────────
test:
  parallel: 4                      # Split tests across 4 runners automatically
  script:
    - pytest --splits 4 --group $CI_NODE_INDEX

# ── Dynamic environments (review apps) ───────────────────────────────
review:
  stage: deploy
  script:
    - kubectl create namespace review-$CI_COMMIT_REF_SLUG || true
    - helm upgrade --install review-$CI_COMMIT_REF_SLUG ./helm
        --namespace review-$CI_COMMIT_REF_SLUG
        --set ingress.host=review-$CI_COMMIT_REF_SLUG.myapp.com
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://review-$CI_COMMIT_REF_SLUG.myapp.com
    on_stop: stop-review
    auto_stop_in: 3 days
  rules:
    - if: '$CI_MERGE_REQUEST_ID'

stop-review:
  script:
    - helm uninstall review-$CI_COMMIT_REF_SLUG
        --namespace review-$CI_COMMIT_REF_SLUG
    - kubectl delete namespace review-$CI_COMMIT_REF_SLUG
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    action: stop
  when: manual

# ── Prevent concurrent production deployments ─────────────────────────
deploy-production:
  resource_group: production        # Only 1 production deploy at a time globally

# ── Canary deployments ────────────────────────────────────────────────
deploy-canary:
  stage: deploy
  script:
    # Deploy to 10% of traffic first
    - kubectl set image deployment/my-app-canary my-app=$IMAGE_URI:$CI_COMMIT_SHORT_SHA
    - kubectl scale deployment/my-app-canary --replicas=1
  environment:
    name: production/canary
  when: manual

promote-to-full:
  stage: deploy
  needs: [deploy-canary]
  script:
    - kubectl set image deployment/my-app my-app=$IMAGE_URI:$CI_COMMIT_SHORT_SHA
    - kubectl scale deployment/my-app-canary --replicas=0
  when: manual
```

---

## 13. Monitoring, Observability & Troubleshooting

`[All Levels]`

### Debugging Failed Pipelines

```bash
# In the GitLab job log, look for:
# 1. The exact failing command
# 2. Exit code
# 3. Error message

# Add debug output to your pipeline:
script:
  - set -x                          # Print every command as it runs
  - echo "Current directory: $(pwd)"
  - echo "Files: $(ls -la)"
  - echo "Docker version: $(docker version)"
  - echo "AWS identity: $(aws sts get-caller-identity)"
```

### Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Runner not found` | No runner with matching tag | Add correct tags to job or runner |
| `docker: not found` | Using wrong image | Add `image: docker:24.0` and `services: [docker:24.0-dind]` |
| `Permission denied to ECR` | IAM permissions missing | Add `ecr:*` permissions to IAM user |
| `kubectl: Unauthorized` | Expired kubeconfig | Regenerate and update KUBECONFIG_BASE64 variable |
| `Artifact not found` | Job didn't pass artifacts to next job | Check `needs` with `artifacts: true` |
| `Variable is masked and cannot be used` | Masked variable used in `script` output | Variables must match `[a-zA-Z0-9+/=]+` format to be masked |

### Pipeline Metrics to Track

```yaml
# Add job duration reporting
after_script:
  - echo "Job $CI_JOB_NAME completed in $CI_JOB_DURATION seconds"
  - echo "Pipeline $CI_PIPELINE_ID for $CI_COMMIT_SHORT_SHA"
```

Key pipeline health metrics to monitor:
- **Pipeline success rate** — target > 95%
- **Average pipeline duration** — keep under 15 minutes
- **Failed deployment rate** — target < 2%
- **Time to recovery** — how fast you can fix a failed deploy

### Notifications

```yaml
# Notify Slack on production deployment failure
notify-failure:
  stage: .post                      # Runs after all other stages
  image: curlimages/curl:latest
  script:
    - |
      curl -X POST $SLACK_WEBHOOK_URL \
        -H 'Content-type: application/json' \
        -d "{
          \"text\": \"❌ Production deploy FAILED for <$CI_PIPELINE_URL|pipeline #$CI_PIPELINE_ID>\",
          \"channel\": \"#deployments\"
        }"
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: on_failure
```

---

## 14. GitLab Runners — Self-Hosted vs Shared

`[Beginner → Intermediate]`

### Comparison

| Feature | Shared Runners (GitLab.com) | Self-Hosted Runners |
|---------|-----------------------------|--------------------|
| **Setup** | Zero — just works | Requires installation |
| **Cost** | Free tier (400 min/month) then paid | EC2/server costs |
| **Performance** | Standard | You control (can use powerful EC2) |
| **AWS Access** | Must use credentials in variables | Can use EC2 instance profile |
| **Data privacy** | Code runs on GitLab's infra | Code runs on your infra |
| **Best for** | Getting started, open source | Production workloads, compliance |

### Installing a Self-Hosted Runner on Ubuntu

```bash
# Step 1: Install GitLab Runner
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install -y gitlab-runner

# Step 2: Get your registration token
# GitLab Project → Settings → CI/CD → Runners → New project runner

# Step 3: Register the runner
sudo gitlab-runner register \
  --url "https://gitlab.com/" \
  --registration-token "YOUR_TOKEN_HERE" \
  --executor "docker" \
  --docker-image "alpine:latest" \
  --description "ubuntu-docker-runner" \
  --tag-list "ubuntu,docker,production" \
  --run-untagged="false" \
  --locked="true"

# Step 4: Verify it's running
sudo gitlab-runner status
sudo gitlab-runner verify
```

### Using Tags to Route Jobs to Specific Runners

```yaml
# Only run this job on your Ubuntu production runner
deploy-production:
  stage: deploy
  tags:
    - ubuntu          # Matches the tag you set during registration
    - production
  script:
    - ./deploy.sh
```

### Runner Executor Types

```
shell      → Runs directly on the host machine (your original use case)
docker     → Each job runs in a fresh Docker container (recommended)
kubernetes → Each job runs as a Kubernetes pod
docker+machine → Autoscales Docker runners on demand
```

---

## 15. Quick Reference Cheat Sheet

### `.gitlab-ci.yml` Skeleton

```yaml
stages: [test, build, push, deploy]

variables:
  IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

default:
  image: python:3.11-slim
  before_script: [pip install -r requirements.txt]

test:
  stage: test
  script: [pytest]

build:
  stage: build
  image: docker:24.0
  services: [docker:24.0-dind]
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $IMAGE .
    - docker push $IMAGE
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy-staging:
  stage: deploy
  script: [./deploy.sh staging]
  environment: {name: staging}
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy-production:
  stage: deploy
  script: [./deploy.sh production]
  environment: {name: production}
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  when: manual
```

### Pre-defined Variables Quick Reference

```bash
$CI_COMMIT_BRANCH          # Current branch
$CI_COMMIT_SHORT_SHA       # Short commit hash (8 chars)
$CI_COMMIT_TAG             # Tag name (if pipeline triggered by tag)
$CI_PIPELINE_ID            # Unique pipeline ID
$CI_REGISTRY_IMAGE         # gitlab.com/group/project container path
$CI_ENVIRONMENT_NAME       # "staging", "production", etc.
$CI_MERGE_REQUEST_ID       # Set only in MR pipelines
$CI_PROJECT_DIR            # Where your code is checked out
```

### Useful `rules` Patterns

```yaml
# Only on main branch
- if: '$CI_COMMIT_BRANCH == "main"'

# Only on merge requests
- if: '$CI_MERGE_REQUEST_ID'

# Only on version tags
- if: '$CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/'

# Only when Python files change
- changes: ["**/*.py", "requirements.txt"]

# Skip on scheduled pipelines
- if: '$CI_PIPELINE_SOURCE == "schedule"'
  when: never
```

### ECS Deploy One-Liner

```bash
aws ecs update-service \
  --cluster $ECS_CLUSTER \
  --service $ECS_SERVICE \
  --force-new-deployment \
  --region $AWS_DEFAULT_REGION
```

### kubectl Rollout Commands

```bash
kubectl rollout status deployment/my-app -n production   # Watch deploy
kubectl rollout history deployment/my-app -n production  # See history
kubectl rollout undo deployment/my-app -n production     # Rollback
kubectl rollout undo deployment/my-app --to-revision=3   # Rollback to specific version
```

---

## Learning Path: Where to Go Next

```
You are here:
   Using GitLab to push code + manual SSH copy
         │
         ▼ [This Guide]
   Basic pipeline with test + deploy stage
         │
         ▼ [Week 2-3]
   Docker builds + GitLab Container Registry
         │
         ▼ [Week 4-6]
   ECS deployment with staging + production
         │
         ▼ [Month 2-3]
   EKS + Helm + OIDC auth + Review Apps
         │
         ▼ [Month 4+]
   GitOps with ArgoCD + full observability
```

### Recommended Next Steps for Your Setup

1. **Today**: Create a basic `.gitlab-ci.yml` that runs tests on every push
2. **This week**: Install a GitLab Runner on your Ubuntu server (shell executor)
3. **Next week**: Add auto-deployment to your server using the runner (replaces manual SSH + copy)
4. **Month 1**: Add Docker build + push to ECR
5. **Month 2**: Deploy to ECS with staging → production pipeline
6. **Month 3**: Graduate to EKS with Helm charts

---

*Built with care for DevOps engineers at every level. Questions and contributions welcome.*
