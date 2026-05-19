# ECS Cluster Reliability: Best Practices and Proactive Monitoring

*A practical guide to building bulletproof ECS clusters and catching problems before they become incidents*

---

## Introduction

You know that sinking feeling when you get paged at 2 AM because your ECS tasks are stuck in a restart loop? Or when you discover your cluster has been quietly failing to launch tasks for the past hour because you hit an AWS service quota?

After years of running ECS in production (and learning many lessons the hard way), I've compiled the best practices that actually matter. This isn't a theoretical checklist—these are battle-tested strategies that keep services running and let you sleep at night.

Let's build a cluster that just works.

---

## Table of Contents

1. [Pre-Flight: Design Decisions That Matter](#pre-flight-design-decisions-that-matter)
2. [Building a Resilient Cluster](#building-a-resilient-cluster)
3. [Service Configuration Best Practices](#service-configuration-best-practices)
4. [The Monitoring Stack You Actually Need](#the-monitoring-stack-you-actually-need)
5. [Proactive Health Checks](#proactive-health-checks)
6. [Alerting That Works](#alerting-that-works)
7. [Common Failure Modes and How to Prevent Them](#common-failure-modes-and-how-to-prevent-them)
8. [Day 2 Operations](#day-2-operations)
9. [The War Room Dashboard](#the-war-room-dashboard)
10. [Troubleshooting Playbook](#troubleshooting-playbook)

---

## Pre-Flight: Design Decisions That Matter

Before you create your first cluster, these architectural decisions will determine how well your system handles failures.

### 1. Choosing Launch Type: EC2 vs Fargate

**Decision Matrix**:

```
Use Fargate when:
✅ You want to focus on apps, not infrastructure
✅ Your workload is unpredictable or spiky
✅ You value operational simplicity over cost optimization
✅ You need fast, predictable scaling
✅ Your team is small or new to AWS

Use EC2 when:
✅ You have steady-state, predictable workloads
✅ You need GPU instances or specific instance types
✅ You're optimizing for cost with Reserved Instances
✅ You need instance-level customization (kernel tuning, custom AMIs)
✅ You have capacity to manage infrastructure
```

**My Recommendation**: Start with Fargate unless you have a specific reason not to. You can always migrate to EC2 later when you understand your workload patterns.

### 2. Network Mode Selection

This matters more than most people realize.

**awsvpc Mode** (Required for Fargate, recommended for EC2):
```
Pros:
- Each task gets its own ENI and private IP
- Security groups per task (finest-grained security)
- No port conflicts
- Full VPC networking features

Cons:
- ENI limits per instance (EC2)
- More IP addresses consumed
- Slightly higher latency (negligible)
```

**Bridge Mode** (EC2 only):
```
Pros:
- One ENI per instance (saves IPs)
- Better instance utilization
- Faster container startup (no ENI attachment)

Cons:
- Port mapping complexity
- Port conflicts possible
- Security groups at instance level only
- Not compatible with many AWS integrations
```

**Best Practice**: Use `awsvpc` mode. The operational simplicity is worth the ENI overhead.

### 3. Subnet and AZ Strategy

**The Golden Rule**: Always use at least 3 Availability Zones.

Why? Math:

```
With 2 AZs:
- One AZ fails → 50% capacity loss
- Need to run 2x capacity to survive failure
- Inefficient and expensive

With 3 AZs:
- One AZ fails → 33% capacity loss
- Run 1.5x minimum capacity to survive failure
- Much more efficient
- Survives any single AZ failure
```

**Subnet Configuration**:
```hcl
# Terraform example
resource "aws_ecs_service" "api" {
  # ... other config ...
  
  network_configuration {
    subnets = [
      aws_subnet.private_us_east_1a.id,
      aws_subnet.private_us_east_1b.id,
      aws_subnet.private_us_east_1c.id,
    ]
    security_groups = [aws_security_group.api_tasks.id]
  }
}
```

**Private vs Public Subnets**:
- **Private subnets** (recommended): Tasks in private subnets with NAT Gateway
- **Public subnets** (rarely needed): Only if tasks need public IPs (unusual)

---

## Building a Resilient Cluster

### 1. Cluster Capacity Planning (EC2 Launch Type)

If you're using EC2, capacity planning is critical. Here's the formula:

```
Required Instances = (Total Task Resources × Safety Buffer) / Instance Resources

Example:
- 20 tasks × 1024 CPU × 2048 MB each
- Safety buffer: 30% (1.3x)
- Instance type: c5.xlarge (4096 CPU, 8192 MB)

Total needed:
- CPU: 20 × 1024 × 1.3 = 26,624 CPU units
- Memory: 20 × 2048 × 1.3 = 53,248 MB

Instances needed:
- CPU: 26,624 / 4096 = 6.5 instances
- Memory: 53,248 / 8192 = 6.5 instances
- Round up: 7 instances minimum
```

**Best Practice**: Always round up and add 1-2 extra instances. Better to have extra capacity than tasks stuck in PENDING.

### 2. Auto Scaling Setup (EC2)

Don't rely on manual scaling. Set up both levels:

**Level 1: ECS Service Auto Scaling** (scales tasks)
```json
{
  "targetTrackingScalingPolicies": [
    {
      "policyName": "cpu-scaling",
      "targetValue": 70.0,
      "predefinedMetricSpecification": {
        "predefinedMetricType": "ECSServiceAverageCPUUtilization"
      },
      "scaleInCooldown": 300,
      "scaleOutCooldown": 60
    }
  ]
}
```

**Level 2: Capacity Provider Auto Scaling** (scales EC2 instances)
```json
{
  "capacityProviders": [
    {
      "name": "my-capacity-provider",
      "autoScalingGroupProvider": {
        "autoScalingGroupArn": "arn:aws:autoscaling:...",
        "managedScaling": {
          "status": "ENABLED",
          "targetCapacity": 80,
          "minimumScalingStepSize": 1,
          "maximumScalingStepSize": 10
        }
      }
    }
  ]
}
```

**Critical**: Set appropriate cooldown periods. Too short = thrashing. Too long = slow response.

### 3. Task Placement Strategies

Tell ECS how to distribute tasks for maximum resilience:

```json
{
  "placementStrategy": [
    {
      "type": "spread",
      "field": "attribute:ecs.availability-zone"
    },
    {
      "type": "spread",
      "field": "instanceId"
    }
  ]
}
```

**What this does**:
1. First, spread across AZs (survive AZ failure)
2. Then, spread across instances (survive instance failure)

**Alternative strategies**:
- `binpack`: Pack tasks tightly (cost optimization, but less resilient)
- `random`: Random placement (use when other strategies don't matter)

**My Recommendation**: Always use the double-spread strategy above for production services.

### 4. Fargate Ephemeral Storage Configuration

If you're using Fargate and your app writes logs or temp files:

```json
{
  "family": "my-task",
  "ephemeralStorage": {
    "sizeInGiB": 30
  }
}
```

Default is 20 GB. Monitor disk usage—tasks get killed when disk fills up.

---

## Service Configuration Best Practices

### 1. Task Definition Essentials

**Resource Allocation**:
```json
{
  "family": "api-server",
  "cpu": "1024",
  "memory": "2048",
  "containerDefinitions": [
    {
      "name": "nginx",
      "image": "nginx:1.21",
      "cpu": 256,
      "memory": 512,
      "memoryReservation": 256,
      "essential": true,
      "portMappings": [{"containerPort": 80}]
    },
    {
      "name": "app",
      "image": "my-app:v1.2.3",
      "cpu": 768,
      "memory": 1536,
      "memoryReservation": 1024,
      "essential": true,
      "portMappings": [{"containerPort": 8080}]
    }
  ]
}
```

**Key Points**:
- **cpu**: Task-level CPU is the sum of all container CPU values
- **memory**: Hard limit—task gets OOM-killed if exceeded
- **memoryReservation**: Soft limit—guaranteed minimum
- **essential**: If true and container dies, the entire task stops

**Best Practice**: Set `memoryReservation` to 70-80% of `memory`. This gives headroom for spikes.

### 2. Logging Configuration (Don't Skip This!)

**Always configure logging**. When things break, logs are your lifeline.

```json
{
  "logConfiguration": {
    "logDriver": "awslogs",
    "options": {
      "awslogs-group": "/ecs/my-api",
      "awslogs-region": "us-east-1",
      "awslogs-stream-prefix": "api",
      "awslogs-create-group": "true"
    }
  }
}
```

**Create a log retention policy**:
```python
import boto3

logs = boto3.client('logs')

logs.put_retention_policy(
    logGroupName='/ecs/my-api',
    retentionInDays=30  # or 7, 14, 60, 90, etc.
)
```

**Why this matters**: Logs cost money to store. Without retention policies, they accumulate forever. I've seen log bills hit $5,000/month.

### 3. Health Checks (The Most Important Config)

**Application Load Balancer Health Check**:
```json
{
  "healthCheckPath": "/health",
  "healthCheckIntervalSeconds": 30,
  "healthCheckTimeoutSeconds": 5,
  "healthyThresholdCount": 2,
  "unhealthyThresholdCount": 3,
  "matcher": "200-299"
}
```

**ECS Service Health Check Grace Period**:
```json
{
  "healthCheckGracePeriodSeconds": 60
}
```

**Health Check Endpoint Best Practices**:

Your `/health` endpoint should check:
```python
from flask import Flask, jsonify
import psycopg2

app = Flask(__name__)

@app.route('/health')
def health():
    checks = {}
    overall_status = "healthy"
    
    # Check database connection
    try:
        conn = psycopg2.connect(DATABASE_URL)
        conn.close()
        checks['database'] = 'healthy'
    except:
        checks['database'] = 'unhealthy'
        overall_status = "unhealthy"
    
    # Check disk space
    import shutil
    disk = shutil.disk_usage('/')
    if disk.free / disk.total < 0.1:  # Less than 10% free
        checks['disk'] = 'unhealthy'
        overall_status = "unhealthy"
    else:
        checks['disk'] = 'healthy'
    
    # Check memory pressure
    import psutil
    if psutil.virtual_memory().percent > 90:
        checks['memory'] = 'unhealthy'
        overall_status = "unhealthy"
    else:
        checks['memory'] = 'healthy'
    
    status_code = 200 if overall_status == "healthy" else 503
    
    return jsonify({
        'status': overall_status,
        'checks': checks
    }), status_code

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

**Tuning Health Checks**:

```
Detection Time = (unhealthyThresholdCount × intervalSeconds)

Example:
- Interval: 30 seconds
- Unhealthy threshold: 3
- Detection time: 90 seconds

Too sensitive (bad):
- Interval: 5 seconds, Threshold: 1 = 5 second detection
- Problem: False positives from brief slowdowns

Too lenient (bad):
- Interval: 60 seconds, Threshold: 5 = 300 second detection
- Problem: Unhealthy tasks serve traffic for 5 minutes

Just right:
- Interval: 30 seconds, Threshold: 2-3 = 60-90 second detection
- Balances false positives vs quick detection
```

### 4. Deployment Configuration

**Rolling Update Settings**:
```json
{
  "deploymentConfiguration": {
    "maximumPercent": 200,
    "minimumHealthyPercent": 100,
    "deploymentCircuitBreaker": {
      "enable": true,
      "rollback": true
    }
  }
}
```

**What these numbers mean**:

```
Desired tasks: 10

maximumPercent: 200
- During deployment, can run up to 20 tasks (10 × 200%)
- ECS launches 10 new tasks before stopping old ones
- Zero-downtime deployment

minimumHealthyPercent: 100
- Must keep at least 10 healthy tasks (10 × 100%)
- Never go below desired count
- Safest option

deploymentCircuitBreaker:
- Automatically rolls back if deployment fails
- Saves you from bad deployments at 3 AM
```

**Alternative configurations**:

```
minimumHealthyPercent: 50, maximumPercent: 100
- Cost-optimized (no extra tasks during deployment)
- Briefly runs at half capacity
- Use for non-critical services

minimumHealthyPercent: 100, maximumPercent: 150
- Balanced approach
- Minimal extra capacity during deployment
- Good default for most services
```

### 5. Service Discovery Integration

If services need to talk to each other:

```json
{
  "serviceRegistries": [
    {
      "registryArn": "arn:aws:servicediscovery:us-east-1:...:service/srv-abc123",
      "containerName": "app",
      "containerPort": 8080
    }
  ]
}
```

Now other services can reach yours via DNS:
```
api-server.production.local
```

**Benefits**:
- No hardcoded IPs
- Automatic updates when tasks scale
- Works across VPCs with proper DNS setup

---

## The Monitoring Stack You Actually Need

Let's build a monitoring system that catches problems before users do.

### 1. Essential CloudWatch Metrics

**Service-Level Metrics** (Check these daily):

```
Metric: CPUUtilization
What it tells you: How hard your tasks are working
Normal range: 30-70%
Warning: >80% sustained
Critical: >90% sustained
Action: Scale up or optimize code

Metric: MemoryUtilization
What it tells you: Memory pressure
Normal range: 40-70%
Warning: >80%
Critical: >90%
Action: Scale up or increase task memory

Metric: RunningTaskCount
What it tells you: How many tasks are actually running
Normal: = DesiredTaskCount
Warning: < DesiredTaskCount for >5 minutes
Critical: < 50% of DesiredTaskCount
Action: Check why tasks won't start (capacity, errors, etc.)

Metric: DesiredTaskCount
What it tells you: How many tasks should be running
Monitor: Track changes (auto-scaling activity)
Action: Verify auto-scaling is working as expected
```

**Target Group Metrics** (If using ALB):

```
Metric: HealthyHostCount
What it tells you: How many targets are passing health checks
Normal: = RunningTaskCount
Warning: <90% of RunningTaskCount
Critical: <50% of RunningTaskCount
Action: Check task health, application errors

Metric: UnhealthyHostCount
What it tells you: Targets failing health checks
Normal: 0
Warning: >0 for >5 minutes
Critical: >20% of total targets
Action: Check logs, health check config

Metric: TargetResponseTime
What it tells you: API latency
Normal: <500ms (varies by app)
Warning: >1 second
Critical: >3 seconds or increasing trend
Action: Check for slow queries, external API issues

Metric: RequestCountPerTarget
What it tells you: Load distribution
Monitor: Should be roughly equal across targets
Warning: One target >2x others
Action: Check for misbehaving task or sticky sessions issue
```

**Cluster-Level Metrics** (EC2 only):

```
Metric: CPUReservation
What it tells you: % of cluster CPU reserved by tasks
Normal: 60-80%
Warning: >85%
Critical: >95%
Action: Add more instances or capacity provider will scale

Metric: MemoryReservation
What it tells you: % of cluster memory reserved by tasks
Normal: 60-80%
Warning: >85%
Critical: >95%
Action: Add more instances

Metric: RegisteredContainerInstancesCount
What it tells you: How many EC2 instances in cluster
Monitor: Should match ASG desired count
Warning: Mismatch for >10 minutes
Action: Check ECS agent, ASG health
```

### 2. Custom Metrics (Build These)

**Task Placement Failures**:
```python
import boto3
import time

cloudwatch = boto3.client('cloudwatch')
ecs = boto3.client('ecs')

def monitor_task_placement():
    events = ecs.describe_services(
        cluster='production',
        services=['api-service']
    )
    
    for event in events['services'][0]['events']:
        if 'unable to place task' in event['message'].lower():
            # Found a placement failure
            cloudwatch.put_metric_data(
                Namespace='CustomECS',
                MetricData=[
                    {
                        'MetricName': 'TaskPlacementFailures',
                        'Value': 1,
                        'Unit': 'Count',
                        'Timestamp': time.time()
                    }
                ]
            )
            
            print(f"ALERT: Task placement failure: {event['message']}")

# Run every minute
while True:
    monitor_task_placement()
    time.sleep(60)
```

**Task Churn Rate**:
```python
def calculate_task_churn():
    """Track how often tasks are restarting"""
    tasks = ecs.list_tasks(
        cluster='production',
        serviceName='api-service'
    )
    
    task_details = ecs.describe_tasks(
        cluster='production',
        tasks=tasks['taskArns']
    )
    
    restarts_last_hour = 0
    one_hour_ago = time.time() - 3600
    
    for task in task_details['tasks']:
        created_at = task['createdAt'].timestamp()
        if created_at > one_hour_ago:
            restarts_last_hour += 1
    
    cloudwatch.put_metric_data(
        Namespace='CustomECS',
        MetricData=[
            {
                'MetricName': 'TaskChurnRate',
                'Value': restarts_last_hour,
                'Unit': 'Count'
            }
        ]
    )
    
    # Alert if >10 tasks restarted in past hour
    if restarts_last_hour > 10:
        print(f"HIGH CHURN: {restarts_last_hour} tasks restarted in past hour")
```

**Deployment Success Rate**:
```python
def track_deployment_success():
    deployments = ecs.describe_services(
        cluster='production',
        services=['api-service']
    )['services'][0]['deployments']
    
    for deployment in deployments:
        if deployment['status'] == 'PRIMARY':
            # Check if deployment reached desired count
            desired = deployment['desiredCount']
            running = deployment['runningCount']
            
            success = 1 if running == desired else 0
            
            cloudwatch.put_metric_data(
                Namespace='CustomECS',
                MetricData=[
                    {
                        'MetricName': 'DeploymentSuccess',
                        'Value': success,
                        'Unit': 'None'
                    }
                ]
            )
```

### 3. Log-Based Metrics

Extract metrics from your application logs:

**CloudWatch Logs Metric Filter**:
```json
{
  "filterName": "ErrorCount",
  "filterPattern": "[time, request_id, level = ERROR, ...]",
  "metricTransformations": [
    {
      "metricName": "ApplicationErrors",
      "metricNamespace": "MyApp",
      "metricValue": "1",
      "defaultValue": 0
    }
  ]
}
```

**Common patterns to track**:
- `ERROR` log lines → Application error rate
- `OutOfMemoryError` → Memory pressure
- `Connection timeout` → External dependency issues
- `Task timed out` → Lambda timeout (if using Lambda with ECS)
- `429` status codes → Rate limiting issues

---

## Proactive Health Checks

Don't wait for things to break. Build checks that catch issues early.

### 1. Synthetic Monitoring

**External Health Check Script** (run from outside AWS):

```python
import requests
import time

def synthetic_health_check():
    endpoints = [
        'https://api.example.com/health',
        'https://api.example.com/v1/users',
        'https://api.example.com/v1/orders'
    ]
    
    for endpoint in endpoints:
        start = time.time()
        
        try:
            response = requests.get(endpoint, timeout=5)
            latency = (time.time() - start) * 1000  # ms
            
            # Send metrics to CloudWatch
            cloudwatch.put_metric_data(
                Namespace='SyntheticMonitoring',
                MetricData=[
                    {
                        'MetricName': 'EndpointAvailability',
                        'Dimensions': [
                            {'Name': 'Endpoint', 'Value': endpoint}
                        ],
                        'Value': 1 if response.status_code == 200 else 0,
                        'Unit': 'None'
                    },
                    {
                        'MetricName': 'EndpointLatency',
                        'Dimensions': [
                            {'Name': 'Endpoint', 'Value': endpoint}
                        ],
                        'Value': latency,
                        'Unit': 'Milliseconds'
                    }
                ]
            )
            
            if response.status_code != 200:
                print(f"ALERT: {endpoint} returned {response.status_code}")
            
            if latency > 1000:
                print(f"ALERT: {endpoint} slow response: {latency:.0f}ms")
                
        except requests.exceptions.Timeout:
            print(f"ALERT: {endpoint} timed out")
            cloudwatch.put_metric_data(
                Namespace='SyntheticMonitoring',
                MetricData=[{
                    'MetricName': 'EndpointAvailability',
                    'Dimensions': [{'Name': 'Endpoint', 'Value': endpoint}],
                    'Value': 0
                }]
            )
        except Exception as e:
            print(f"ALERT: {endpoint} error: {str(e)}")

# Run every minute from outside AWS (different region or on-prem)
while True:
    synthetic_health_check()
    time.sleep(60)
```

**Why this matters**: AWS internal monitoring might miss external connectivity issues (DNS, CloudFront, etc.)

### 2. Capacity Headroom Checks

**Proactive Capacity Alert**:
```python
def check_capacity_headroom():
    """Alert before you run out of capacity"""
    
    # Get cluster stats (EC2)
    cluster = ecs.describe_clusters(
        clusters=['production'],
        include=['STATISTICS']
    )['clusters'][0]
    
    cpu_reservation = next(
        s['value'] for s in cluster['statistics'] 
        if s['name'] == 'CPUReservation'
    )
    
    mem_reservation = next(
        s['value'] for s in cluster['statistics'] 
        if s['name'] == 'MemoryReservation'
    )
    
    # Alert at 85% (before you hit 100%)
    if cpu_reservation > 85:
        print(f"WARNING: CPU reservation at {cpu_reservation}%")
        print("Action: Scale up cluster before hitting capacity limit")
    
    if mem_reservation > 85:
        print(f"WARNING: Memory reservation at {mem_reservation}%")
        print("Action: Scale up cluster before hitting capacity limit")
    
    # For Fargate, check service limits
    quotas = boto3.client('service-quotas')
    
    try:
        fargate_quota = quotas.get_service_quota(
            ServiceCode='fargate',
            QuotaCode='L-3032A538'  # Fargate On-Demand vCPU resource count
        )
        
        # Get current usage (would need custom tracking)
        # Alert at 80% of quota
        
    except:
        pass
```

### 3. Service Limit Checks

AWS has limits that can bite you:

```python
def check_service_limits():
    """Check if you're approaching AWS limits"""
    
    limits_to_check = {
        'ecs': [
            ('clusters-per-account', 10000),
            ('services-per-cluster', 2000),
            ('tasks-per-service', 10000),
            ('container-instances-per-cluster', 5000)
        ],
        'ec2': [
            ('vpc-elastic-ips', 5),
            ('vpc-security-groups', 2500),
            ('instances-per-region', 'varies')
        ]
    }
    
    # Get current counts
    clusters = ecs.list_clusters()
    cluster_count = len(clusters['clusterArns'])
    
    if cluster_count > 8000:  # 80% of limit
        print(f"WARNING: Using {cluster_count}/10000 clusters")
    
    # Check ENI limits (critical for awsvpc mode)
    ec2 = boto3.client('ec2')
    enis = ec2.describe_network_interfaces()
    
    eni_count = len(enis['NetworkInterfaces'])
    eni_limit = 5000  # Default limit, check your account
    
    if eni_count > (eni_limit * 0.8):
        print(f"WARNING: Using {eni_count}/{eni_limit} ENIs")
        print("This affects awsvpc networking mode")
```

---

## Alerting That Works

Not all alerts are created equal. Here's how to build an alerting strategy that doesn't cause burnout.

### Alert Severity Levels

**P1 - Critical** (Page immediately, 24/7):
```
- Service completely down (0 healthy tasks)
- >50% error rate
- Database unreachable
- Complete AZ failure
```

**P2 - High** (Page during business hours):
```
- Service degraded (>30% unhealthy tasks)
- 10-50% error rate
- Single AZ failure
- Deployment failing
```

**P3 - Medium** (Slack notification):
```
- Elevated error rate (5-10%)
- Latency increasing
- CPU/Memory >80%
- Task churn elevated
```

**P4 - Low** (Email, daily digest):
```
- Cost anomalies
- Approaching service limits
- Non-critical config warnings
```

### CloudWatch Alarm Examples

**Critical: Service Down**
```json
{
  "AlarmName": "api-service-down",
  "MetricName": "HealthyHostCount",
  "Namespace": "AWS/ApplicationELB",
  "Statistic": "Average",
  "Period": 60,
  "EvaluationPeriods": 2,
  "Threshold": 1,
  "ComparisonOperator": "LessThanThreshold",
  "AlarmActions": [
    "arn:aws:sns:us-east-1:123456789:critical-alerts"
  ],
  "TreatMissingData": "breaching"
}
```

**Warning: High CPU**
```json
{
  "AlarmName": "api-service-high-cpu",
  "MetricName": "CPUUtilization",
  "Namespace": "AWS/ECS",
  "Dimensions": [
    {"Name": "ServiceName", "Value": "api-service"},
    {"Name": "ClusterName", "Value": "production"}
  ],
  "Statistic": "Average",
  "Period": 300,
  "EvaluationPeriods": 2,
  "Threshold": 80,
  "ComparisonOperator": "GreaterThanThreshold",
  "AlarmActions": [
    "arn:aws:sns:us-east-1:123456789:warning-alerts"
  ]
}
```

**Composite Alarm** (reduces noise):
```json
{
  "AlarmName": "api-service-degraded",
  "AlarmRule": "(ALARM(high-cpu-alarm) OR ALARM(high-memory-alarm)) AND ALARM(elevated-errors-alarm)",
  "AlarmActions": [
    "arn:aws:sns:us-east-1:123456789:critical-alerts"
  ]
}
```

This only fires when BOTH resource pressure AND errors occur (not just high CPU alone).

---

## Common Failure Modes and How to Prevent Them

Let me share the issues I've seen (and caused) in production, and how to avoid them.

### Failure Mode 1: Tasks Stuck in PENDING

**Symptoms**:
```
DesiredTaskCount: 10
RunningTaskCount: 5
PendingTaskCount: 5 (stuck here for >5 minutes)
```

**Common Causes**:

**A) Insufficient Cluster Capacity** (EC2):
```
Error in events: "unable to place a task because no container instance met all of its requirements"

Solution:
- Check CPUReservation and MemoryReservation metrics
- Add more instances or enable capacity provider auto-scaling
- Reduce task resource requirements
```

**B) ENI Limit Exceeded** (awsvpc mode):
```
Error: "network interface limit exceeded"

Solution:
- Each instance type has ENI limits (t3.small: 3 ENIs, m5.large: 10 ENIs)
- Either use larger instances or bridge networking
- Request ENI limit increase
```

**C) Insufficient IP Addresses**:
```
Error: "no IP addresses available in subnet"

Solution:
- Each task in awsvpc needs an IP from your VPC
- Use larger subnets (/24 = 256 IPs, /23 = 512 IPs)
- Add more subnets across AZs
```

**D) Image Pull Failures**:
```
Error: "CannotPullContainerError"

Solution:
- Check ECR permissions (task execution role)
- Verify image exists and tag is correct
- Check network connectivity (NAT Gateway working?)
- Enable ECR VPC endpoint for faster pulls
```

**Prevention**:
```python
# Daily capacity check
def check_pending_tasks():
    services = ecs.describe_services(
        cluster='production',
        services=ecs.list_services(cluster='production')['serviceArns']
    )
    
    for service in services['services']:
        desired = service['desiredCount']
        running = service['runningCount']
        pending = desired - running
        
        if pending > 0:
            print(f"WARNING: {service['serviceName']} has {pending} pending tasks")
            
            # Check recent events
            for event in service['events'][:5]:
                if 'unable to place' in event['message'].lower():
                    print(f"  Reason: {event['message']}")
```

### Failure Mode 2: Tasks Constantly Restarting

**Symptoms**:
```
Tasks starting and stopping every 30-60 seconds
HealthyHostCount fluctuating
High TaskChurnRate
```

**Common Causes**:

**A) Application Crashes**:
```
Check logs for:
- OutOfMemoryError
- Segmentation fault
- Uncaught exceptions

Solution:
- Increase task memory
- Fix application bugs
- Add proper error handling
```

**B) Failing Health Checks**:
```
Task starts → Health check fails → Task killed → Repeat

Solution:
- Verify health check endpoint works
- Increase healthCheckGracePeriodSeconds (app startup time)
- Fix dependency issues (DB connection, etc.)
```

**C) OOM Kills**:
```
Check task stopped reason: "OutOfMemoryError: Container killed due to memory usage"

Solution:
- Increase task memory allocation
- Fix memory leaks in application
- Add memory profiling to find the leak
```

**Prevention**:
```python
# Monitor task stop reasons
def analyze_task_stops():
    tasks = ecs.list_tasks(
        cluster='production',
        serviceName='api-service',
        desiredStatus='STOPPED',
        maxResults=10
    )
    
    task_details = ecs.describe_tasks(
        cluster='production',
        tasks=tasks['taskArns']
    )
    
    stop_reasons = {}
    
    for task in task_details['tasks']:
        reason = task.get('stoppedReason', 'Unknown')
        stop_reasons[reason] = stop_reasons.get(reason, 0) + 1
    
    # Alert on patterns
    for reason, count in stop_reasons.items():
        if count >= 5:
            print(f"PATTERN: {count} tasks stopped due to: {reason}")
```

### Failure Mode 3: Deployment Failures

**Symptoms**:
```
Deployment stuck
Circuit breaker triggered
Old tasks still running after deployment
```

**Common Causes**:

**A) New Tasks Fail Health Checks**:
```
New revision tasks start → Fail health checks → Get killed → Repeat

Solution:
- Test health check endpoint before deploying
- Increase grace period for slower-starting apps
- Check for breaking changes in new code
```

**B) Resource Requirements Changed**:
```
New task definition needs more CPU/memory than available

Solution:
- Verify cluster has capacity for new requirements
- Scale cluster before deploying
- Use deployment circuit breaker (auto-rollback)
```

**Prevention**:
```python
# Pre-deployment validation
def validate_deployment(new_task_def):
    """Run checks before deploying"""
    
    # 1. Can cluster fit new tasks?
    cluster_stats = ecs.describe_clusters(
        clusters=['production'],
        include=['STATISTICS']
    )
    
    cpu_available = 100 - cluster_stats['clusters'][0]['statistics']['CPUReservation']
    mem_available = 100 - cluster_stats['clusters'][0]['statistics']['MemoryReservation']
    
    new_task_cpu = int(new_task_def['cpu'])
    new_task_mem = int(new_task_def['memory'])
    
    # Need 20% buffer
    if cpu_available < 20 or mem_available < 20:
        print("ERROR: Insufficient capacity for safe deployment")
        return False
    
    # 2. Is the image pullable?
    try:
        ecr = boto3.client('ecr')
        image_uri = new_task_def['containerDefinitions'][0]['image']
        # Verify image exists...
    except:
        print("ERROR: Cannot verify image exists")
        return False
    
    print("✓ Deployment pre-checks passed")
    return True
```

### Failure Mode 4: Silent Degradation

**Symptoms**:
```
Service appears healthy
All metrics green
But users report slow responses or errors
```

**This is the scariest because monitoring misses it.**

**Common Causes**:

**A) External Dependency Slow**:
```
Your service: healthy
External API: slow (3s response time)
Result: Your API times out waiting

Solution:
- Monitor external dependency latency
- Implement timeouts and circuit breakers
- Add fallback mechanisms
```

**B) Database Connection Pool Exhausted**:
```
Tasks: healthy (they're running)
Database: has connections available
But: Connection pool maxed out

Solution:
- Monitor connection pool metrics
- Increase pool size or add more tasks
- Implement connection retry logic
```

**C) Cascading Failures**:
```
One service slows down → Others timeout waiting → Entire system degraded

Solution:
- Implement circuit breakers
- Use timeout on all external calls
- Add bulkheads (isolate failures)
```

**Prevention**:
```python
# Monitor the gaps
def check_hidden_issues():
    """Catch issues that standard metrics miss"""
    
    # 1. Compare external vs internal metrics
    synthetic_success_rate = get_metric('SyntheticMonitoring', 'SuccessRate')
    alb_success_rate = 100 - get_metric('AWS/ApplicationELB', '5XXError')
    
    if synthetic_success_rate < alb_success_rate - 5:
        print("ALERT: External monitoring shows worse than internal")
        print("Possible: DNS issues, CloudFront problems, geo-specific failure")
    
    # 2. Check API latency distribution (not just average)
    response_times = get_detailed_latency_metrics()
    p50 = response_times['p50']
    p99 = response_times['p99']
    
    if p99 > (p50 * 10):
        print("ALERT: High latency variance")
        print(f"p50: {p50}ms, p99: {p99}ms")
        print("Some requests 10x slower than average")
```

---

## Day 2 Operations

Building the cluster is one thing. Keeping it healthy long-term is another.

### Daily Checks

**Morning Dashboard Review** (5 minutes):
```
□ All services at desired count?
□ CPU/Memory utilization normal (30-70%)?
□ Error rates <1%?
□ Any deployment failures overnight?
□ Cost anomalies?
```

**Automated Daily Report**:
```python
def daily_health_report():
    report = {
        'date': datetime.now().strftime('%Y-%m-%d'),
        'services': []
    }
    
    services = ecs.list_services(cluster='production')['serviceArns']
    
    for service_arn in services:
        service = ecs.describe_services(
            cluster='production',
            services=[service_arn]
        )['services'][0]
        
        service_health = {
            'name': service['serviceName'],
            'desired': service['desiredCount'],
            'running': service['runningCount'],
            'pending': service['desiredCount'] - service['runningCount'],
            'deployments': len(service['deployments']),
            'events': service['events'][:3]
        }
        
        report['services'].append(service_health)
    
    # Send to Slack/email
    send_daily_report(report)
```

### Weekly Tasks

**Capacity Planning Review**:
```python
def weekly_capacity_review():
    """Check if you need to adjust capacity"""
    
    # Get last 7 days of utilization
    end = datetime.now()
    start = end - timedelta(days=7)
    
    cloudwatch = boto3.client('cloudwatch')
    
    cpu_stats = cloudwatch.get_metric_statistics(
        Namespace='AWS/ECS',
        MetricName='CPUUtilization',
        Dimensions=[
            {'Name': 'ServiceName', 'Value': 'api-service'},
            {'Name': 'ClusterName', 'Value': 'production'}
        ],
        StartTime=start,
        EndTime=end,
        Period=3600,
        Statistics=['Average', 'Maximum']
    )
    
    avg_cpu = sum(p['Average'] for p in cpu_stats['Datapoints']) / len(cpu_stats['Datapoints'])
    max_cpu = max(p['Maximum'] for p in cpu_stats['Datapoints'])
    
    print(f"Weekly CPU - Avg: {avg_cpu:.1f}%, Peak: {max_cpu:.1f}%")
    
    if avg_cpu < 30:
        print("RECOMMENDATION: Consider scaling down (low utilization)")
    elif max_cpu > 85:
        print("RECOMMENDATION: Add capacity (hitting limits)")
    else:
        print("✓ Capacity looks good")
```

**Cost Review**:
```python
def weekly_cost_review():
    """Check for cost anomalies"""
    
    ce = boto3.client('ce')  # Cost Explorer
    
    end = datetime.now().date()
    start = end - timedelta(days=7)
    
    cost = ce.get_cost_and_usage(
        TimePeriod={
            'Start': str(start),
            'End': str(end)
        },
        Granularity='DAILY',
        Metrics=['UnblendedCost'],
        Filter={
            'Dimensions': {
                'Key': 'SERVICE',
                'Values': ['Amazon Elastic Container Service']
            }
        }
    )
    
    daily_costs = [
        float(day['Total']['UnblendedCost']['Amount'])
        for day in cost['ResultsByTime']
    ]
    
    avg_daily = sum(daily_costs) / len(daily_costs)
    max_daily = max(daily_costs)
    
    print(f"ECS Cost - Daily avg: ${avg_daily:.2f}, Peak: ${max_daily:.2f}")
    
    if max_daily > (avg_daily * 1.5):
        print("WARNING: Cost spike detected")
```

### Monthly Tasks

- Review and adjust Reserved Instances
- Audit security group rules
- Update task definitions with latest base images
- Review and archive old CloudWatch Logs
- Test disaster recovery procedures
- Update runbooks based on recent incidents

---

## The War Room Dashboard

When things go wrong, you need answers fast. Build a dashboard that shows everything critical on one screen.

**CloudWatch Dashboard JSON**:
```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "title": "Service Health Overview",
        "metrics": [
          ["AWS/ECS", "RunningTaskCount", {"stat": "Average", "label": "Running"}],
          [".", "DesiredTaskCount", {"stat": "Average", "label": "Desired"}],
          ["AWS/ApplicationELB", "HealthyHostCount", {"stat": "Average"}],
          [".", "UnHealthyHostCount", {"stat": "Average"}]
        ],
        "period": 60,
        "region": "us-east-1",
        "yAxis": {"left": {"min": 0}}
      }
    },
    {
      "type": "metric",
      "properties": {
        "title": "API Performance",
        "metrics": [
          ["AWS/ApplicationELB", "TargetResponseTime", {"stat": "Average"}],
          ["...", {"stat": "p99"}],
          [".", "RequestCount", {"stat": "Sum", "yAxis": "right"}]
        ],
        "period": 60
      }
    },
    {
      "type": "metric",
      "properties": {
        "title": "Error Rates",
        "metrics": [
          ["AWS/ApplicationELB", "HTTPCode_Target_5XX_Count", {"stat": "Sum"}],
          [".", "HTTPCode_Target_4XX_Count", {"stat": "Sum"}],
          ["CustomECS", "ApplicationErrors", {"stat": "Sum"}]
        ],
        "period": 60
      }
    },
    {
      "type": "log",
      "properties": {
        "title": "Recent Errors",
        "query": "SOURCE '/ecs/api-service' | fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 20",
        "region": "us-east-1"
      }
    }
  ]
}
```

**Key Metrics on War Room Dashboard**:
- Running vs Desired task count (detect placement issues)
- Healthy vs Unhealthy targets (detect health check failures)
- Request count and latency (understand load)
- Error rates (detect application issues)
- CPU/Memory utilization (detect resource pressure)
- Recent error logs (investigate issues)

---

## Troubleshooting Playbook

### Issue: Tasks Won't Start

**Step 1: Check Service Events**
```bash
aws ecs describe-services \
  --cluster production \
  --services api-service \
  --query 'services[0].events[0:5]'
```

Look for:
- "unable to place task" = capacity issue
- "CannotPullContainer" = image pull issue
- "ResourceInitializationError" = networking/IAM issue

**Step 2: Check Cluster Capacity** (EC2)
```bash
aws ecs describe-clusters \
  --cluster production \
  --include STATISTICS
```

If CPUReservation or MemoryReservation >95% → Add capacity

**Step 3: Check Task Definition**
```bash
aws ecs describe-task-definition \
  --task-definition api-service:42
```

Verify:
- Image exists and is accessible
- CPU/memory requirements reasonable
- IAM roles have correct permissions

**Step 4: Check Logs**
```bash
aws logs tail /ecs/api-service --follow
```

### Issue: High CPU/Memory

**Step 1: Identify Which Tasks**
```bash
aws ecs list-tasks \
  --cluster production \
  --service-name api-service \
  --desired-status RUNNING

aws ecs describe-tasks \
  --cluster production \
  --tasks <task-arn>
```

**Step 2: Check Container-Level Metrics**
```python
# Get metrics for specific container
cloudwatch.get_metric_statistics(
    Namespace='AWS/ECS',
    MetricName='CPUUtilization',
    Dimensions=[
        {'Name': 'ClusterName', 'Value': 'production'},
        {'Name': 'ServiceName', 'Value': 'api-service'},
        {'Name': 'TaskId', 'Value': 'abc123'}  # Specific task
    ],
    StartTime=datetime.now() - timedelta(hours=1),
    EndTime=datetime.now(),
    Period=300,
    Statistics=['Average', 'Maximum']
)
```

**Step 3: Connect to Task** (for deep debugging)
```bash
# Enable ECS Exec first (in task definition)
aws ecs execute-command \
  --cluster production \
  --task <task-id> \
  --container app \
  --interactive \
  --command "/bin/bash"

# Then inside container:
top -b -n 1  # Check processes
df -h        # Check disk usage
free -m      # Check memory
```

### Issue: Deployment Stuck

**Step 1: Check Deployment Status**
```bash
aws ecs describe-services \
  --cluster production \
  --services api-service \
  --query 'services[0].deployments'
```

**Step 2: Check New Task Health**
```bash
# List tasks from new deployment
aws ecs list-tasks \
  --cluster production \
  --service-name api-service \
  --desired-status RUNNING

# Check which ones are from new revision
aws ecs describe-tasks --cluster production --tasks <task-arns> \
  --query 'tasks[*].[taskArn, taskDefinitionArn, lastStatus, healthStatus]'
```

**Step 3: Manual Rollback** (if needed)
```bash
# Update service to previous task definition
aws ecs update-service \
  --cluster production \
  --service api-service \
  --task-definition api-service:41  # Previous working version
```

---

## Summary: The Checklist

Before going to production, verify:

### Architecture
- ✅ Using 3+ Availability Zones
- ✅ Proper subnet sizing (enough IPs)
- ✅ awsvpc networking mode configured
- ✅ Task placement strategy spreads across AZs
- ✅ Load balancer with health checks configured

### Capacity
- ✅ 30% capacity buffer (EC2) or sufficient limits (Fargate)
- ✅ Auto-scaling configured (both service and cluster for EC2)
- ✅ Capacity provider auto-scaling enabled (EC2)

### Task Configuration
- ✅ Resource requests realistic (tested under load)
- ✅ Health check endpoint implemented and tested
- ✅ Health check grace period covers startup time
- ✅ Logging configured with retention policy
- ✅ Environment variables secured (Secrets Manager/SSM)

### Deployment
- ✅ Circuit breaker enabled with rollback
- ✅ Deployment configuration allows zero-downtime
- ✅ Pre-deployment validation script
- ✅ Rollback procedure documented

### Monitoring
- ✅ CloudWatch alarms for critical metrics
- ✅ Synthetic monitoring from external source
- ✅ Custom metrics for task placement failures
- ✅ Log-based metrics for application errors
- ✅ Dashboard for war room situations

### Operations
- ✅ Daily health check process
- ✅ Weekly capacity review
- ✅ Runbooks for common issues
- ✅ On-call rotation and escalation path
- ✅ Incident post-mortem process

---

## Final Thoughts

Building a reliable ECS cluster isn't just about the initial setup—it's about building observability, automation, and processes that catch problems early.

The best clusters are boring. They just work. You don't think about them. That's the goal.

Start with the basics: proper multi-AZ distribution, solid health checks, and basic monitoring. Then layer on the advanced stuff: custom metrics, synthetic monitoring, automated remediation.

And remember: every outage is a learning opportunity. When something breaks (and it will), update your runbooks, add monitoring for that specific failure mode, and make the system more resilient.

**What would you like to explore next?**
- Deep dive into specific failure scenarios with step-by-step debugging?
- Building automated remediation (self-healing clusters)?
- Cost optimization strategies while maintaining reliability?
- Migrating from EC2 to Fargate (or vice versa) without downtime?

Let me know and we'll dig deeper!
