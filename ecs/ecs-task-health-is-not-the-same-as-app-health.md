# ECS Task Health Is Not Application Health

> A running task is an infrastructure fact. A healthy application is an operational outcome. Confusing them is one of the most expensive mistakes you can make in a production ECS environment.


---

## The lie your ECS console tells you

Open the ECS console during a deployment. You see this:

```
Service:  api-service
Status:   ACTIVE
Running:  3 / 3 desired
```

Everything looks green. The deployment event stream says tasks launched. The service stabilized. Your CI/CD pipeline marked the deployment as successful.

And yet — your application is returning 500s. Or it's silently timing out on database connections. Or it started fine but is stuck in a half-initialized state that no health check is catching.

The console told you the truth about orchestration. It told you nothing about your application.

That gap is where incidents are born.

---

## What "task is RUNNING" actually means

When ECS marks a task as `RUNNING`, it means one specific thing: **the ECS scheduler placed the task on compute, the container runtime started the process, and the container did not immediately exit.**

That's it. That's the full claim.

It does not mean:

- The application finished initializing
- External dependencies (databases, caches, downstream APIs) are reachable
- Secrets were successfully fetched and injected
- The application can actually process requests
- The JVM finished its warmup cycle
- The connection pool established its minimum connections
- The application's internal health state is green

To put it bluntly: `RUNNING` means the process started. It says nothing about what the process is doing.

This is not a flaw in ECS. It's a correct and honest signal at the orchestration layer. The mistake is reading an orchestration signal and drawing an application conclusion from it.

---

## The four layers of health — and where teams stop reading

I've seen teams at every stage of ECS maturity, and the pattern is consistent: they stop reading at the wrong layer.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 4 — Application behavior                                  │
│  "Can the app handle real requests correctly and consistently?"  │
│  Signals: error rates, latency p99, business logic correctness   │
├─────────────────────────────────────────────────────────────────┤
│  Layer 3 — Load balancer / target group health                   │
│  "Is the ALB registering this task as healthy?"                  │
│  Signals: target group health status, ECS health check state     │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2 — Container / process health                            │
│  "Did the main process start and stay running?"                  │
│  Signals: container exit codes, OOM events, restart counts       │
├─────────────────────────────────────────────────────────────────┤
│  Layer 1 — Task state                                            │
│  "Did ECS launch the task and keep it alive?"                    │
│  Signals: task status (RUNNING/STOPPED), ECS service events      │
└─────────────────────────────────────────────────────────────────┘
```

Layer 1 is the cheapest signal to read — it's right there in the console. Layer 4 is the most expensive signal to instrument but the only one that tells you what your users are actually experiencing.

Most teams stop at Layer 2. They see tasks running and containers alive, and they call it done.

A serious production environment reads all four layers, all the time.

---

## Why weak health checks are more dangerous than no health checks

This is counterintuitive. Surely a shallow health check is better than nothing? In one sense, yes. But there's a specific failure mode that shallow health checks create: **false confidence**.

Here's the classic example. Your Node.js API has this health check in its Task Definition:

```json
"healthCheck": {
  "command": ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"],
  "interval": 30,
  "timeout": 5,
  "retries": 3,
  "startPeriod": 10
}
```

And your `/health` endpoint looks like this:

```javascript
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok' });
});
```

That check will pass the moment Express is listening. It says nothing about whether:

- The database connection pool is established
- Redis is reachable
- A required feature flag service responded at startup
- Background workers are processing
- The app's internal state machine finished initializing

Your health check returns 200. ECS marks the task healthy. The load balancer registers it. Traffic flows. And your users start hitting `503 Database connection timeout` because the connection pool never came up.

The health check didn't fail. The application did. And your infrastructure never knew the difference.

### What a meaningful health check looks like

A production-grade `/health` endpoint actually interrogates the application's dependencies:

```javascript
// health.js — a health check that means something

const db = require('./db');
const redis = require('./redis');
const { isReady } = require('./startup-state');

app.get('/health', async (req, res) => {
  if (!isReady()) {
    return res.status(503).json({
      status: 'starting',
      message: 'Application still initializing'
    });
  }

  const checks = await Promise.allSettled([
    db.query('SELECT 1'),
    redis.ping(),
  ]);

  const dbHealthy  = checks[0].status === 'fulfilled';
  const redisHealthy = checks[1].status === 'fulfilled';

  const allHealthy = dbHealthy && redisHealthy;

  return res.status(allHealthy ? 200 : 503).json({
    status: allHealthy ? 'healthy' : 'degraded',
    checks: {
      database: dbHealthy ? 'ok' : 'failing',
      redis:    redisHealthy ? 'ok' : 'failing',
    }
  });
});
```

This check can actually fail a deployment. The shallow version cannot.

> **The test for a meaningful health check:** if your application is functionally broken — returning errors for all real requests — would your health check fail? If the answer is no, your health check is not protecting you.

---

## The `startPeriod` trap

The `startPeriod` field in ECS health checks is how long ECS will ignore health check failures during container startup. It exists because applications take time to initialize, and you don't want three failing health checks during the first 30 seconds of JVM warmup to kill a perfectly valid task.

Here's the trap: teams set `startPeriod` too short, health checks fail during initialization, ECS kills the task and starts a new one — which also fails — which creates a restart loop that looks exactly like a deployment failure even if the application is fine.

The fix seems obvious: increase `startPeriod`. But here's the subtler trap on the other side: teams set `startPeriod` so long (say, 300 seconds) that a genuinely broken deployment spends five minutes churning through restart cycles before the circuit breaker triggers. Meanwhile, if this is a rolling deployment, your old healthy tasks are potentially being drained while the new ones are in a death loop.

**The discipline:** measure your actual p99 startup time in staging, add a 50% buffer, and set that as your `startPeriod`. For most services this is 30–90 seconds. For JVM services with heavy initialization, it might be 120 seconds.

```json
"healthCheck": {
  "command": ["CMD-SHELL", "curl -sf http://localhost:8080/actuator/health || exit 1"],
  "interval": 15,
  "timeout": 5,
  "retries": 3,
  "startPeriod": 90
}
```

A good rule of thumb for the other values: `interval` should be short enough to detect failures quickly (15–30 seconds), `retries` * `interval` gives you the failure detection window. Three retries at 15 seconds = 45 seconds to detect and act on a failing health check.

---

## The two-health-check system you probably aren't using

ECS and ALB give you two separate, independent health check mechanisms, and most teams only configure one.

### Health check 1: ECS container health check

Defined in your Task Definition. Executed by the container agent inside the task itself. This check is what ECS uses to determine whether to restart a container or mark a task as unhealthy.

```json
"healthCheck": {
  "command": ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"],
  "interval": 30,
  "timeout": 5,
  "retries": 3,
  "startPeriod": 60
}
```

### Health check 2: ALB target group health check

Configured on your load balancer target group, completely separate from ECS. This is what the ALB uses to decide whether to route traffic to a given task.

```hcl
resource "aws_lb_target_group" "api" {
  name     = "api-tg"
  port     = 3000
  protocol = "HTTP"
  vpc_id   = var.vpc_id

  health_check {
    enabled             = true
    path                = "/health/ready"   # separate readiness endpoint
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 15
    matcher             = "200"
  }
}
```

The key insight: **these two health checks serve different purposes and should check different things.**

| | ECS Container Health Check | ALB Target Group Health Check |
|---|---|---|
| Purpose | Should ECS restart this container? | Should ALB route traffic here? |
| Checks | Liveness — is the app alive? | Readiness — is the app ready for traffic? |
| Failure action | ECS kills and restarts the container | ALB stops routing to this task |
| Path | `/health` or `/ping` | `/health/ready` or `/health/live` |

**Liveness vs Readiness — the Kubernetes pattern that ECS teams ignore**

In Kubernetes, the liveness/readiness split is a first-class concept. In ECS, you have to implement it yourself, but the principle is just as important.

A **liveness** check answers: "Is the process alive and not deadlocked?" If this fails, restart the container. It should be fast, cheap, and almost always succeed. A hung goroutine or a deadlocked thread pool is what it catches.

A **readiness** check answers: "Can this instance actually serve traffic right now?" If this fails, pull it from the load balancer but don't restart it. A downstream database being temporarily unreachable is what it catches — you don't want to restart the container, you just want to stop routing traffic to it until the DB recovers.

Use your ECS container health check as a liveness check. Use your ALB target group health check as a readiness check. Point them at different endpoints.

```javascript
// Liveness — is the process alive?
app.get('/health/live', (req, res) => {
  res.status(200).json({ status: 'alive' });
});

// Readiness — can we serve traffic?
app.get('/health/ready', async (req, res) => {
  try {
    await db.query('SELECT 1');
    await cache.ping();
    res.status(200).json({ status: 'ready' });
  } catch (err) {
    res.status(503).json({ status: 'not_ready', error: err.message });
  }
});
```

---

## How a deployment can lie to you

Here is the sequence of events for a deployment that looks successful but isn't:

```
T+0:00   new Task Definition rev:8 registered
T+0:10   ECS launches 3 new tasks (rolling deployment)
T+0:45   containers start, process starts
T+0:50   /health endpoint responds 200 (Express is listening)
T+0:55   ECS marks tasks as HEALTHY
T+1:00   ALB registers new tasks as healthy
T+1:05   ECS drains old tasks
T+1:20   old tasks stopped — deployment "complete"

T+1:25   database connection pool throws connection refused
T+1:25   app starts returning 500 on all write endpoints
T+1:25   users start seeing errors
T+1:25   ECS health check still passes (Express is still running)
T+1:25   ALB target group still shows healthy
T+1:25   your CI/CD pipeline shows green
T+6:30   on-call gets paged
```

The application was broken at T+1:25. The infrastructure didn't know until T+6:30 when someone checked. Every signal said healthy. The health check was checking the wrong thing.

This is not a rare edge case. This is a common production incident pattern.

---

## What to instrument instead

If you accept that task state is a weak proxy for application health, then the question becomes: what do you actually instrument to know your deployment is good?

### Signal 1 — Deployment event timeline from ECS

The ECS service event stream tells you exactly what happened and when. Don't just check final state — check the timing.

```bash
aws ecs describe-services \
  --cluster my-cluster \
  --services api-service \
  --query 'services[0].events[0:20]' \
  --output table
```

What you're looking for: how long did tasks spend in PENDING before reaching RUNNING? Did any tasks stop and restart? Was the service state stable or cycling?

A task that went `PENDING → RUNNING → STOPPED → PENDING → RUNNING` is a restart loop, and it looks "healthy" on the final state alone.

### Signal 2 — Target group health, not task health

When a new deployment goes out, watch the ALB target group health, not the ECS task status. The target group shows you what the load balancer actually thinks — which is closer to what your users will experience.

```bash
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --query 'TargetHealthDescriptions[*].{ID:Target.Id,Port:Target.Port,State:TargetHealth.State,Reason:TargetHealth.Reason}' \
  --output table
```

A task in ECS `RUNNING` state with ALB target health `unhealthy` is a split-brain situation that absolutely should block a deployment from marking itself complete.

### Signal 3 — Container Insights metrics during deployment

If you have Container Insights enabled (you should), watch these metrics during and immediately after a deployment:

```
ECS/ContainerInsights:
  - CpuUtilized (spike = initialization load, sustained = app issue)
  - MemoryUtilized (spike post-deploy = memory leak in new version)
  - RunningTaskCount (dip during rolling deploy = min_healthy violation)
  - TaskSetDesiredCount vs TaskSetRunningCount (gap = tasks failing to start)
```

A post-deployment memory spike that stabilizes is normal. A memory spike that keeps climbing is a leak in the new version. Container Insights will show you this within 60 seconds of traffic flowing.

### Signal 4 — CloudWatch Logs Insights for startup errors

The most direct application-layer signal is logs. Run this query immediately after a deployment completes:

```
fields @timestamp, @message
| filter @logStream like /YOUR_SERVICE_PREFIX/
| filter @timestamp > 1698000000000  # deployment time in ms
| filter @message like /ERROR|WARN|FATAL|Exception|failed|connection refused/
| sort @timestamp asc
| limit 100
```

If the first 60 seconds of your new tasks are clean, that's a strong positive signal. If they're full of `Connection refused` or `Unable to acquire connection from pool`, your deployment is not healthy regardless of what ECS says.

### Signal 5 — Application-level error rate at the load balancer

Your ALB publishes `HTTPCode_Target_5XX_Count` and `TargetResponseTime` to CloudWatch. Create a CloudWatch alarm on 5xx rate:

```hcl
resource "aws_cloudwatch_metric_alarm" "deployment_5xx" {
  alarm_name          = "api-5xx-post-deploy"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "HTTPCode_Target_5XX_Count"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  statistic           = "Sum"
  threshold           = 10
  alarm_description   = "5xx errors spiked — possible bad deployment"

  dimensions = {
    LoadBalancer = aws_lb.api.arn_suffix
    TargetGroup  = aws_lb_target_group.api.arn_suffix
  }
}
```

A deployment that spikes 5xx errors should automatically trigger a rollback. With ECS deployment circuit breaker, you get this at the task launch layer. But task launch succeeding + application returning 500s is a gap the circuit breaker doesn't cover — only your ALB metrics can catch it.

---

## Making your CI/CD pipeline honest about deployment health

A pipeline that declares success based on task launch is making a claim it cannot support. Here's what an honest deployment verification step looks like:

```bash
#!/usr/bin/env bash
# deploy-verify.sh — post-deployment health verification

CLUSTER=$1
SERVICE=$2
TIMEOUT=300  # 5 minutes
INTERVAL=15

echo "Verifying deployment health for $SERVICE..."

# Step 1: Wait for service stability (all tasks running and healthy)
aws ecs wait services-stable \
  --cluster "$CLUSTER" \
  --services "$SERVICE" \
  --cli-read-timeout "$TIMEOUT"

if [ $? -ne 0 ]; then
  echo "FAIL: Service did not stabilize within ${TIMEOUT}s"
  exit 1
fi

echo "Tasks launched and stable. Verifying application health..."

# Step 2: Check ALB target group — not ECS task health
TG_ARN=$(aws ecs describe-services \
  --cluster "$CLUSTER" \
  --services "$SERVICE" \
  --query 'services[0].loadBalancers[0].targetGroupArn' \
  --output text)

UNHEALTHY=$(aws elbv2 describe-target-health \
  --target-group-arn "$TG_ARN" \
  --query 'length(TargetHealthDescriptions[?TargetHealth.State!=`healthy`])' \
  --output text)

if [ "$UNHEALTHY" -gt 0 ]; then
  echo "FAIL: $UNHEALTHY tasks are unhealthy in the ALB target group"
  aws elbv2 describe-target-health \
    --target-group-arn "$TG_ARN" \
    --query 'TargetHealthDescriptions[*].{IP:Target.Id,State:TargetHealth.State,Reason:TargetHealth.Reason}' \
    --output table
  exit 1
fi

# Step 3: Check for 5xx errors in the last 2 minutes
LB_ARN_SUFFIX=$(aws elbv2 describe-load-balancers \
  --query "LoadBalancers[?contains(DNSName, 'api')].LoadBalancerArn | [0]" \
  --output text | sed 's|.*loadbalancer/||')

TG_ARN_SUFFIX=$(echo "$TG_ARN" | sed 's|.*:targetgroup/||;s|arn:aws:elasticloadbalancing:[^:]*:[^:]*:||')

END_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ)
START_TIME=$(date -u -d '2 minutes ago' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null \
  || date -u -v-2M +%Y-%m-%dT%H:%M:%SZ)  # macOS fallback

ERROR_COUNT=$(aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_Target_5XX_Count \
  --dimensions "Name=LoadBalancer,Value=$LB_ARN_SUFFIX" \
                "Name=TargetGroup,Value=targetgroup/$TG_ARN_SUFFIX" \
  --start-time "$START_TIME" \
  --end-time "$END_TIME" \
  --period 120 \
  --statistics Sum \
  --query 'Datapoints[0].Sum' \
  --output text)

if [ "$ERROR_COUNT" != "None" ] && [ "$(echo "$ERROR_COUNT > 5" | bc)" -eq 1 ]; then
  echo "WARN: ${ERROR_COUNT} 5xx errors observed in the last 2 minutes"
  echo "Consider investigating before declaring deployment healthy"
fi

# Step 4: Check for recent task restarts
RESTARTS=$(aws ecs describe-services \
  --cluster "$CLUSTER" \
  --services "$SERVICE" \
  --query 'services[0].events[?contains(message, `stopped`) || contains(message, `restarted`)] | length(@)' \
  --output text)

if [ "$RESTARTS" -gt 0 ]; then
  echo "WARN: $RESTARTS restart events detected in service event stream"
fi

echo ""
echo "Deployment verification complete."
echo "  Tasks: RUNNING and STABLE"
echo "  ALB targets: ALL HEALTHY"
echo "  5xx errors: ${ERROR_COUNT:-0}"
echo "  Restart events: ${RESTARTS:-0}"
```

This is the minimum bar. A deployment that passes all four checks has a much higher chance of being genuinely healthy than one that only checks task state.

---

## The readiness vs liveness split in ECS — implementing it properly

Here's the full implementation pattern I use in production, combining ECS container health check for liveness and ALB health check for readiness.

### Task Definition health check (liveness)

```json
{
  "containerDefinitions": [
    {
      "name": "api",
      "image": "your-image:tag",
      "healthCheck": {
        "command": [
          "CMD-SHELL",
          "curl -sf http://localhost:3000/health/live || exit 1"
        ],
        "interval": 15,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

### ALB Target Group (readiness)

```hcl
resource "aws_lb_target_group" "api" {
  health_check {
    enabled             = true
    path                = "/health/ready"
    interval            = 15
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 3
    matcher             = "200"
  }
}
```

### Application endpoints

```javascript
// Liveness — ECS uses this to decide whether to restart
// Should almost never return non-200
// Only fails if the process is genuinely broken (deadlock, OOM about to happen, etc.)
app.get('/health/live', (req, res) => {
  res.status(200).json({
    status: 'alive',
    uptime: process.uptime(),
    pid: process.pid
  });
});

// Readiness — ALB uses this to decide whether to route traffic
// Can return 503 if dependencies are down
// ECS will NOT restart the container just because this fails
app.get('/health/ready', async (req, res) => {
  const checks = {};
  let isReady = true;

  // Check 1: Startup complete
  if (!appState.isInitialized()) {
    return res.status(503).json({ status: 'starting' });
  }

  // Check 2: Database
  try {
    await db.raw('SELECT 1');
    checks.database = 'ok';
  } catch (e) {
    checks.database = `failing: ${e.message}`;
    isReady = false;
  }

  // Check 3: Cache (warning only, not blocking)
  try {
    await redis.ping();
    checks.cache = 'ok';
  } catch (e) {
    checks.cache = `degraded: ${e.message}`;
    // Cache failure degrades but doesn't block — depends on your app
  }

  const statusCode = isReady ? 200 : 503;
  return res.status(statusCode).json({
    status: isReady ? 'ready' : 'not_ready',
    checks,
    version: process.env.APP_VERSION || 'unknown'
  });
});
```

---

## Common failure patterns and what they actually mean

### Pattern 1 — Tasks cycling (RUNNING → STOPPED → RUNNING)

What the console shows: desired count met, service events show repeated launches.

What it actually means: the container is starting successfully but dying shortly after. Common causes: out-of-memory kills (check CloudWatch for `OOMKilled`), application crash on startup, health check failing with no `startPeriod` grace, or an essential sidecar container dying and taking the main container with it.

How to diagnose:
```bash
# Check stopped task exit codes
aws ecs describe-tasks \
  --cluster my-cluster \
  --tasks $(aws ecs list-tasks --cluster my-cluster --service-name api --desired-status STOPPED --query 'taskArns[0]' --output text) \
  --query 'tasks[0].containers[*].{name:name,exitCode:exitCode,reason:reason}'
```

Exit code 137 = OOM kill. Exit code 1 = application crash. Exit code 0 from a non-essential container that shouldn't have exited = you have an `essential: true` bug.

### Pattern 2 — Deployment hangs at minimum healthy percent

What the console shows: deployment in progress indefinitely, old tasks not draining.

What it actually means: new tasks are failing health checks. ECS refuses to drain old tasks because doing so would drop below `minimum_healthy_percent`. It keeps trying to launch new tasks, they keep failing, the old ones stay up.

This is actually your safety net working correctly — but it means the deployment is stuck. The fix is to look at why new tasks are failing health checks, not to lower `minimum_healthy_percent`.

### Pattern 3 — Deployment succeeds, errors spike after 2 minutes

What the console shows: clean deployment, all tasks healthy, no anomalies.

What it actually means: the application dependency (usually database, cache, or external API) was reachable during health checks but degraded once traffic arrived at scale. The health check never exercises the actual load path.

This is the hardest failure mode to catch with infrastructure signals alone. You need application-level error rate monitoring (ALB 5xx metrics, custom CloudWatch metrics, or an APM tool) to catch it.

### Pattern 4 — Healthy tasks, unhealthy users

What the console shows: all tasks running, health checks passing, no events.

What it actually means: the health check path is not representative of actual application behavior. Classic examples: `/health` always returns 200 regardless of app state, or the health check tests a read path but writes are broken, or the health check bypasses authentication middleware that's actually broken.

The diagnostic question: if I curl the health check endpoint from outside the VPC (if publicly accessible), does it return 200? And if I then make a real business request, does that also succeed? If the answer to the second question is no, your health check is lying.

---

## The deployment verification checklist

Before you declare a deployment successful, verify all of these:

**Infrastructure signals (30 seconds)**
- [ ] `aws ecs wait services-stable` returned 0 (all tasks running, desired count met)
- [ ] ECS service event stream shows no unexpected STOPPED events
- [ ] No tasks cycling (same task ARN appearing in STOPPED + new task launched)

**Application signals (2–5 minutes)**
- [ ] ALB target group: all registered targets in `healthy` state
- [ ] ALB metrics: `HTTPCode_Target_5XX_Count` baseline unchanged post-deploy
- [ ] ALB metrics: `TargetResponseTime` p99 unchanged or improved post-deploy
- [ ] CloudWatch Logs: no ERROR/FATAL messages in first 2 minutes of new task logs

**Deep verification (5–10 minutes, for high-risk deploys)**
- [ ] Smoke test: synthetic transaction through the application's critical path
- [ ] Dependency check: downstream service call success rates unchanged
- [ ] Memory trend: `MemoryUtilized` not climbing post-deploy (ruling out memory leak)
- [ ] Business metric baseline: if you have it, confirm key business metrics are unchanged

If you automate nothing else from this article, automate the ALB target group health check as a post-deployment gate. It is the single highest-signal check you can add with the lowest implementation cost.

---

## The deployment circuit breaker gap

The ECS deployment circuit breaker is valuable. Enable it. Set `rollback = true`. But understand precisely what it protects against and what it doesn't.

**What the circuit breaker catches:**
Tasks that fail to launch, fail to start, fail container health checks before reaching `RUNNING`, or fail to register with the ALB target group within the deployment window.

**What the circuit breaker does NOT catch:**
Tasks that launch successfully, pass health checks, register with the ALB, begin serving traffic — and then start failing at the application layer. Once a task is considered "successfully deployed," the circuit breaker is done evaluating it.

That second failure mode is more common than the first in well-instrumented environments, because the first failure mode is well-understood and teams have gotten good at preventing it. The gap is the application-layer failure that no infrastructure signal can see.

Filling that gap requires you to build your own post-deployment verification — either in your CI/CD pipeline, in a deployment hook Lambda, or via CloudWatch alarms that trigger SNS → CodeDeploy rollback.

---

## Deep insight: the `startPeriod` and `deregistration_delay` relationship

These two settings interact in a way that most documentation doesn't explain clearly.

During a rolling deployment:

1. New task starts, `startPeriod` grace period begins
2. ECS ignores health check failures during `startPeriod`
3. After `startPeriod`, the health check must pass within `retries * interval` seconds
4. Once the task passes health checks, it registers with the ALB
5. Once registered, the ALB's own health check must pass (`healthy_threshold` consecutive checks)
6. Once ALB-healthy, old tasks begin draining
7. Old tasks stay up for `deregistration_delay` while in-flight requests complete
8. After `deregistration_delay`, old tasks are stopped

The minimum safe deployment window for any task is:
```
startPeriod + (retries × interval) + (healthy_threshold × alb_interval) + deregistration_delay
```

For typical values: `60 + (3 × 15) + (2 × 15) + 30 = 60 + 45 + 30 + 30 = 165 seconds`

If your CI/CD pipeline marks a deployment "complete" in under 3 minutes, it is almost certainly not waiting for this full window. It's marking the deployment complete based on task launch, not deployment completion.

---

## Practical recommendations

**For any ECS service carrying production traffic:**

1. Implement separate liveness and readiness endpoints. Point your container health check at liveness and your ALB health check at readiness.

2. Your readiness endpoint must actively test dependencies. An endpoint that returns 200 unconditionally is not a health check — it's a heartbeat. Heartbeats are useful but are not health checks.

3. Measure and set `startPeriod` based on actual p99 startup time. Check this number every time you make significant changes to application initialization (new database migrations, new secret fetching, heavier startup logic).

4. Add `aws ecs wait services-stable` as a blocking step in your deployment pipeline, followed by ALB target group verification. These two checks together take 2–3 minutes and catch the majority of deployment failures.

5. Create a CloudWatch alarm on ALB 5xx rate. Configure it to notify within 2 evaluation periods (2 minutes). A deployment that causes a 5xx spike should page someone — automatically.

6. Enable ECS deployment circuit breaker with `rollback = true`. This is a free safety net. There is no reason not to have it.

7. Log the `APP_VERSION` (or git SHA) in every log line and in your health check response. When an incident happens, you want to know immediately which version is running without grepping through ECS console.

---

## The bottom line

ECS task health is an infrastructure signal. It tells you the container runtime did its job. That's valuable and necessary — but it's not sufficient.

Application health is what your users experience. It lives at Layer 4: actual requests being handled correctly, at acceptable latency, without errors.

The gap between them is where most production incidents hide. A task can be `RUNNING`, pass every container health check, register as healthy with the ALB, and still be returning 500s to users because the health check doesn't test what users actually do.

Close that gap with meaningful health checks, layered observability, and a deployment verification pipeline that doesn't stop at task launch. The infrastructure can't tell you the application is healthy. Only the application can tell you that — if you ask it the right questions.

---

## Reference: health check configuration cheat sheet

```json
{
  "healthCheck": {
    "command": ["CMD-SHELL", "curl -sf http://localhost:PORT/health/live || exit 1"],
    "interval": 15,
    "timeout": 5,
    "retries": 3,
    "startPeriod": 60
  }
}
```

| Parameter | What it does | Recommendation |
|---|---|---|
| `interval` | Seconds between checks | 15–30s for production |
| `timeout` | Seconds before check fails | 5s; if your health check takes >5s, the endpoint is too heavy |
| `retries` | Consecutive failures before UNHEALTHY | 3; gives 45s window at 15s interval |
| `startPeriod` | Grace period at startup (failures ignored) | Measure p99 startup + 50% buffer |

```hcl
# ALB target group health check (readiness)
health_check {
  path                = "/health/ready"
  interval            = 15
  timeout             = 5
  healthy_threshold   = 2    # 2 consecutive passes to become healthy
  unhealthy_threshold = 3    # 3 consecutive failures to become unhealthy
  matcher             = "200"
}
```

| Parameter | What it does | Recommendation |
|---|---|---|
| `healthy_threshold` | Consecutive passes to mark healthy | 2 (fast registration on deploy) |
| `unhealthy_threshold` | Consecutive fails to mark unhealthy | 3 (avoid flapping on transient failures) |
| `interval` | Seconds between checks | Match ECS health check interval |

---

*All AWS CLI commands tested against AWS CLI v2. ECS deployment circuit breaker requires Docker platform version 1.4.0 or later on Fargate, or ECS agent 1.45.0 or later on EC2. Container Insights must be enabled on the cluster to access per-task CPU/memory metrics.*
