# The Deploy Isn't Done When the Pipeline Goes Green 🟢

> Most outages don't happen during the deploy.  
> They happen **10 minutes after** — when everyone closed their laptops.  
> This is what you run before you walk away.  
> Whether you're on Kubernetes, ECS, or an EC2 server running Tomcat.

---

## The Mindset

Passing CI/CD means your code **built and tested cleanly**.  
It says nothing about whether real users are actually okay.

```
Pipeline Green  ≠  Users Happy
                ≠  Performance nominal
                ≠  Database healthy
                ≠  JVM warmed up
                ≠  Nothing silently broken
```

Post-deployment validation is the 10-minute ritual that catches the 3AM page before it happens. It works the same whether your app runs in a pod, an ECS task, or a Tomcat instance a previous engineer named `prod-server-2-FINAL-v3`.

Pick your platform. Run the checks. Walk away confident.

---

## Jump to Your Platform

| I deploy to... | Go to |
|---|---|
| Kubernetes / EKS | [Section A](#section-a--kubernetes--eks) |
| AWS ECS (Fargate or EC2) | [Section B](#section-b--aws-ecs) |
| EC2 + Tomcat / JBoss / Spring Boot JAR | [Section C](#section-c--ec2--tomcat--jboss--spring-boot) |
| All platforms | [Shared Checks](#shared-checks-all-platforms) |

---

## Section A — Kubernetes / EKS

### ⚡ A1 — First 60 Seconds (The Gut Check)

```bash
# Are all pods actually running?
kubectl get pods -n production | grep -v Running | grep -v Completed
# Any pod NOT Running/Completed = stop here

# Did the rollout complete cleanly?
kubectl rollout status deployment/myapp -n production
# "successfully rolled out" is the only acceptable output

# Is the image correct? No old pods still sneaking traffic?
kubectl get pods -n production -l app=myapp \
  -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}' | sort -u
# Should show ONLY the new SHA
```

```
✅ All pods Running
✅ Rollout status = successfully rolled out
✅ Only new image SHA visible
✅ No CrashLoopBackOff anywhere in the namespace
```

### 🩺 A2 — Health Check

```bash
# Direct check — bypasses the load balancer entirely
kubectl exec -it \
  $(kubectl get pod -l app=myapp -n production -o jsonpath='{.items[0].metadata.name}') \
  -n production -- curl -s http://localhost:8080/health | jq .

# Then check THROUGH the load balancer — different code path!
curl -sf https://myapp.com/health | jq .
```

**What a bad response means:**

| Response | Root Cause |
|---|---|
| `"db": "timeout"` | Connection pool exhausted or migration still running |
| HTTP 503 from LB | Pod is up but readiness probe failing — check `kubectl describe pod` |
| `"version": "old-sha"` | Old pods still serving — rollout not complete |
| Connection refused | Service selector mismatch |

### 📊 A3 — Metrics (The Real Test)

```bash
# Error rate right now
kubectl exec -n monitoring prometheus-0 -- \
  curl -sg 'http://localhost:9090/api/v1/query' \
  --data-urlencode \
  'query=sum(rate(http_requests_total{job="myapp",status=~"5.."}[5m]))/sum(rate(http_requests_total{job="myapp"}[5m]))*100' \
  | jq '.data.result[0].value[1]'
# < 0.5% = fine   0.5-2% = watch it   > 2% = rollback

# P99 latency creeping up?
kubectl exec -n monitoring prometheus-0 -- \
  curl -sg 'http://localhost:9090/api/v1/query' \
  --data-urlencode \
  'query=histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{job="myapp"}[5m]))' \
  | jq '.data.result[0].value[1]'
# > 20% higher than baseline = investigate before leaving
```

### 🔍 A4 — Logs

```bash
# New errors since deploy only
kubectl logs -n production \
  --selector app=myapp \
  --since=5m \
  | grep -iE "error|exception|traceback|panic|fatal" \
  | head -30

# Before vs after comparison
BEFORE=$(kubectl logs -n production \
  $(kubectl get pod -l app=myapp -n production \
    -o jsonpath='{.items[0].metadata.name}') \
  --previous --since=5m 2>/dev/null | grep -ic "error" || echo "0")
AFTER=$(kubectl logs -n production \
  --selector app=myapp --since=5m | grep -ic "error")
echo "Errors — before: $BEFORE | after: $AFTER"
```

### 💸 A5 — Resources

```bash
# Memory leak check — is it climbing or stable?
kubectl top pods -n production -l app=myapp
# Run this twice, 2 minutes apart. Growing number = memory leak.

# HPA reacting unexpectedly?
kubectl get hpa -n production
# Should look like baseline — not suddenly scaling to max
```

### ↩️ A6 — Rollback Command (Know It Before You Need It)

```bash
kubectl rollout undo deployment/myapp -n production
kubectl rollout status deployment/myapp -n production
```

---

## Section B — AWS ECS

ECS is different from Kubernetes in one important way: **tasks are ephemeral and don't have rollout status commands**. Your signals come from the service, target groups, and CloudWatch.

### ⚡ B1 — First 60 Seconds

```bash
CLUSTER=my-ecs-cluster
SERVICE=myapp-service

# Is the service stable? Are all tasks running?
aws ecs describe-services \
  --cluster $CLUSTER \
  --services $SERVICE \
  --query 'services[0].{
    Status:status,
    Running:runningCount,
    Desired:desiredCount,
    Pending:pendingCount,
    Deployments:deployments[*].{ID:id,Status:status,Running:runningCount,Desired:desiredCount}
  }' \
  --output json

# What you want to see:
# Running == Desired (e.g., 3 == 3)
# Pending == 0
# Only ONE deployment with status PRIMARY
# No deployment with status ACTIVE still lingering (old tasks draining)
```

```bash
# Confirm new task definition is what's running
aws ecs list-tasks \
  --cluster $CLUSTER \
  --service-name $SERVICE \
  --query 'taskArns' \
  --output text | \
xargs aws ecs describe-tasks \
  --cluster $CLUSTER \
  --tasks \
  --query 'tasks[0].{TaskDef:taskDefinitionArn,Status:lastStatus,Health:healthStatus}'

# taskDefinitionArn should end in :NEW_REVISION_NUMBER
# lastStatus should be RUNNING
# healthStatus should be HEALTHY
```

```
✅ runningCount == desiredCount
✅ pendingCount == 0
✅ Only one PRIMARY deployment (no old ACTIVE deployment lingering)
✅ Tasks running new task definition revision
✅ healthStatus: HEALTHY on all tasks
```

### 🩺 B2 — Target Group Health

This is the ECS equivalent of Kubernetes readiness probes. The ALB target group tells you if tasks are actually passing health checks.

```bash
# Find the target group ARN for your service
TG_ARN=$(aws elbv2 describe-target-groups \
  --query "TargetGroups[?contains(TargetGroupName,'myapp')].TargetGroupArn" \
  --output text)

# Check health of every registered target
aws elbv2 describe-target-health \
  --target-group-arn $TG_ARN \
  --query 'TargetHealthDescriptions[*].{
    Target:Target.Id,
    Port:Target.Port,
    State:TargetHealth.State,
    Reason:TargetHealth.Reason,
    Description:TargetHealth.Description
  }' \
  --output table

# Every target should show: State = healthy
# If State = unhealthy: check Reason column
#   HealthCheckFailed      → app not returning 200 on health path
#   Target.NotRegistered   → task just started, give it 30s
#   Target.Timeout         → app too slow to respond to health check
```

**Common ECS health check failures and fixes:**

| Reason | What's happening | Fix |
|---|---|---|
| `HealthCheckFailed` | Task running but app returning non-200 | Check task logs — app startup failed |
| `Target.NotRegistered` | Task not yet registered with ALB | Wait 30-60 seconds, normal during deploy |
| `Target.Timeout` | App not responding within timeout | App under load or JVM cold start — increase timeout |
| `Task.FailedHealthChecks` | ECS container health check failed | Check container logs immediately |

### 📊 B3 — CloudWatch Metrics

```bash
# ECS service CPU and memory utilization
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name CPUUtilization \
  --dimensions Name=ClusterName,Value=$CLUSTER Name=ServiceName,Value=$SERVICE \
  --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Average \
  --query 'sort_by(Datapoints, &Timestamp)[-3:].[Timestamp,Average]' \
  --output table

# ALB 5xx errors — did they spike after deploy?
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_Target_5XX_Count \
  --dimensions Name=LoadBalancer,Value=app/my-alb/abc123 \
  --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Sum \
  --output table

# Target response time — is it slower than before?
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name TargetResponseTime \
  --dimensions Name=LoadBalancer,Value=app/my-alb/abc123 \
  --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics p99 \
  --output table
```

### 🔍 B4 — ECS Task Logs (CloudWatch Logs)

```bash
LOG_GROUP="/ecs/myapp"

# Get the most recent log stream (newest task)
STREAM=$(aws logs describe-log-streams \
  --log-group-name $LOG_GROUP \
  --order-by LastEventTime \
  --descending \
  --max-items 1 \
  --query 'logStreams[0].logStreamName' \
  --output text)

echo "Checking log stream: $STREAM"

# Last 5 minutes of logs
aws logs get-log-events \
  --log-group-name $LOG_GROUP \
  --log-stream-name "$STREAM" \
  --start-time $(($(date +%s%3N) - 300000)) \
  --query 'events[*].message' \
  --output text | grep -iE "error|exception|failed|fatal" | head -20

# CloudWatch Insights — search across ALL tasks at once
aws logs start-query \
  --log-group-name $LOG_GROUP \
  --start-time $(($(date +%s) - 300)) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message
    | filter @message like /(?i)(error|exception|fatal)/
    | sort @timestamp desc
    | limit 20'
# Then: aws logs get-query-results --query-id <id-from-above>
```

### 🔄 B5 — ECS Circuit Breaker Status

```bash
# Did the ECS deployment circuit breaker trigger?
aws ecs describe-services \
  --cluster $CLUSTER \
  --services $SERVICE \
  --query 'services[0].deployments[0].{
    RolloutState:rolloutState,
    RolloutReason:rolloutStateReason,
    FailedTasks:failedTasks
  }'

# RolloutState values:
# IN_PROGRESS  → Still deploying (normal)
# COMPLETED    → Deploy succeeded ✅
# FAILED       → Circuit breaker triggered — check failedTasks count ❌
```

### ↩️ B6 — ECS Rollback Command

```bash
# Get current revision number and decrement by 1
FAMILY=$(aws ecs describe-services \
  --cluster $CLUSTER --services $SERVICE \
  --query 'services[0].taskDefinition' --output text | cut -d: -f1 | cut -d/ -f2)

CURRENT_REV=$(aws ecs describe-services \
  --cluster $CLUSTER --services $SERVICE \
  --query 'services[0].taskDefinition' --output text | cut -d: -f2)

PREV_REV=$((CURRENT_REV - 1))

aws ecs update-service \
  --cluster $CLUSTER \
  --service $SERVICE \
  --task-definition $FAMILY:$PREV_REV \
  --force-new-deployment

aws ecs wait services-stable --cluster $CLUSTER --services $SERVICE
echo "Rolled back to $FAMILY:$PREV_REV"
```

---

## Section C — EC2 + Tomcat / JBoss / Spring Boot

This is the real world. Plenty of companies still run Java apps on EC2 servers. Financial services, healthcare, government, enterprises that started in 2008 and never fully migrated. If you inherited one of these — this section is for you.

The checks here are different because **there's no orchestrator watching your back**. No Kubernetes restarting failed containers. No ECS circuit breaker. Just you, SSH, and `systemctl`.

### ⚡ C1 — First 60 Seconds: Is the Process Actually Running?

```bash
# SSH in — or use SSM Session Manager (no SSH key needed on modern setups)
aws ssm start-session --target i-0abc1234567890def

# ── TOMCAT ────────────────────────────────────────────────────────

# Is the Tomcat process alive?
ps aux | grep -i tomcat | grep -v grep
# Should show a java process with catalina in the path

# Or if managed by systemd:
systemctl status tomcat
# Active: active (running) = good
# Active: failed           = bad — check: journalctl -u tomcat -n 50

# What version is deployed? Check the WAR timestamp
ls -lh /opt/tomcat/webapps/myapp.war
# Timestamp should match your deploy time

ls -lh /opt/tomcat/webapps/myapp/
# Tomcat auto-explodes the WAR — this directory should exist and be fresh
# If it doesn't exist: Tomcat didn't finish deploying yet, wait 30 seconds

# ── JBOSS / WILDFLY ───────────────────────────────────────────────

systemctl status wildfly
ps aux | grep -i jboss | grep -v grep

# Check deployment status via JBoss CLI
/opt/jboss/bin/jboss-cli.sh --connect \
  --command="deployment-info --name=myapp.war"
# STATUS column should show: OK
# FAILED = your deployment is broken

# The deployment scanner drops marker files — check them
ls /opt/jboss/standalone/deployments/
# myapp.war.deployed   = deployed successfully ✅
# myapp.war.failed     = deployment failed ❌  → check server.log immediately
# myapp.war.isdeploying = still in progress, wait

# ── SPRING BOOT JAR (systemd service) ────────────────────────────

systemctl status myapp
# Look for: Active: active (running) since <your deploy time>

# Confirm WHICH version is running
ps aux | grep java | grep -v grep
# Should show: -jar /opt/myapp/myapp-1.3.0.jar
# Version in JAR name should match what you deployed
```

```
✅ Process running (ps aux confirms it)
✅ systemctl status = active (running)
✅ WAR/JAR timestamp matches deploy time
✅ JBoss: myapp.war.deployed exists, myapp.war.failed does NOT
```

### 🩺 C2 — Application Health (Is the JVM Actually Ready?)

Java apps have a gotcha Kubernetes engineers forget: **the process starts fast, but the JVM takes time to warm up**. A fresh deploy might pass a process check but still serve slow or errored responses for 30-90 seconds.

```
T+0s    Java process starts. Port not open yet.
T+5s    JVM loading classes. Spring context initializing.
T+15s   DB connection pool established. App technically "up".
T+30s   First requests are SLOW — JIT compiler not warmed up yet.
T+90s   JVM has compiled hot paths. Performance is nominal.
T+120s  THIS is when you run your performance check.

Checking at T+15s and seeing slow responses is normal.
Still slow at T+120s — that's your bug.
```

```bash
SERVER=10.0.1.25
PORT=8080

# Basic reachability
curl -sf http://$SERVER:$PORT/myapp/health

# Spring Boot Actuator endpoint (most Java apps should have this)
curl -sf http://$SERVER:$PORT/actuator/health | jq .
# {
#   "status": "UP",
#   "components": {
#     "db": {"status": "UP"},
#     "diskSpace": {"status": "UP"}
#   }
# }

# Check individual dependencies
curl -sf http://$SERVER:$PORT/actuator/health/db | jq .
# {"status":"UP","details":{"database":"PostgreSQL",...}}

# Tomcat Manager (if enabled)
curl -su admin:$TOMCAT_PASS \
  "http://$SERVER:$PORT/manager/text/list"
# /myapp:running:0:myapp   ← running, 0 active sessions yet
# /myapp:stopped:...       ← WAR didn't deploy correctly

# JBoss management API
curl -sf http://$SERVER:9990/management/deployment/myapp.war \
  -u admin:$JBOSS_ADMIN_PASS | jq '.status'
# "OK" = deployed fine
```

### 📊 C3 — JVM Metrics (The Part Nobody Checks But Should)

This is the most important section for Java apps and the most commonly skipped. A memory leak or thread explosion in new code won't show up in any dashboard for 20 minutes — but it will show up here right now.

```bash
PID=$(ps aux | grep java | grep -v grep | awk '{print $2}' | head -1)

# ── SPRING BOOT ACTUATOR (the easy path) ─────────────────────────

# Heap memory used — is it reasonable? Is it climbing?
curl -sf http://$SERVER:$PORT/actuator/metrics/jvm.memory.used?tag=area:heap | \
  jq '.measurements[0].value / 1048576 | floor | tostring + " MB heap used"'
# Run this twice, 2 minutes apart. Climbing = memory leak in new code.

# Thread count — abnormally high is a thread leak
curl -sf http://$SERVER:$PORT/actuator/metrics/jvm.threads.live | \
  jq '.measurements[0].value | tostring + " live threads"'
# Normal: 20-100   Suspicious: > 300   Alarm: > 500

# Active HTTP threads — how busy is Tomcat right now?
curl -sf http://$SERVER:$PORT/actuator/metrics/tomcat.threads.busy | \
  jq '.measurements[0].value | tostring + " busy threads"'

# ── JSTACK / JMAP (when Actuator isn't available) ────────────────

# Thread dump — find blocked or stuck threads
sudo -u tomcat jstack $PID | \
  grep -E "java.lang.Thread.State:|BLOCKED|waiting on" | \
  head -30
# Many BLOCKED = lock contention = possible deadlock in new code
# Many "waiting on" condition = normal
# 0 threads = process is frozen

# Heap summary — quick sanity check
sudo -u tomcat jmap -heap $PID 2>/dev/null | \
  grep -E "used|capacity|Heap|MaxHeapSize"
# If used / capacity > 90% right after deploy = memory issue

# GC activity — is garbage collection thrashing?
sudo -u tomcat jstat -gcutil $PID 1000 5
# S0   S1   E    O    M    YGC  YGCT  FGC  FGCT
# 0.0  45.2 67.1 32.4 95.1  12  0.45   0   0.00
#                ↑ O = Old gen. If this climbs toward 100% = memory leak.
#                                FGC = Full GC. Rising every minute = heap dying.
```

**JVM signals that mean something broke:**

```
SIGNAL                              MEANING
──────────────────────────────────  ─────────────────────────────────────
Heap > 90% and climbing             Memory leak introduced in new code
FGC count increasing every minute   JVM is fighting to stay alive — dying slowly
Thread count > 500                  Thread leak in new code — executor not shut down
Many BLOCKED threads in jstack      Deadlock or DB lock contention
OutOfMemoryError in logs            Already dead, just doesn't know it yet
Metaspace full                      Class loading leak — common after hot redeployment
```

### 🪵 C4 — Application Logs

Log locations differ by setup. Here's where to look for each Java server:

```bash
TOMCAT_HOME=/opt/tomcat   # Ubuntu: /var/lib/tomcat9

# ── TOMCAT ────────────────────────────────────────────────────────

# Tomcat's main log — startup errors, class loading failures, unhandled exceptions
tail -100 $TOMCAT_HOME/logs/catalina.out | \
  grep -iE "SEVERE|ERROR|Exception" | tail -30

# Errors SINCE your deploy (filter by time)
grep -A3 "$(date +'%d-%b-%Y %H:')" $TOMCAT_HOME/logs/catalina.out | \
  grep -iE "SEVERE|ERROR" | head -20

# Access log — are requests actually hitting the new deployment?
tail -20 $TOMCAT_HOME/logs/localhost_access_log.$(date +%Y-%m-%d).txt

# Your app's own log (if it writes one)
tail -50 $TOMCAT_HOME/logs/myapp.log 2>/dev/null || \
tail -50 /var/log/myapp/application.log 2>/dev/null

# ── JBOSS / WILDFLY ───────────────────────────────────────────────

# Main server log — everything goes here
tail -100 /opt/jboss/standalone/log/server.log | \
  grep -iE "ERROR|WARN|Exception" | tail -30

# Post-deploy errors only (last 10 minutes)
awk -v ts="$(date -d '10 minutes ago' '+%H:%M')" \
  '$0 > ts' /opt/jboss/standalone/log/server.log | \
  grep -iE "ERROR|Exception" | head -20

# ── SPRING BOOT JAR (systemd) ─────────────────────────────────────

# journalctl is your best friend for systemd-managed services
journalctl -u myapp --since "10 minutes ago" | \
  grep -iE "ERROR|Exception|failed" | head -30

# Or if logging to a file:
tail -50 /var/log/myapp/spring.log | grep -iE "ERROR|Exception" | head -20
```

**Java-specific log patterns that mean rollback now:**

```
"ClassNotFoundException"            → Dependency missing from WAR/classpath
"NoSuchMethodError"                 → JAR version conflict — check your pom.xml
"Cannot open JDBC Connection"       → DB pool exhausted or credentials wrong
"Address already in use :8080"      → Old process not stopped before new one started
"Failed to start bean"              → Spring context failed to load — read the full stacktrace
"HikariPool: Connection not available" → DB connection pool exhausted
"java.net.SocketTimeoutException"   → Downstream service slow or dead
"PermGen / Metaspace OutOfMemory"   → Class loading leak — JVM needs a restart
```

### 🔧 C5 — Server-Level Resources

```bash
# CPU, memory, load average — the big three
top -bn1 | head -5
# load average: 0.45, 0.32, 0.28
# If load > number of CPUs (check: nproc) = system is overloaded

# Memory — JVMs are hungry, make sure there's headroom
free -h
#              total   used    free    available
# Mem:         7.7Gi   5.2Gi   0.4Gi   1.8Gi
# available < 500MB = dangerously low for a JVM app
# JVM will start swapping to disk — catastrophically slow requests

# Disk space — Java servers write logs aggressively
df -h /opt /var/log /tmp
# Use% > 85% = check log rotation policy immediately
# Full disk = your app will start throwing errors when it tries to log

# Is the app actually listening on its port?
ss -tlnp | grep java
# LISTEN  0  100  0.0.0.0:8080  ...  users:(("java",...))  ← healthy
# Nothing on port 8080 = app didn't bind to port (startup failed silently)

# Open file descriptors — Java apps can exhaust system FD limits
cat /proc/$PID/limits | grep "open files"
ls /proc/$PID/fd | wc -l
# If current FDs / limit > 80% — apps start throwing "too many open files"
```

### 🌐 C6 — Load Balancer (ALB/ELB in Front of EC2)

Most EC2-based Java apps sit behind an AWS load balancer. Check both the app AND the LB — they can fail independently.

```bash
# Which instances are healthy in the target group?
TG_ARN=$(aws elbv2 describe-target-groups \
  --query "TargetGroups[?contains(TargetGroupName,'myapp')].TargetGroupArn" \
  --output text)

aws elbv2 describe-target-health \
  --target-group-arn $TG_ARN \
  --output table
# Every target should show State = healthy
# If any show "unhealthy" — check your health check path and port in the TG config

# Classic ELB (yes, some still exist):
aws elb describe-instance-health \
  --load-balancer-name my-classic-elb \
  --output table

# 5xx errors at the ALB vs at your app — different things!
# ELB 5xx = load balancer itself is having trouble
# Target 5xx = your app is returning errors
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_ELB_5XX_Count \
  --dimensions Name=LoadBalancer,Value=app/my-alb/abc123 \
  --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 --statistics Sum --output table
```

### 🔄 C7 — Multi-Server Validation

If you run multiple EC2 instances behind the LB, validate each one before declaring success.

```bash
#!/bin/bash
# validate-fleet.sh
SERVERS=("10.0.1.10" "10.0.1.11" "10.0.1.12")
PORT=8080
HEALTH_PATH="/myapp/health"
ALL_GOOD=true

for SERVER in "${SERVERS[@]}"; do
  echo ""
  echo "── $SERVER ─────────────────────────────"

  # Process alive?
  if ! ssh ec2-user@$SERVER "pgrep -f catalina" > /dev/null 2>&1; then
    echo "  ❌ Process NOT running"
    ALL_GOOD=false; continue
  fi
  echo "  ✅ Process running"

  # Health endpoint?
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" --max-time 5 \
    "http://$SERVER:$PORT$HEALTH_PATH")
  [ "$STATUS" = "200" ] && echo "  ✅ Health: HTTP $STATUS" \
    || { echo "  ❌ Health: HTTP $STATUS"; ALL_GOOD=false; }

  # Error count in logs?
  ERRORS=$(ssh ec2-user@$SERVER \
    "grep -c 'SEVERE\|ERROR' /opt/tomcat/logs/catalina.out 2>/dev/null || echo 0")
  echo "  📋 Error lines in catalina.out: $ERRORS"
done

echo ""
$ALL_GOOD && echo "✅ All servers healthy" || { echo "❌ Issues found"; exit 1; }
```

### ↩️ C8 — Rollback on EC2

No `kubectl rollout undo`. This is manual — but fast if you prepared.

```bash
# ── TOMCAT WAR ROLLBACK ───────────────────────────────────────────
# Best practice: keep previous WAR alongside current during deploy

TOMCAT_HOME=/opt/tomcat
WEBAPPS=$TOMCAT_HOME/webapps

sudo systemctl stop tomcat
sudo rm -rf $WEBAPPS/myapp $WEBAPPS/myapp.war
sudo cp /opt/deploys/myapp-previous.war $WEBAPPS/myapp.war
sudo chown tomcat:tomcat $WEBAPPS/myapp.war
sudo systemctl start tomcat

sleep 20
curl -sf http://localhost:$PORT/myapp/health && echo "✅ Rollback successful"

# ── SPRING BOOT JAR ROLLBACK ──────────────────────────────────────
# Symlink pattern makes this a one-liner

sudo systemctl stop myapp
sudo ln -sf /opt/myapp/myapp-previous.jar /opt/myapp/myapp-current.jar
sudo systemctl start myapp

sleep 30   # Spring Boot needs ~30 seconds for startup
curl -sf http://localhost:$PORT/actuator/health | jq .status

# ── JBOSS ROLLBACK ────────────────────────────────────────────────

sudo cp /opt/deploys/myapp-previous.war \
  /opt/jboss/standalone/deployments/myapp.war
sudo touch /opt/jboss/standalone/deployments/myapp.war.dodeploy

sleep 15
ls /opt/jboss/standalone/deployments/
# Should show myapp.war.deployed — not myapp.war.failed
```

---

## Shared Checks — All Platforms

These run the same whether your app is in a pod, a task, or a Tomcat server.

### 🌐 End-to-End Smoke Test

```bash
API_URL="https://myapp.com"
PASS=0; FAIL=0

check() {
  local name=$1 expected=$2 url=$3
  actual=$(curl -s -o /dev/null -w "%{http_code}" --max-time 5 "$url")
  [ "$actual" = "$expected" ] \
    && { echo "  ✅ $name (HTTP $actual)"; PASS=$((PASS+1)); } \
    || { echo "  ❌ $name (expected $expected, got $actual)"; FAIL=$((FAIL+1)); }
}

echo "── Smoke Tests ─────────────────────────"
check "Public homepage"   "200" "$API_URL/"
check "Health endpoint"   "200" "$API_URL/health"
check "Auth rejects anon" "401" "$API_URL/api/v1/users/me"

# Response time check
TIME=$(curl -s -o /dev/null -w "%{time_total}" --max-time 5 "$API_URL/health")
(( $(echo "$TIME < 2.0" | bc -l) )) \
  && echo "  ✅ Response time: ${TIME}s" \
  || echo "  ⚠️  Response time: ${TIME}s — slow (JVM still warming up?)"

echo ""
echo "Result: $PASS passed / $FAIL failed"
[ $FAIL -gt 0 ] && exit 1
```

### 🗄️ Database (The Silent Killer — Every Platform)

```bash
# PostgreSQL / Aurora PostgreSQL
psql -h $DB_HOST -U $DB_USER $DB_NAME -c \
  "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"
# "idle in transaction" > 5 = connection leak in new code

# Long-running queries since deploy
psql -h $DB_HOST -U $DB_USER $DB_NAME -c \
  "SELECT pid, round(EXTRACT(EPOCH FROM now()-query_start)) AS secs, left(query,80)
   FROM pg_stat_activity
   WHERE state = 'active' AND query_start < now() - interval '5 seconds'
   ORDER BY secs DESC;"
# Anything here that wasn't here before = new slow query introduced

# MySQL / Aurora MySQL
mysql -h $DB_HOST -u $DB_USER -p$DB_PASS -e \
  "SHOW PROCESSLIST;" | grep -v Sleep | head -20

# Did the migration run? (Flyway — standard for Java apps)
psql -h $DB_HOST -U $DB_USER $DB_NAME -c \
  "SELECT version, description, success
   FROM flyway_schema_history
   ORDER BY installed_on DESC LIMIT 5;"
# Latest migration should show success = true

# Alembic (Python apps)
psql -h $DB_HOST -U $DB_USER $DB_NAME -c \
  "SELECT version_num FROM alembic_version;"
```

### 🔔 Documentation & Alerts

```bash
# A deploy that isn't documented never happened (until 2AM when it did)
cat << EOF
─── Deploy Record ────────────────────────
Time:      $(date)
Version:   $VERSION
Platform:  $PLATFORM
Deployed:  $USER
Commit:    $(git rev-parse --short HEAD 2>/dev/null || echo N/A)
Change:    $COMMIT_MESSAGE
──────────────────────────────────────────
EOF

# Slack notification
curl -sf -X POST "$SLACK_WEBHOOK" \
  -H "Content-Type: application/json" \
  -d "{\"text\":\"✅ Deployed $VERSION to production on $PLATFORM\"}"

# Grafana annotation — so on-call can see EXACTLY when you deployed
curl -sf -X POST "http://grafana.internal/api/annotations" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $GRAFANA_KEY" \
  -d "{\"text\":\"Deploy: $VERSION ($PLATFORM)\",\"tags\":[\"deploy\",\"production\"]}"
```

---

## The Soak Period Table

| Deploy Type | Soak Time | K8s / ECS | EC2 / Java |
|---|---|---|---|
| Config change only | 5 min | Watch pods/tasks | Watch logs |
| Bug fix, small scope | 10 min | Check error rate | Check catalina.out |
| New feature | 20 min | Metrics + smoke | JVM heap + smoke |
| DB migration included | 45 min | Full checklist | Full checklist + pg_stat_activity |
| Major refactor / framework upgrade | 60 min + canary | All eyes on it | All eyes on it |
| Friday afternoon | **Don't.** | N/A | Still no. |

---

## When Something Looks Wrong

```
SYMPTOM                   KUBERNETES              ECS                     EC2 / JAVA
──────────────────────    ──────────────────────  ──────────────────────  ──────────────────────────────
App not responding        kubectl describe pod    Check TG health         ss -tlnp | grep java
Error rate spiked         kubectl logs --since    CloudWatch Logs         tail catalina.out / server.log
Memory climbing           kubectl top pods        CloudWatch MemUtil      jmap -heap $PID
Slow responses            Prometheus P99          ALB TargetResponseTime  jstack $PID — look for BLOCKED
Old version still serving kubectl rollout status  Describe service tasks  ls -lh webapps/myapp.war
DB connections maxed      kubectl exec to psql    Task logs + psql        pg_stat_activity
Won't start               kubectl describe pod    CloudWatch task logs    journalctl -u myapp -n 100
Rollback needed           kubectl rollout undo    aws ecs update-service  cp previous.war + restart
```

---

## The Non-Negotiable Rules

```
1. Green pipeline ≠ healthy production. Always validate.

2. Java apps need 90 seconds to warm up.
   Don't declare success at T+15s.

3. Check logs ON THE SERVER, not just the dashboard.
   Some exceptions only appear in catalina.out.

4. If something looks even slightly wrong — roll back first,
   investigate second. Pride is not worth a 2AM page.

5. Document every deploy. Slack, Grafana annotation, a sticky note.
   Future you at 2AM will be grateful.

6. Never deploy on Friday afternoon.
   (You know this. You do it anyway. Stop.)
```




