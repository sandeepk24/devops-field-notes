# Complete GitHub DevOps Guide: AWS ECS Deployment

> **A comprehensive guide to modern CI/CD practices using GitHub Actions and AWS ECS/Fargate**  
> *Suitable for DevOps engineers from intermediate to senior level*

[![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=for-the-badge&logo=github-actions&logoColor=white)](https://github.com/features/actions)
[![AWS ECS](https://img.shields.io/badge/AWS_ECS-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/ecs/)
[![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=for-the-badge&logo=terraform&logoColor=white)](https://www.terraform.io/)

---

## Table of Contents

- [Introduction](#introduction)
- [Understanding GitHub for DevOps](#understanding-github-for-devops)
- [GitHub CI/CD Features](#github-cicd-features)
- [DevOps Best Practices](#devops-best-practices)
- [AWS Integration Strategies](#aws-integration-strategies)
- [ECS Fargate Deployment Architecture](#ecs-fargate-deployment-architecture)
- [Complete Implementation](#complete-implementation)
- [Fargate vs Fargate Spot](#fargate-vs-fargate-spot)
- [Monitoring and Observability](#monitoring-and-observability)
- [Cost Optimization](#cost-optimization)
- [Troubleshooting](#troubleshooting)
- [Advanced Patterns](#advanced-patterns)

---

## Introduction

### What is GitHub?

**GitHub** is a cloud-based platform that combines:
- **Version Control** (Git) - Track every code change
- **Collaboration** - Pull requests, code reviews, discussions
- **CI/CD** - GitHub Actions for automation
- **Security** - Dependabot, secret scanning, OIDC authentication
- **Project Management** - Issues, project boards, milestones

**For DevOps Engineers**, GitHub is:
- Your single source of truth for infrastructure-as-code
- Your automation engine for build, test, and deployment pipelines
- Your security gateway with policy enforcement and secret management
- Your audit log for compliance and traceability

### Why GitHub + AWS ECS?

| Feature | Benefit |
|---------|---------|
| **Serverless Containers** | No EC2 instance management with Fargate |
| **Native Integration** | AWS official GitHub Actions |
| **Security** | OIDC keyless authentication |
| **Cost Efficiency** | Fargate Spot up to 70% savings |
| **Scalability** | Auto-scaling based on metrics |
| **Developer Experience** | Git push to production pipeline |

---

## Understanding GitHub for DevOps

### The GitHub Workflow

```mermaid
graph LR
    A[Developer] -->|Push| B[Feature Branch]
    B -->|Create| C[Pull Request]
    C -->|Trigger| D[GitHub Actions]
    D -->|Run| E[Tests]
    D -->|Build| F[Docker Image]
    E -->|Pass| G[Review]
    G -->|Approve| H[Merge to Main]
    H -->|Deploy| I[Staging]
    I -->|Manual Approval| J[Production]
```

### Key Concepts

#### 1. **Repository Structure**
```
my-app/
├── .github/
│   ├── workflows/           # CI/CD pipeline definitions
│   │   ├── ci.yml          # Continuous Integration
│   │   ├── deploy-staging.yml
│   │   └── deploy-prod.yml
│   ├── CODEOWNERS          # Auto-assign reviewers
│   └── dependabot.yml      # Automated dependency updates
├── terraform/              # Infrastructure as Code
│   ├── environments/
│   │   ├── dev/
│   │   ├── staging/
│   │   └── prod/
│   ├── modules/
│   └── backend.tf
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── src/                    # Application code
├── tests/
├── requirements.txt
└── README.md
```

#### 2. **Branch Strategy**

**Trunk-Based Development** (Recommended for modern DevOps):
```
main (protected, always deployable)
  ↑
  ├── feature/user-auth
  ├── feature/api-optimization  
  └── hotfix/security-patch
```

**Best Practices:**
- Keep feature branches short-lived (< 2 days)
- Main branch is always production-ready
- Use feature flags for incomplete features
- Protect main with required reviews and status checks

---

## GitHub CI/CD Features

### 1. GitHub Actions

**What is it?** An event-driven automation platform integrated directly into GitHub.

#### Core Components

```yaml
# Anatomy of a GitHub Actions workflow
name: CI Pipeline                    # Workflow name
on:                                  # Trigger events
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:                                # Units of work
  test:                              # Job name
    runs-on: ubuntu-latest           # Runner environment
    steps:                           # Sequential tasks
      - uses: actions/checkout@v4    # Reusable action
      - run: pytest tests/           # Shell command
```

#### Workflow Triggers

| Trigger | Use Case | Example |
|---------|----------|---------|
| `push` | Code changes | Deploy on push to main |
| `pull_request` | Code review | Run tests on PR |
| `schedule` | Cron jobs | Nightly builds |
| `workflow_dispatch` | Manual | Manual production deploy |
| `release` | Version tagging | Build release artifacts |
| `workflow_call` | Reusable | Shared deployment logic |

#### Matrix Builds

Test across multiple configurations simultaneously:

```yaml
strategy:
  matrix:
    python-version: ['3.9', '3.10', '3.11', '3.12']
    os: [ubuntu-latest, macos-latest]
```

### 2. GitHub Environments

Environments provide deployment protection and environment-specific secrets.

```yaml
jobs:
  deploy-production:
    environment: 
      name: production
      url: https://api.example.com
    steps:
      - name: Deploy
        run: ./deploy.sh
```

**Environment Protection Rules:**
- Required reviewers (1-6 people)
- Wait timer (delay deployment)
- Deployment branches (only from main)
- Environment secrets (prod database credentials)

### 3. Secrets Management

**Three levels of secrets:**

1. **Organization Secrets** - Shared across all repos
2. **Repository Secrets** - Specific to one repo
3. **Environment Secrets** - Specific to deployment environment

```yaml
steps:
  - name: Deploy
    env:
      AWS_REGION: ${{ secrets.AWS_REGION }}
      DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
```

⚠️ **Security Best Practices:**
- Never log secrets (`echo $SECRET` exposes it)
- Use `::add-mask::` to hide dynamic values
- Rotate secrets regularly
- Use OIDC instead of long-lived credentials

### 4. Reusable Workflows

DRY principle for CI/CD:

```yaml
# .github/workflows/deploy-template.yml
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
    secrets:
      aws-role:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to ${{ inputs.environment }}
        run: echo "Deploying..."
```

```yaml
# .github/workflows/deploy-prod.yml
jobs:
  deploy:
    uses: ./.github/workflows/deploy-template.yml
    with:
      environment: production
    secrets:
      aws-role: ${{ secrets.PROD_AWS_ROLE }}
```

### 5. GitHub Packages (Container Registry)

Alternative to Docker Hub or AWS ECR:

```yaml
- name: Login to GitHub Container Registry
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- name: Build and push
  run: |
    docker build -t ghcr.io/${{ github.repository }}:${{ github.sha }} .
    docker push ghcr.io/${{ github.repository }}:${{ github.sha }}
```

### 6. Dependabot

Automated dependency updates and security patches:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
      
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

---

## DevOps Best Practices

### 1. The Pipeline Pyramid

```
                    Production Deploy
                    (slowest, requires approval)
                           ▲
                    ┌──────┴──────┐
              Staging Deploy   Security Scan
                     ▲               ▲
              ┌──────┴───────────────┘
        Integration Tests
              ▲
        ┌─────┴─────┐
    Unit Tests   Linting
    (fastest, run on every commit)
```

**Principle:** Fast feedback first, expensive operations last.

### 2. Immutable Infrastructure

```yaml
# ❌ BAD: Mutable infrastructure
- name: Deploy
  run: |
    ssh production "cd /app && git pull && pip install -r requirements.txt"

# ✅ GOOD: Immutable infrastructure
- name: Deploy
  run: |
    aws ecs update-service \
      --cluster production \
      --service api \
      --task-definition api:${{ github.sha }}
```

**Why?**
- Reproducible deployments
- Easy rollbacks
- No configuration drift
- Clear audit trail

### 3. GitOps Workflow

```
Git Repository (Source of Truth)
        ↓
    Git Push
        ↓
  GitHub Actions (Automation)
        ↓
   Infrastructure Changes
        ↓
   Desired State Applied
```

**Principles:**
- Declarative infrastructure (Terraform, CloudFormation)
- Git as single source of truth
- Automated sync between Git and infrastructure
- Pull requests for all changes

### 4. Shift-Left Security

Catch issues early in the pipeline:

```yaml
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      # SAST - Static Application Security Testing
      - name: Run Bandit (Python security linter)
        run: bandit -r src/
      
      # Dependency scanning
      - name: Check for known vulnerabilities
        run: safety check
      
      # Secret scanning
      - name: Scan for leaked secrets
        uses: trufflesecurity/trufflehog@main
      
      # Container scanning
      - name: Scan Docker image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:latest
          severity: CRITICAL,HIGH
```

### 5. Observability from Day One

**The Three Pillars:**

```python
# Logs (structured JSON)
import structlog
logger = structlog.get_logger()
logger.info("user_login", user_id=123, ip="1.2.3.4")

# Metrics
from prometheus_client import Counter
login_counter = Counter('user_logins_total', 'Total user logins')
login_counter.inc()

# Traces
from aws_xray_sdk.core import xray_recorder

@xray_recorder.capture('process_order')
def process_order(order_id):
    # Distributed tracing
    pass
```

### 6. Progressive Deployment Strategies

#### Blue/Green Deployment
```
Blue (current)    Green (new)
    100% ────────→ 0%
      ↓
     50% ────────→ 50%
      ↓
      0% ────────→ 100%
```

#### Canary Deployment
```
Stable: 95%
Canary: 5%  ← Monitor for errors
    ↓
Stable: 50%
Canary: 50%
    ↓
Stable: 0%
Canary: 100% (becomes new stable)
```

---

## AWS Integration Strategies

### Comparison: Access Keys vs OIDC

| Feature | IAM Access Keys | OIDC (Recommended) |
|---------|----------------|-------------------|
| **Security** | Long-lived credentials | Short-lived tokens (1 hour) |
| **Rotation** | Manual (90 days) | Automatic |
| **Exposure Risk** | High (stored in GitHub) | Low (never stored) |
| **Permissions** | Static IAM user | Dynamic role assumption |
| **Setup Complexity** | Simple | Moderate |
| **Best Practice** | ❌ Legacy | ✅ Modern |

### OIDC Setup (Recommended)

#### Architecture

```
GitHub Actions Workflow
        ↓
   OIDC Token (JWT)
        ↓
AWS STS AssumeRoleWithWebIdentity
        ↓
Temporary Credentials (1 hour)
        ↓
   AWS API Calls
```

#### Step 1: Create OIDC Provider in AWS

```bash
# Using AWS CLI
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

Or with Terraform:

```hcl
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"
  
  client_id_list = [
    "sts.amazonaws.com",
  ]
  
  thumbprint_list = [
    "6938fd4d98bab03faadb97b34396831e3780aea1"
  ]
}
```

#### Step 2: Create IAM Role

```hcl
data "aws_iam_policy_document" "github_actions_assume_role" {
  statement {
    effect = "Allow"
    
    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }
    
    actions = ["sts:AssumeRoleWithWebIdentity"]
    
    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }
    
    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      # Only this repo and branch can assume the role
      values   = ["repo:your-org/your-repo:ref:refs/heads/main"]
    }
  }
}

resource "aws_iam_role" "github_actions" {
  name               = "GitHubActionsDeployRole"
  assume_role_policy = data.aws_iam_policy_document.github_actions_assume_role.json
}

resource "aws_iam_role_policy_attachment" "ecr_power_user" {
  role       = aws_iam_role.github_actions.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser"
}

# Custom policy for ECS deployments
resource "aws_iam_role_policy" "ecs_deploy" {
  name = "ECSDeploymentPolicy"
  role = aws_iam_role.github_actions.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ecs:DescribeTaskDefinition",
          "ecs:RegisterTaskDefinition",
          "ecs:UpdateService",
          "ecs:DescribeServices"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = "iam:PassRole"
        Resource = "arn:aws:iam::YOUR_ACCOUNT:role/ecsTaskExecutionRole"
      }
    ]
  })
}
```

#### Step 3: Use in GitHub Actions

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # Required for OIDC
      contents: read
    
    steps:
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsDeployRole
          aws-region: us-east-1
      
      - name: Verify credentials
        run: aws sts get-caller-identity
```

### Legacy: Access Keys (Not Recommended)

```yaml
# Store in GitHub Secrets: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-1
```

⚠️ **Risks:**
- Credentials can be leaked
- No automatic expiration
- Broad permissions difficult to scope
- Rotation requires manual intervention

---

## ECS Fargate Deployment Architecture

### Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    GitHub Repository                     │
│  ┌────────────┐    ┌─────────────┐   ┌──────────────┐  │
│  │   Code     │───▶│GitHub Actions│──▶│  Artifacts   │  │
│  └────────────┘    └─────────────┘   └──────────────┘  │
└────────────────────────────┬────────────────────────────┘
                             │ OIDC Auth
                             ▼
┌─────────────────────────────────────────────────────────┐
│                         AWS Cloud                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │                  Amazon ECR                       │  │
│  │           (Docker Image Registry)                 │  │
│  └─────────────────────┬────────────────────────────┘  │
│                        │                                │
│  ┌─────────────────────▼────────────────────────────┐  │
│  │              ECS Task Definition                  │  │
│  │  ┌──────────────────────────────────────────┐    │  │
│  │  │ Container: myapp:sha-abc123              │    │  │
│  │  │ CPU: 256, Memory: 512                    │    │  │
│  │  │ Port: 8000                                │    │  │
│  │  │ Env: PROD                                 │    │  │
│  │  └──────────────────────────────────────────┘    │  │
│  └─────────────────────┬────────────────────────────┘  │
│                        │                                │
│  ┌─────────────────────▼────────────────────────────┐  │
│  │              ECS Fargate Service                  │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐          │  │
│  │  │  Task   │  │  Task   │  │  Task   │          │  │
│  │  │ (AZ-1)  │  │ (AZ-2)  │  │ (AZ-3)  │          │  │
│  │  └────┬────┘  └────┬────┘  └────┬────┘          │  │
│  └───────┼────────────┼────────────┼───────────────┘  │
│          └────────────┴────────────┘                   │
│                      │                                 │
│  ┌───────────────────▼──────────────────────────────┐ │
│  │      Application Load Balancer (ALB)             │ │
│  │         Health Checks: /health                   │ │
│  └───────────────────┬──────────────────────────────┘ │
│                      │                                 │
└──────────────────────┼─────────────────────────────────┘
                       │
                       ▼
                  Internet Users
```

### Key Components Explained

#### 1. **ECS Cluster**
Logical grouping of tasks/services. Think of it as a namespace.

```hcl
resource "aws_ecs_cluster" "main" {
  name = "production-cluster"
  
  setting {
    name  = "containerInsights"
    value = "enabled"  # Enable CloudWatch Container Insights
  }
}
```

#### 2. **Task Definition**
Blueprint for your container: image, resources, environment variables.

```json
{
  "family": "myapp",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest",
      "portMappings": [{"containerPort": 8000}],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/myapp",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

#### 3. **ECS Service**
Ensures desired number of tasks are running and handles load balancing.

```hcl
resource "aws_ecs_service" "app" {
  name            = "myapp-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 3
  launch_type     = "FARGATE"
  
  network_configuration {
    subnets          = aws_subnet.private[*].id
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false
  }
  
  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "app"
    container_port   = 8000
  }
  
  # Blue/Green deployment configuration
  deployment_configuration {
    maximum_percent         = 200  # Can run 2x tasks during deploy
    minimum_healthy_percent = 100  # Never go below desired count
  }
  
  # Automatic rollback on failure
  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }
}
```

#### 4. **IAM Roles**

Two roles are needed:

**Task Execution Role** (pulls image, writes logs):
```hcl
resource "aws_iam_role" "ecs_task_execution" {
  name = "ecsTaskExecutionRole"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_task_execution" {
  role       = aws_iam_role.ecs_task_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}
```

**Task Role** (application permissions - S3, DynamoDB, etc.):
```hcl
resource "aws_iam_role" "ecs_task" {
  name = "ecsTaskRole"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
    }]
  })
}

# Attach policies for your app's AWS resource access
resource "aws_iam_role_policy" "app_s3_access" {
  role = aws_iam_role.ecs_task.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "s3:GetObject",
        "s3:PutObject"
      ]
      Resource = "arn:aws:s3:::my-app-bucket/*"
    }]
  })
}
```

---

## Complete Implementation

### 1. Python Application Structure

```
myapp/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── config.py
│   ├── models/
│   ├── routes/
│   └── utils/
├── tests/
│   ├── test_api.py
│   └── test_models.py
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
├── terraform/
├── Dockerfile
├── requirements.txt
├── requirements-dev.txt
└── README.md
```

### 2. Sample FastAPI Application

```python
# app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
import structlog
import os

# Structured logging
structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.stdlib.PositionalArgumentsFormatter(),
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.UnicodeDecoder(),
        structlog.processors.JSONRenderer()
    ],
    wrapper_class=structlog.stdlib.BoundLogger,
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
)

logger = structlog.get_logger()

app = FastAPI(
    title="MyApp API",
    version="1.0.0",
    description="Production-ready FastAPI application"
)

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=os.getenv("ALLOWED_ORIGINS", "*").split(","),
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/health")
async def health_check():
    """Health check endpoint for ALB"""
    return JSONResponse(
        status_code=200,
        content={
            "status": "healthy",
            "version": "1.0.0",
            "environment": os.getenv("ENVIRONMENT", "unknown")
        }
    )

@app.get("/")
async def root():
    logger.info("root_endpoint_accessed", environment=os.getenv("ENVIRONMENT"))
    return {"message": "Welcome to MyApp API"}

@app.get("/api/v1/data")
async def get_data():
    logger.info("data_endpoint_accessed")
    # Your business logic here
    return {"data": [1, 2, 3, 4, 5]}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

```python
# app/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    environment: str = "development"
    debug: bool = False
    database_url: str
    redis_url: str
    aws_region: str = "us-east-1"
    
    class Config:
        env_file = ".env"

settings = Settings()
```

### 3. Dockerfile (Multi-stage Build)

```dockerfile
# Dockerfile
FROM python:3.11-slim as builder

# Build arguments for metadata
ARG BUILD_DATE
ARG VCS_REF
ARG VERSION=1.0.0

# Labels following OCI image spec
LABEL org.opencontainers.image.created=$BUILD_DATE \
      org.opencontainers.image.authors="DevOps Team" \
      org.opencontainers.image.url="https://github.com/yourorg/myapp" \
      org.opencontainers.image.source="https://github.com/yourorg/myapp" \
      org.opencontainers.image.version=$VERSION \
      org.opencontainers.image.revision=$VCS_REF \
      org.opencontainers.image.title="MyApp" \
      org.opencontainers.image.description="Production FastAPI application"

WORKDIR /build

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Copy and install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# =============================================================================
# Final stage
# =============================================================================
FROM python:3.11-slim

WORKDIR /app

# Copy Python packages from builder
COPY --from=builder /root/.local /root/.local

# Copy application code
COPY ./app /app/app

# Create non-root user for security
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

# Make sure scripts in .local are usable
ENV PATH=/root/.local/bin:$PATH \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

# Health check
HEALTHCHECK --interval=30s \
            --timeout=3s \
            --start-period=40s \
            --retries=3 \
  CMD python -c "import requests; requests.get('http://localhost:8000/health', timeout=2)" || exit 1

EXPOSE 8000

# Use exec form for proper signal handling
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "2"]
```

### 4. GitHub Actions CI Workflow

```yaml
# .github/workflows/ci.yml
name: Continuous Integration

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main, develop]

env:
  PYTHON_VERSION: '3.11'

jobs:
  lint:
    name: Lint Code
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: 'pip'
      
      - name: Install dependencies
        run: |
          pip install ruff mypy black isort
      
      - name: Run Black
        run: black --check .
      
      - name: Run isort
        run: isort --check-only .
      
      - name: Run Ruff
        run: ruff check .
      
      - name: Run MyPy
        run: mypy app/

  test:
    name: Run Tests
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: 'pip'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-dev.txt
      
      - name: Run unit tests
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379/0
        run: |
          pytest tests/ \
            --cov=app \
            --cov-report=xml \
            --cov-report=html \
            --cov-report=term-missing \
            --junitxml=pytest-report.xml \
            -v
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.xml
          flags: unittests
          name: codecov-umbrella
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: pytest-report.xml

  security:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      
      - name: Install dependencies
        run: |
          pip install safety bandit
      
      - name: Run Safety (dependency vulnerabilities)
        run: safety check --json
      
      - name: Run Bandit (SAST)
        run: bandit -r app/ -f json -o bandit-report.json || true
      
      - name: Upload Bandit report
        uses: actions/upload-artifact@v4
        with:
          name: bandit-report
          path: bandit-report.json

  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [lint, test, security]
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Build image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: false
          tags: myapp:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            VCS_REF=${{ github.sha }}
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
      
      - name: Upload Trivy results to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
```

### 5. GitHub Actions CD Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy to'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: myapp
  ECS_CLUSTER: production-cluster
  ECS_SERVICE: myapp-service
  CONTAINER_NAME: app

jobs:
  deploy:
    name: Deploy to ${{ github.event.inputs.environment || 'production' }}
    runs-on: ubuntu-latest
    environment: 
      name: ${{ github.event.inputs.environment || 'production' }}
      url: https://api.example.com
    
    permissions:
      id-token: write
      contents: read
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}
          role-session-name: GitHubActions-${{ github.run_id }}
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Build, tag, and push image
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker buildx build \
            --platform linux/amd64 \
            --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
            --build-arg VCS_REF=${{ github.sha }} \
            --build-arg VERSION=${{ github.ref_name }} \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:latest \
            --push \
            .
          
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT
      
      - name: Download current task definition
        run: |
          aws ecs describe-task-definition \
            --task-definition ${{ env.ECS_SERVICE }} \
            --query taskDefinition \
            > task-definition.json
          
          # Remove fields that can't be used in RegisterTaskDefinition
          cat task-definition.json | \
            jq 'del(.taskDefinitionArn, .revision, .status, .requiresAttributes, .compatibilities, .registeredAt, .registeredBy)' \
            > task-def-clean.json
      
      - name: Fill in new image ID in task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-def-clean.json
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ steps.build-image.outputs.image }}
      
      - name: Deploy to Amazon ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
          wait-for-minutes: 10
      
      - name: Verify deployment
        run: |
          echo "✅ Deployment successful!"
          echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
          echo "📦 Image: ${{ steps.build-image.outputs.image }}"
          echo "🎯 Cluster: ${{ env.ECS_CLUSTER }}"
          echo "🚀 Service: ${{ env.ECS_SERVICE }}"
          echo "🌍 Region: ${{ env.AWS_REGION }}"
          echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
          
          # Get service details
          aws ecs describe-services \
            --cluster ${{ env.ECS_CLUSTER }} \
            --services ${{ env.ECS_SERVICE }} \
            --query 'services[0].{RunningCount:runningCount,DesiredCount:desiredCount,Status:status}' \
            --output table
      
      - name: Create deployment summary
        run: |
          cat >> $GITHUB_STEP_SUMMARY <<EOF
          ## Deployment Summary 🚀
          
          - **Environment**: ${{ github.event.inputs.environment || 'production' }}
          - **Image**: \`${{ steps.build-image.outputs.image }}\`
          - **Cluster**: ${{ env.ECS_CLUSTER }}
          - **Service**: ${{ env.ECS_SERVICE }}
          - **Region**: ${{ env.AWS_REGION }}
          - **Deployed by**: @${{ github.actor }}
          - **Commit**: ${{ github.sha }}
          
          ### Verification
          Check the deployment at: https://console.aws.amazon.com/ecs/v2/clusters/${{ env.ECS_CLUSTER }}/services/${{ env.ECS_SERVICE }}
          EOF

  smoke-test:
    name: Run Smoke Tests
    needs: deploy
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Wait for service stabilization
        run: sleep 30
      
      - name: Run smoke tests
        run: |
          # Replace with your production URL
          ENDPOINT="https://api.example.com"
          
          echo "Testing health endpoint..."
          curl -f $ENDPOINT/health || exit 1
          
          echo "Testing root endpoint..."
          curl -f $ENDPOINT/ || exit 1
          
          echo "✅ Smoke tests passed!"

  rollback:
    name: Rollback Deployment
    needs: [deploy, smoke-test]
    if: failure()
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Rollback to previous task definition
        run: |
          echo "🔄 Rolling back to previous task definition..."
          
          # Get previous task definition
          PREVIOUS_TASK_DEF=$(aws ecs describe-services \
            --cluster ${{ env.ECS_CLUSTER }} \
            --services ${{ env.ECS_SERVICE }} \
            --query 'services[0].deployments[1].taskDefinition' \
            --output text)
          
          if [ "$PREVIOUS_TASK_DEF" != "None" ]; then
            aws ecs update-service \
              --cluster ${{ env.ECS_CLUSTER }} \
              --service ${{ env.ECS_SERVICE }} \
              --task-definition $PREVIOUS_TASK_DEF \
              --force-new-deployment
            
            echo "✅ Rolled back to: $PREVIOUS_TASK_DEF"
          else
            echo "❌ No previous task definition found"
            exit 1
          fi
```

### 6. Complete Terraform Infrastructure

```hcl
# terraform/main.tf
terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket         = "myapp-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-lock"
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Project     = "MyApp"
      Environment = var.environment
      ManagedBy   = "Terraform"
      Repository  = "github.com/yourorg/myapp"
    }
  }
}

# Variables
variable "aws_region" {
  description = "AWS region"
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  default     = "production"
}

variable "app_name" {
  description = "Application name"
  default     = "myapp"
}

variable "app_port" {
  description = "Port exposed by the Docker image"
  default     = 8000
}

variable "app_count" {
  description = "Number of docker containers to run"
  default     = 3
}

variable "fargate_cpu" {
  description = "Fargate instance CPU units (1024 = 1 vCPU)"
  default     = 512
}

variable "fargate_memory" {
  description = "Fargate instance memory (MB)"
  default     = 1024
}

variable "health_check_path" {
  description = "Health check path"
  default     = "/health"
}

# Data sources
data "aws_availability_zones" "available" {
  state = "available"
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.app_name}-vpc-${var.environment}"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.app_name}-igw-${var.environment}"
  }
}

# Public Subnets
resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.${count.index + 10}.0/24"
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.app_name}-public-subnet-${count.index + 1}-${var.environment}"
    Type = "Public"
  }
}

# Private Subnets
resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 1}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "${var.app_name}-private-subnet-${count.index + 1}-${var.environment}"
    Type = "Private"
  }
}

# NAT Gateway EIP
resource "aws_eip" "nat" {
  count  = 2
  domain = "vpc"

  tags = {
    Name = "${var.app_name}-nat-eip-${count.index + 1}-${var.environment}"
  }
}

# NAT Gateways
resource "aws_nat_gateway" "main" {
  count         = 2
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name = "${var.app_name}-nat-${count.index + 1}-${var.environment}"
  }

  depends_on = [aws_internet_gateway.main]
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${var.app_name}-public-rt-${var.environment}"
  }
}

resource "aws_route_table" "private" {
  count  = 2
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = {
    Name = "${var.app_name}-private-rt-${count.index + 1}-${var.environment}"
  }
}

# Route Table Associations
resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = 2
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# Security Groups
resource "aws_security_group" "alb" {
  name        = "${var.app_name}-alb-sg-${var.environment}"
  description = "Security group for Application Load Balancer"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP from anywhere"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS from anywhere"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "Allow all outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.app_name}-alb-sg-${var.environment}"
  }
}

resource "aws_security_group" "ecs_tasks" {
  name        = "${var.app_name}-ecs-tasks-sg-${var.environment}"
  description = "Security group for ECS tasks"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "Allow traffic from ALB"
    from_port       = var.app_port
    to_port         = var.app_port
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    description = "Allow all outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.app_name}-ecs-tasks-sg-${var.environment}"
  }
}

# Application Load Balancer
resource "aws_lb" "main" {
  name               = "${var.app_name}-alb-${var.environment}"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id

  enable_deletion_protection = var.environment == "production" ? true : false
  enable_http2               = true
  enable_cross_zone_load_balancing = true

  tags = {
    Name = "${var.app_name}-alb-${var.environment}"
  }
}

# Target Group
resource "aws_lb_target_group" "app" {
  name        = "${var.app_name}-tg-${var.environment}"
  port        = var.app_port
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = var.health_check_path
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 3
  }

  deregistration_delay = 30

  tags = {
    Name = "${var.app_name}-tg-${var.environment}"
  }
}

# Listener
resource "aws_lb_listener" "app" {
  load_balancer_arn = aws_lb.main.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

# ECR Repository
resource "aws_ecr_repository" "app" {
  name                 = var.app_name
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "AES256"
  }

  tags = {
    Name = "${var.app_name}-ecr-${var.environment}"
  }
}

# ECR Lifecycle Policy
resource "aws_ecr_lifecycle_policy" "app" {
  repository = aws_ecr_repository.app.name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Keep last 10 images"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["v"]
          countType     = "imageCountMoreThan"
          countNumber   = 10
        }
        action = {
          type = "expire"
        }
      },
      {
        rulePriority = 2
        description  = "Remove untagged images after 7 days"
        selection = {
          tagStatus   = "untagged"
          countType   = "sinceImagePushed"
          countUnit   = "days"
          countNumber = 7
        }
        action = {
          type = "expire"
        }
      }
    ]
  })
}

# CloudWatch Log Group
resource "aws_cloudwatch_log_group" "app" {
  name              = "/ecs/${var.app_name}-${var.environment}"
  retention_in_days = 7

  tags = {
    Name = "${var.app_name}-logs-${var.environment}"
  }
}

# ECS Cluster
resource "aws_ecs_cluster" "main" {
  name = "${var.app_name}-cluster-${var.environment}"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  tags = {
    Name = "${var.app_name}-cluster-${var.environment}"
  }
}

# ECS Cluster Capacity Providers
resource "aws_ecs_cluster_capacity_providers" "main" {
  cluster_name = aws_ecs_cluster.main.name

  capacity_providers = ["FARGATE", "FARGATE_SPOT"]

  default_capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 1
    base              = 1
  }

  default_capacity_provider_strategy {
    capacity_provider = "FARGATE_SPOT"
    weight            = 4
  }
}

# IAM Role for ECS Task Execution
resource "aws_iam_role" "ecs_task_execution" {
  name = "${var.app_name}-ecs-task-execution-${var.environment}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      }
    ]
  })

  tags = {
    Name = "${var.app_name}-ecs-task-execution-${var.environment}"
  }
}

resource "aws_iam_role_policy_attachment" "ecs_task_execution" {
  role       = aws_iam_role.ecs_task_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# Additional policy for ECR access
resource "aws_iam_role_policy" "ecs_task_execution_ecr" {
  name = "ecr-access"
  role = aws_iam_role.ecs_task_execution.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage"
        ]
        Resource = "*"
      }
    ]
  })
}

# IAM Role for ECS Task
resource "aws_iam_role" "ecs_task" {
  name = "${var.app_name}-ecs-task-${var.environment}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      }
    ]
  })

  tags = {
    Name = "${var.app_name}-ecs-task-${var.environment}"
  }
}

# Example: Grant S3 access to task role
resource "aws_iam_role_policy" "ecs_task_s3" {
  name = "s3-access"
  role = aws_iam_role.ecs_task.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject"
        ]
        Resource = "arn:aws:s3:::${var.app_name}-${var.environment}-data/*"
      }
    ]
  })
}

# ECS Task Definition
resource "aws_ecs_task_definition" "app" {
  family                   = "${var.app_name}-${var.environment}"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = var.fargate_cpu
  memory                   = var.fargate_memory
  execution_role_arn       = aws_iam_role.ecs_task_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    {
      name      = var.app_name
      image     = "${aws_ecr_repository.app.repository_url}:latest"
      essential = true

      portMappings = [
        {
          containerPort = var.app_port
          protocol      = "tcp"
        }
      ]

      environment = [
        {
          name  = "ENVIRONMENT"
          value = var.environment
        },
        {
          name  = "AWS_REGION"
          value = var.aws_region
        }
      ]

      # Use AWS Secrets Manager for sensitive data
      secrets = [
        # {
        #   name      = "DATABASE_URL"
        #   valueFrom = aws_secretsmanager_secret.db_url.arn
        # }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.app.name
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "ecs"
        }
      }

      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:${var.app_port}${var.health_check_path} || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 60
      }
    }
  ])

  tags = {
    Name = "${var.app_name}-task-def-${var.environment}"
  }
}

# ECS Service
resource "aws_ecs_service" "app" {
  name            = "${var.app_name}-service-${var.environment}"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = var.app_count

  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 1
    base              = 1
  }

  capacity_provider_strategy {
    capacity_provider = "FARGATE_SPOT"
    weight            = 4
  }

  network_configuration {
    subnets          = aws_subnet.private[*].id
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = var.app_name
    container_port   = var.app_port
  }

  deployment_configuration {
    maximum_percent         = 200
    minimum_healthy_percent = 100
  }

  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }

  depends_on = [aws_lb_listener.app]

  tags = {
    Name = "${var.app_name}-service-${var.environment}"
  }
}

# Auto Scaling Target
resource "aws_appautoscaling_target" "ecs" {
  max_capacity       = 10
  min_capacity       = var.app_count
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.app.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

# Auto Scaling Policy - CPU
resource "aws_appautoscaling_policy" "ecs_cpu" {
  name               = "${var.app_name}-cpu-scaling-${var.environment}"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value       = 70.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}

# Auto Scaling Policy - Memory
resource "aws_appautoscaling_policy" "ecs_memory" {
  name               = "${var.app_name}-memory-scaling-${var.environment}"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageMemoryUtilization"
    }
    target_value       = 80.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}

# Outputs
output "alb_dns_name" {
  description = "DNS name of the load balancer"
  value       = aws_lb.main.dns_name
}

output "ecr_repository_url" {
  description = "ECR repository URL"
  value       = aws_ecr_repository.app.repository_url
}

output "ecs_cluster_name" {
  description = "ECS cluster name"
  value       = aws_ecs_cluster.main.name
}

output "ecs_service_name" {
  description = "ECS service name"
  value       = aws_ecs_service.app.name
}

output "cloudwatch_log_group" {
  description = "CloudWatch log group name"
  value       = aws_cloudwatch_log_group.app.name
}
```

---

## Fargate vs Fargate Spot

### Comparison Table

| Feature | Fargate | Fargate Spot |
|---------|---------|--------------|
| **Cost** | Standard pricing | Up to 70% discount |
| **Availability** | Guaranteed | Best-effort (can be interrupted) |
| **Interruption** | None | 2-minute warning before termination |
| **Use Cases** | Production critical workloads | Fault-tolerant, stateless workloads |
| **SLA** | Standard AWS SLA | No SLA |
| **Best For** | APIs, databases, real-time services | Batch jobs, CI/CD, data processing |

### Capacity Provider Strategy

```hcl
# Mix of Fargate and Fargate Spot
resource "aws_ecs_cluster_capacity_providers" "main" {
  cluster_name = aws_ecs_cluster.main.name

  capacity_providers = ["FARGATE", "FARGATE_SPOT"]

  # Strategy 1: Mostly Spot (80% Spot, 20% Regular)
  default_capacity_provider_strategy {
    capacity_provider = "FARGATE_SPOT"
    weight            = 4  # 80% of tasks
  }

  default_capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 1  # 20% of tasks
    base              = 1  # At least 1 task on regular Fargate
  }
}

# Strategy 2: Conservative (50/50 split with guaranteed base)
resource "aws_ecs_service" "app_conservative" {
  # ...
  
  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 1
    base              = 2  # Minimum 2 tasks always on Fargate
  }

  capacity_provider_strategy {
    capacity_provider = "FARGATE_SPOT"
    weight            = 1
  }
}
```

### Handling Spot Interruptions

```python
# app/spot_handler.py
import signal
import sys
import structlog

logger = structlog.get_logger()

def handle_sigterm(signum, frame):
    """
    Handle SIGTERM signal from Fargate Spot interruption.
    You have 120 seconds to gracefully shutdown.
    """
    logger.warning("Received SIGTERM - Fargate Spot interruption", 
                   signal=signum)
    
    # 1. Stop accepting new requests
    # 2. Finish processing current requests
    # 3. Close database connections
    # 4. Flush logs
    # 5. Exit cleanly
    
    sys.exit(0)

signal.signal(signal.SIGTERM, handle_sigterm)
```

---

## Monitoring and Observability

### CloudWatch Dashboards

```hcl
resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = "${var.app_name}-${var.environment}"

  dashboard_body = jsonencode({
    widgets = [
      {
        type = "metric"
        properties = {
          metrics = [
            ["AWS/ECS", "CPUUtilization", { stat = "Average" }],
            [".", "MemoryUtilization", { stat = "Average" }]
          ]
          period = 300
          stat   = "Average"
          region = var.aws_region
          title  = "ECS Resource Utilization"
        }
      },
      {
        type = "metric"
        properties = {
          metrics = [
            ["AWS/ApplicationELB", "TargetResponseTime", { stat = "Average" }],
            [".", "RequestCount", { stat = "Sum" }],
            [".", "HTTPCode_Target_5XX_Count", { stat = "Sum" }]
          ]
          period = 300
          region = var.aws_region
          title  = "ALB Metrics"
        }
      }
    ]
  })
}
```

### CloudWatch Alarms

```hcl
# High CPU Alarm
resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "${var.app_name}-cpu-high-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/ECS"
  period              = "300"
  statistic           = "Average"
  threshold           = "80"
  alarm_description   = "This metric monitors ECS CPU utilization"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    ClusterName = aws_ecs_cluster.main.name
    ServiceName = aws_ecs_service.app.name
  }
}

# High Error Rate Alarm
resource "aws_cloudwatch_metric_alarm" "http_5xx_high" {
  alarm_name          = "${var.app_name}-5xx-high-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "HTTPCode_Target_5XX_Count"
  namespace           = "AWS/ApplicationELB"
  period              = "300"
  statistic           = "Sum"
  threshold           = "10"
  alarm_description   = "High 5XX error rate"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    LoadBalancer = aws_lb.main.arn_suffix
    TargetGroup  = aws_lb_target_group.app.arn_suffix
  }
}

# SNS Topic for Alerts
resource "aws_sns_topic" "alerts" {
  name = "${var.app_name}-alerts-${var.environment}"
}

resource "aws_sns_topic_subscription" "alerts_email" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "email"
  endpoint  = "devops@example.com"
}
```

### Application Performance Monitoring

```python
# app/monitoring.py
from aws_xray_sdk.core import xray_recorder, patch_all
from aws_xray_sdk.ext.flask.middleware import XRayMiddleware
from prometheus_client import Counter, Histogram, Gauge
import time

# Patch libraries for X-Ray tracing
patch_all()

# Prometheus metrics
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['method', 'endpoint']
)

ACTIVE_CONNECTIONS = Gauge(
    'active_connections',
    'Number of active connections'
)

@xray_recorder.capture('process_request')
def process_request():
    start_time = time.time()
    
    # Your business logic
    result = do_work()
    
    # Record metrics
    duration = time.time() - start_time
    REQUEST_LATENCY.labels(method='GET', endpoint='/api/data').observe(duration)
    REQUEST_COUNT.labels(method='GET', endpoint='/api/data', status=200).inc()
    
    return result
```

---

## Cost Optimization

### 1. Right-Sizing Containers

```bash
# Analyze CloudWatch metrics to right-size
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name CPUUtilization \
  --dimensions Name=ServiceName,Value=myapp-service \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-08T00:00:00Z \
  --period 3600 \
  --statistics Average,Maximum
```

### 2. Fargate Spot Savings

```hcl
# Calculate potential savings
# Regular Fargate: $0.04048/vCPU/hour + $0.004445/GB/hour
# Fargate Spot: Up to 70% discount

# Example: 512 CPU (0.5 vCPU), 1024 MB (1 GB)
# Regular: (0.5 * $0.04048) + (1 * $0.004445) = $0.024685/hour
# Spot: $0.024685 * 0.3 = $0.007406/hour
# Savings: $0.017279/hour = ~$12.44/month per task
```

### 3. Reserved Capacity (Savings Plans)

For predictable workloads:
- **Compute Savings Plans**: 1 or 3 year commitment
- Save up to 50% on Fargate costs
- Flexible across instance types and regions

### 4. Auto-Scaling Optimization

```hcl
# Scale down aggressively during off-hours
resource "aws_appautoscaling_scheduled_action" "scale_down_night" {
  name               = "scale-down-night"
  service_namespace  = "ecs"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = "ecs:service:DesiredCount"
  schedule           = "cron(0 22 * * ? *)"  # 10 PM UTC

  scalable_target_action {
    min_capacity = 1
    max_capacity = 3
  }
}

resource "aws_appautoscaling_scheduled_action" "scale_up_morning" {
  name               = "scale-up-morning"
  service_namespace  = "ecs"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = "ecs:service:DesiredCount"
  schedule           = "cron(0 6 * * ? *)"  # 6 AM UTC

  scalable_target_action {
    min_capacity = 3
    max_capacity = 10
  }
}
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. **Service Not Starting**

```bash
# Check service events
aws ecs describe-services \
  --cluster production-cluster \
  --services myapp-service \
  --query 'services[0].events[0:10]'

# Check task stopped reason
aws ecs describe-tasks \
  --cluster production-cluster \
  --tasks <task-arn> \
  --query 'tasks[0].stoppedReason'

# View container logs
aws logs tail /ecs/myapp-production --follow
```

**Common Causes:**
- Insufficient CPU/memory
- Health check failures
- ECR pull permission issues
- Security group misconfiguration

#### 2. **Health Check Failures**

```python
# Ensure /health endpoint is responsive
@app.get("/health")
async def health_check():
    # Check dependencies
    try:
        # Database check
        await db.execute("SELECT 1")
        
        # Redis check
        await redis.ping()
        
        return {"status": "healthy"}
    except Exception as e:
        logger.error("health_check_failed", error=str(e))
        return JSONResponse(
            status_code=503,
            content={"status": "unhealthy", "error": str(e)}
        )
```

#### 3. **OIDC Authentication Errors**

```yaml
# Ensure correct permissions in workflow
permissions:
  id-token: write  # REQUIRED for OIDC
  contents: read

# Verify role trust policy allows your repo
# Condition must match: repo:your-org/your-repo:*
```

#### 4. **Task Definition Update Not Deploying**

```bash
# Force new deployment
aws ecs update-service \
  --cluster production-cluster \
  --service myapp-service \
  --force-new-deployment

# Update service with new task definition
aws ecs update-service \
  --cluster production-cluster \
  --service myapp-service \
  --task-definition myapp:25
```

---

## Advanced Patterns

### 1. Blue/Green Deployment with CodeDeploy

```hcl
# CodeDeploy Application
resource "aws_codedeploy_app" "app" {
  compute_platform = "ECS"
  name             = "${var.app_name}-${var.environment}"
}

# CodeDeploy Deployment Group
resource "aws_codedeploy_deployment_group" "app" {
  app_name               = aws_codedeploy_app.app.name
  deployment_config_name = "CodeDeployDefault.ECSAllAtOnce"
  deployment_group_name  = "${var.app_name}-dg-${var.environment}"
  service_role_arn       = aws_iam_role.codedeploy.arn

  auto_rollback_configuration {
    enabled = true
    events  = ["DEPLOYMENT_FAILURE", "DEPLOYMENT_STOP_ON_ALARM"]
  }

  blue_green_deployment_config {
    deployment_ready_option {
      action_on_timeout = "CONTINUE_DEPLOYMENT"
    }

    terminate_blue_instances_on_deployment_success {
      action                           = "TERMINATE"
      termination_wait_time_in_minutes = 5
    }
  }

  deployment_style {
    deployment_option = "WITH_TRAFFIC_CONTROL"
    deployment_type   = "BLUE_GREEN"
  }

  ecs_service {
    cluster_name = aws_ecs_cluster.main.name
    service_name = aws_ecs_service.app.name
  }

  load_balancer_info {
    target_group_pair_info {
      prod_traffic_route {
        listener_arns = [aws_lb_listener.app.arn]
      }

      target_group {
        name = aws_lb_target_group.blue.name
      }

      target_group {
        name = aws_lb_target_group.green.name
      }
    }
  }
}
```

### 2. Multi-Environment Deployment

```yaml
# .github/workflows/deploy-multi-env.yml
name: Multi-Environment Deployment

on:
  push:
    branches: [develop, staging, main]

jobs:
  set-environment:
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.set-env.outputs.environment }}
    steps:
      - id: set-env
        run: |
          if [ "${{ github.ref }}" == "refs/heads/main" ]; then
            echo "environment=production" >> $GITHUB_OUTPUT
          elif [ "${{ github.ref }}" == "refs/heads/staging" ]; then
            echo "environment=staging" >> $GITHUB_OUTPUT
          else
            echo "environment=development" >> $GITHUB_OUTPUT
          fi

  deploy:
    needs: set-environment
    uses: ./.github/workflows/deploy-template.yml
    with:
      environment: ${{ needs.set-environment.outputs.environment }}
      aws_region: us-east-1
    secrets: inherit
```

### 3. Database Migrations in CI/CD

```yaml
- name: Run Database Migrations
  env:
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
  run: |
    # Run migrations before deployment
    python manage.py migrate
    
    # Verify migrations succeeded
    if [ $? -ne 0 ]; then
      echo "Migration failed!"
      exit 1
    fi
```

### 4. Canary Deployment

```python
# app/feature_flags.py
from fastapi import Request

async def is_canary_user(request: Request) -> bool:
    """Route 10% of traffic to canary deployment"""
    user_id = request.headers.get("X-User-ID")
    if user_id:
        # Consistent hashing for same user
        return int(user_id) % 10 == 0
    return False

@app.get("/api/feature")
async def feature_endpoint(request: Request):
    if await is_canary_user(request):
        return new_feature_implementation()
    else:
        return old_feature_implementation()
```

---

## Summary

This guide covers:

✅ **GitHub Fundamentals** - Actions, workflows, secrets, environments  
✅ **AWS Integration** - OIDC authentication, IAM roles, ECR  
✅ **ECS Architecture** - Task definitions, services, Fargate vs Spot  
✅ **Complete Implementation** - Working code for production deployments  
✅ **Best Practices** - Security, observability, cost optimization  
✅ **Advanced Patterns** - Blue/green, multi-env, canary deployments

---

## Next Steps

1. **Set up your repository** with the provided workflow files
2. **Configure OIDC** between GitHub and AWS
3. **Deploy infrastructure** using Terraform
4. **Test the pipeline** with a sample application
5. **Monitor and optimize** based on metrics

---

## Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [AWS ECS Best Practices](https://docs.aws.amazon.com/AmazonECS/latest/bestpracticesguide/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [OIDC with GitHub Actions](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)

---

**Questions or issues?** Open a GitHub Issue or Discussion.

**License:** MIT

**Maintained by:** DevOps Team
