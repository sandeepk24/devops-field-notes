# GitHub DevOps Fundamentals & Best Practices

> **Part 1 of 3: Understanding GitHub for DevOps and AWS Integration**  
> *A practical guide for DevOps engineers*

[![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=for-the-badge&logo=github-actions&logoColor=white)](https://github.com/features/actions)
[![AWS](https://img.shields.io/badge/AWS-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/)

---

## 📖 Guide Structure

This guide is split into three parts:

1. **Part 1 (This Document)**: GitHub Fundamentals, Features, Best Practices, AWS Integration
2. **Part 2**: Complete Implementation - Code, Terraform, Workflows
3. **Part 3**: Production Deployment - Fargate, Monitoring, Troubleshooting, Advanced Patterns

---

## Table of Contents

- [Introduction](#introduction)
- [Understanding GitHub for DevOps](#understanding-github-for-devops)
- [GitHub CI/CD Features Deep Dive](#github-cicd-features-deep-dive)
- [DevOps Best Practices](#devops-best-practices)
- [AWS Integration Strategies](#aws-integration-strategies)
- [Next Steps](#next-steps)

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

### Why GitHub + AWS?

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

```
Developer Push → Feature Branch → Pull Request → GitHub Actions
                                        ↓
                                   Run Tests
                                   Build Image
                                        ↓
                                   Code Review
                                        ↓
                                  Merge to Main
                                        ↓
                                Deploy to Staging
                                        ↓
                               Manual Approval
                                        ↓
                              Deploy to Production
```

### Repository Structure

```
my-app/
├── .github/
│   ├── workflows/              # CI/CD pipeline definitions
│   │   ├── ci.yml             # Continuous Integration
│   │   ├── deploy-staging.yml
│   │   └── deploy-prod.yml
│   ├── CODEOWNERS             # Auto-assign reviewers
│   └── dependabot.yml         # Automated dependency updates
├── terraform/                 # Infrastructure as Code
│   ├── environments/
│   │   ├── dev/
│   │   ├── staging/
│   │   └── prod/
│   └── modules/
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── src/                       # Application code
├── tests/
├── requirements.txt
└── README.md
```

### Branch Strategy: Trunk-Based Development

**Recommended for Modern DevOps:**

```
main (protected, always deployable)
  ↑
  ├── feature/user-auth (short-lived)
  ├── feature/api-optimization (short-lived)
  └── hotfix/security-patch (urgent)
```

**Best Practices:**
- Keep feature branches short-lived (< 2 days)
- Main branch is always production-ready
- Use feature flags for incomplete features
- Protect main with required reviews and status checks

**Branch Protection Rules:**
```yaml
Required status checks:
  ✓ All tests must pass
  ✓ Code coverage > 80%
  ✓ Security scan passes
  ✓ At least 1 approval required
  ✓ Dismiss stale reviews on new commits
  ✓ Require linear history (no merge commits)
```

---

## GitHub CI/CD Features Deep Dive

### 1. GitHub Actions

**What is it?** An event-driven automation platform integrated directly into GitHub.

#### Anatomy of a Workflow

```yaml
name: CI Pipeline                    # Workflow name displayed in UI
on:                                  # Trigger events
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'             # Daily at 2 AM
  workflow_dispatch:                 # Manual trigger

env:                                 # Global environment variables
  PYTHON_VERSION: '3.11'

jobs:                                # Units of work
  test:                              # Job ID
    name: Run Tests                  # Display name
    runs-on: ubuntu-latest           # Runner environment
    
    steps:                           # Sequential tasks
      - name: Checkout code
        uses: actions/checkout@v4    # Reusable action
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      
      - name: Run tests
        run: pytest tests/           # Shell command
```

#### Workflow Triggers Reference

| Trigger | Use Case | Example |
|---------|----------|---------|
| `push` | Code changes | Deploy on push to main |
| `pull_request` | Code review | Run tests on PR |
| `schedule` | Cron jobs | Nightly builds, cleanup |
| `workflow_dispatch` | Manual | Manual production deploy |
| `release` | Version tagging | Build release artifacts |
| `workflow_call` | Reusable | Shared deployment logic |
| `repository_dispatch` | External API | Trigger from webhook |

#### Matrix Builds

Test across multiple configurations simultaneously:

```yaml
jobs:
  test:
    strategy:
      matrix:
        python-version: ['3.9', '3.10', '3.11', '3.12']
        os: [ubuntu-latest, macos-latest, windows-latest]
    
    runs-on: ${{ matrix.os }}
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: pytest tests/
```

This creates 12 parallel jobs (4 Python versions × 3 operating systems).

### 2. GitHub Environments

Environments provide deployment protection and environment-specific configuration.

```yaml
jobs:
  deploy-production:
    environment: 
      name: production
      url: https://api.example.com    # Deployment URL
    
    steps:
      - name: Deploy
        run: ./deploy.sh
```

**Environment Protection Rules:**

| Protection | Description |
|------------|-------------|
| **Required reviewers** | 1-6 people must approve |
| **Wait timer** | Delay deployment (e.g., 5 minutes) |
| **Deployment branches** | Only deploy from main/release branches |
| **Environment secrets** | Isolated secrets per environment |

**Example Configuration:**
```
Production Environment:
  ✓ Required reviewers: @devops-lead, @tech-lead
  ✓ Wait timer: 300 seconds
  ✓ Deployment branches: main only
  ✓ Secrets: PROD_DATABASE_URL, PROD_API_KEY
```

### 3. Secrets Management

**Three Levels of Secrets:**

```
Organization Secrets (shared across all repos)
    ↓
Repository Secrets (specific to one repo)
    ↓
Environment Secrets (specific to deployment environment)
```

**Using Secrets in Workflows:**

```yaml
steps:
  - name: Deploy
    env:
      AWS_REGION: ${{ secrets.AWS_REGION }}
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
      API_KEY: ${{ secrets.API_KEY }}
    run: ./deploy.sh
```

**Security Best Practices:**

```yaml
# ❌ BAD: Never log secrets
- run: echo "API Key is ${{ secrets.API_KEY }}"

# ✅ GOOD: Use secrets without logging
- run: |
    export API_KEY="${{ secrets.API_KEY }}"
    ./script.sh

# ✅ GOOD: Mask dynamic values
- run: |
    TOKEN=$(generate_token)
    echo "::add-mask::$TOKEN"
    echo "TOKEN=$TOKEN" >> $GITHUB_ENV
```

**Secret Rotation:**
- Rotate secrets every 90 days
- Use OIDC to eliminate long-lived credentials
- Monitor secret usage in audit logs
- Use AWS Secrets Manager for runtime secrets

### 4. Reusable Workflows

Apply DRY principle to your CI/CD:

**Template Workflow:**
```yaml
# .github/workflows/deploy-template.yml
name: Deployment Template

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      aws-region:
        required: false
        type: string
        default: 'us-east-1'
    secrets:
      aws-role:
        required: true
    outputs:
      deployment-url:
        description: "Deployed application URL"
        value: ${{ jobs.deploy.outputs.url }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    outputs:
      url: ${{ steps.deploy.outputs.url }}
    
    steps:
      - name: Deploy to ${{ inputs.environment }}
        id: deploy
        run: |
          echo "Deploying to ${{ inputs.environment }}..."
          echo "url=https://${{ inputs.environment }}.example.com" >> $GITHUB_OUTPUT
```

**Using the Template:**
```yaml
# .github/workflows/deploy-prod.yml
name: Deploy Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    uses: ./.github/workflows/deploy-template.yml
    with:
      environment: production
      aws-region: us-east-1
    secrets:
      aws-role: ${{ secrets.PROD_AWS_ROLE }}
```

### 5. GitHub Container Registry (GHCR)

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
    IMAGE_NAME=ghcr.io/${{ github.repository }}:${{ github.sha }}
    docker build -t $IMAGE_NAME .
    docker push $IMAGE_NAME
```

**Advantages:**
- Free for public repositories
- Integrated with GitHub permissions
- Automatic cleanup policies
- No separate credential management

### 6. Dependabot

Automated dependency updates and security patches:

```yaml
# .github/dependabot.yml
version: 2
updates:
  # Python dependencies
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
    open-pull-requests-limit: 10
    reviewers:
      - "devops-team"
    labels:
      - "dependencies"
      - "python"
    
  # Docker base images
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
    
  # GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    
  # Terraform modules
  - package-ecosystem: "terraform"
    directory: "/terraform"
    schedule:
      interval: "weekly"
```

**Dependabot Features:**
- Automatic security updates
- Version compatibility checks
- Grouped updates for efficiency
- Auto-merge for minor/patch versions

---

## DevOps Best Practices

### 1. The Pipeline Pyramid

Fast feedback first, expensive operations last:

```
                    Production Deploy
                    (slowest, manual approval required)
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

**Time Distribution:**
- Unit Tests: 2-5 minutes (run on every commit)
- Integration Tests: 5-10 minutes (run on PR)
- Security Scans: 3-7 minutes (run on PR)
- Staging Deploy: 10-15 minutes (automatic on merge)
- Production Deploy: 10-15 minutes + manual approval

### 2. Immutable Infrastructure

```yaml
# ❌ BAD: Mutable infrastructure
- name: Deploy
  run: |
    ssh production "cd /app && git pull"
    ssh production "pip install -r requirements.txt"
    ssh production "systemctl restart app"

# ✅ GOOD: Immutable infrastructure
- name: Deploy
  run: |
    # Build new Docker image
    docker build -t myapp:${{ github.sha }} .
    
    # Push to registry
    docker push myapp:${{ github.sha }}
    
    # Update ECS service with new task definition
    aws ecs update-service \
      --cluster production \
      --service api \
      --task-definition api:${{ github.sha }}
```

**Why Immutable Infrastructure?**
- **Reproducible**: Same image in dev, staging, prod
- **Easy Rollback**: Just deploy previous image
- **No Drift**: Configuration can't change over time
- **Audit Trail**: Every deployment has a Git commit

### 3. GitOps Workflow

```
Git Repository (Single Source of Truth)
        ↓
    Git Commit/Push
        ↓
  GitHub Actions (Automation)
        ↓
   Apply Infrastructure Changes
        ↓
   Desired State Achieved
```

**Core Principles:**
1. **Declarative**: Define what you want, not how to get there
2. **Versioned**: All changes tracked in Git
3. **Automated**: No manual kubectl/terraform apply
4. **Auditable**: Git history shows who changed what when

**Example Flow:**
```yaml
# Developer commits to infrastructure repo
git add terraform/production/main.tf
git commit -m "Increase ECS task count from 3 to 5"
git push origin main

# GitHub Actions automatically:
# 1. Plans Terraform changes
# 2. Shows preview in PR
# 3. After approval, applies changes
# 4. Updates infrastructure
```

### 4. Shift-Left Security

Catch security issues early in the development cycle:

```yaml
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      # SAST - Static Application Security Testing
      - name: Run Bandit (Python security linter)
        run: bandit -r src/ -ll
      
      # Dependency scanning
      - name: Check for known vulnerabilities
        run: |
          pip install safety
          safety check --json
      
      # Secret scanning
      - name: Scan for leaked secrets
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.pull_request.base.sha }}
          head: ${{ github.event.pull_request.head.sha }}
      
      # Container scanning
      - name: Scan Docker image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          severity: CRITICAL,HIGH
          exit-code: 1
      
      # Infrastructure scanning
      - name: Terraform security scan
        uses: aquasecurity/tfsec-action@v1.0.0
```

**Security Checklist:**
- [ ] SAST scanning on every PR
- [ ] Dependency vulnerability checks
- [ ] Container image scanning
- [ ] Secret scanning in commits
- [ ] Infrastructure-as-code security checks
- [ ] SBOM (Software Bill of Materials) generation

### 5. Observability from Day One

**The Three Pillars:**

**Logs (Structured JSON):**
```python
import structlog

logger = structlog.get_logger()

logger.info(
    "user_login",
    user_id=123,
    ip="1.2.3.4",
    method="oauth",
    duration_ms=45
)

# Output:
# {"event": "user_login", "user_id": 123, "ip": "1.2.3.4", 
#  "method": "oauth", "duration_ms": 45, "timestamp": "2024-01-15T10:30:00Z"}
```

**Metrics (Prometheus):**
```python
from prometheus_client import Counter, Histogram, Gauge

# Counter: monotonically increasing
login_counter = Counter('user_logins_total', 'Total user logins')
login_counter.inc()

# Histogram: distribution of values
request_latency = Histogram('http_request_duration_seconds', 'HTTP request latency')
with request_latency.time():
    process_request()

# Gauge: current value
active_users = Gauge('active_users', 'Number of active users')
active_users.set(150)
```

**Traces (AWS X-Ray / OpenTelemetry):**
```python
from aws_xray_sdk.core import xray_recorder

@xray_recorder.capture('process_order')
def process_order(order_id):
    # Automatically traces:
    # - Function execution time
    # - Database queries
    # - HTTP calls
    # - Errors and exceptions
    
    with xray_recorder.capture('validate_order'):
        validate(order_id)
    
    with xray_recorder.capture('charge_payment'):
        charge_payment(order_id)
    
    with xray_recorder.capture('send_confirmation'):
        send_email(order_id)
```

### 6. Progressive Deployment Strategies

#### Blue/Green Deployment

```
Step 1: Both versions running
Blue (v1.0): 100% traffic ←─┐
Green (v2.0): 0% traffic     │
                             │
Step 2: Gradually shift      │
Blue (v1.0): 50% traffic     │
Green (v2.0): 50% traffic    │ Monitor metrics
                             │
Step 3: Complete cutover     │
Blue (v1.0): 0% traffic      │
Green (v2.0): 100% traffic ──┘

Step 4: Keep blue for rollback (30 min)
Then terminate old version
```

#### Canary Deployment

```
Step 1: Deploy canary
Stable (v1.0): 95% traffic
Canary (v2.0): 5% traffic ← Monitor for 15 minutes
                            Watch error rates, latency

Step 2: Expand if healthy
Stable (v1.0): 50% traffic
Canary (v2.0): 50% traffic ← Monitor for 30 minutes

Step 3: Full rollout
Stable (v1.0): 0% traffic
Canary (v2.0): 100% traffic (becomes new stable)
```

**Automatic Rollback Conditions:**
- Error rate > 1%
- P95 latency > 500ms
- Health check failures > 10%

---

## AWS Integration Strategies

### OIDC vs IAM Access Keys Comparison

| Feature | IAM Access Keys | OIDC (Recommended) |
|---------|----------------|-------------------|
| **Security** | Long-lived credentials | Short-lived tokens (1 hour) |
| **Storage** | Stored in GitHub Secrets | No credentials stored |
| **Rotation** | Manual every 90 days | Automatic per workflow |
| **Exposure Risk** | High if leaked | Low (expires quickly) |
| **Permissions** | Static IAM user | Dynamic role assumption |
| **Audit Trail** | CloudTrail shows user | CloudTrail shows role + workflow |
| **Setup Complexity** | Simple | Moderate |
| **AWS Best Practice** | ❌ Legacy | ✅ Modern |

### OIDC Setup Guide

#### Architecture Flow

```
GitHub Actions Workflow
        ↓
Generates JWT Token (includes repo, branch, SHA)
        ↓
AWS STS AssumeRoleWithWebIdentity
        ↓
Validates token against OIDC provider
        ↓
Returns Temporary Credentials (expires in 1 hour)
        ↓
GitHub Actions uses credentials for AWS API calls
        ↓
Credentials automatically expire after workflow
```

#### Step 1: Create OIDC Provider in AWS

**Using AWS CLI:**
```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

**Using Terraform:**
```hcl
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"
  
  client_id_list = [
    "sts.amazonaws.com",
  ]
  
  thumbprint_list = [
    "6938fd4d98bab03faadb97b34396831e3780aea1"
  ]
  
  tags = {
    Name = "GitHub Actions OIDC Provider"
  }
}
```

#### Step 2: Create IAM Role with Trust Policy

```hcl
# Trust policy - who can assume this role
data "aws_iam_policy_document" "github_actions_assume_role" {
  statement {
    effect = "Allow"
    
    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }
    
    actions = ["sts:AssumeRoleWithWebIdentity"]
    
    # Validate the audience
    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }
    
    # Only allow specific repository and branch
    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      # repo:ORG/REPO:ref:refs/heads/BRANCH
      values   = [
        "repo:your-org/your-repo:ref:refs/heads/main",
        "repo:your-org/your-repo:ref:refs/heads/develop"
      ]
    }
  }
}

resource "aws_iam_role" "github_actions" {
  name               = "GitHubActionsDeployRole"
  assume_role_policy = data.aws_iam_policy_document.github_actions_assume_role.json
  
  tags = {
    Purpose = "GitHub Actions CI/CD"
  }
}
```

#### Step 3: Attach Permissions to Role

```hcl
# ECR permissions
resource "aws_iam_role_policy_attachment" "ecr" {
  role       = aws_iam_role.github_actions.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser"
}

# Custom ECS deployment policy
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
        Resource = [
          "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
          "arn:aws:iam::123456789012:role/ecsTaskRole"
        ]
        Condition = {
          StringEquals = {
            "iam:PassedToService": "ecs-tasks.amazonaws.com"
          }
        }
      }
    ]
  })
}

# CloudWatch Logs (for viewing deployment logs)
resource "aws_iam_role_policy" "logs" {
  name = "CloudWatchLogsAccess"
  role = aws_iam_role.github_actions.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents",
          "logs:DescribeLogStreams"
        ]
        Resource = "arn:aws:logs:*:*:log-group:/ecs/*"
      }
    ]
  })
}
```

#### Step 4: Use OIDC in GitHub Actions

```yaml
name: Deploy with OIDC

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    # REQUIRED: Grant permissions for OIDC
    permissions:
      id-token: write   # Required to request JWT
      contents: read    # Required to checkout code
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Configure AWS credentials using OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsDeployRole
          aws-region: us-east-1
          role-session-name: GitHubActions-${{ github.run_id }}
      
      - name: Verify AWS credentials
        run: |
          aws sts get-caller-identity
          echo "Successfully authenticated with AWS using OIDC!"
      
      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Build and push Docker image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/myapp:$IMAGE_TAG .
          docker push $ECR_REGISTRY/myapp:$IMAGE_TAG
```

### Security Best Practices for OIDC

**1. Restrict by Repository and Branch:**
```hcl
condition {
  test     = "StringLike"
  variable = "token.actions.githubusercontent.com:sub"
  # Only allow main and release branches
  values   = [
    "repo:your-org/your-repo:ref:refs/heads/main",
    "repo:your-org/your-repo:ref:refs/heads/release/*"
  ]
}
```

**2. Restrict by Environment:**
```hcl
condition {
  test     = "StringEquals"
  variable = "token.actions.githubusercontent.com:sub"
  # Only allow production environment
  values   = ["repo:your-org/your-repo:environment:production"]
}
```

**3. Least Privilege Permissions:**
```hcl
# Create separate roles for different workflows
resource "aws_iam_role" "github_ci" {
  name = "GitHubActions-CI"
  # Only ECR read/write, no deployment permissions
}

resource "aws_iam_role" "github_deploy" {
  name = "GitHubActions-Deploy"
  # Full deployment permissions
}
```

**4. Monitor and Audit:**
```bash
# CloudTrail query to see all OIDC authentications
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceType,AttributeValue=AWS::IAM::Role \
  --max-results 50 \
  --query 'Events[?contains(CloudTrailEvent, `AssumeRoleWithWebIdentity`)]'
```

### Legacy: IAM Access Keys (Not Recommended)

If you must use access keys:

```yaml
- name: Configure AWS Credentials (Access Keys)
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-1
```

**Required GitHub Secrets:**
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`

**Risks:**
- Credentials stored in GitHub
- Long-lived (until rotated manually)
- If leaked, attacker has persistent access
- Harder to scope permissions

---

## Next Steps

Continue to **Part 2: Complete Implementation** for:
- Production-ready FastAPI application
- Dockerfiles with multi-stage builds
- Complete GitHub Actions CI/CD workflows
- Full Terraform infrastructure code
- ECS Task Definitions and Services

---

## Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [OIDC with GitHub Actions](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
- [AWS ECS Best Practices](https://docs.aws.amazon.com/AmazonECS/latest/bestpracticesguide/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)

---

**Part 1 of 3 Complete** | [Continue to Part 2 →](./02_GITHUB_DEVOPS_IMPLEMENTATION.md)
