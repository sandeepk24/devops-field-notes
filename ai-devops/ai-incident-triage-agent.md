# AI Incident Triage Agent: From Alert Noise to Root Cause in Seconds

**Category:** AIOps · AI DevOps · SRE · Incident Response  
**Why this matters:** The average engineer spends 20–40 minutes on initial triage before even starting remediation. An AI triage agent compresses that to seconds — querying logs, correlating metrics, identifying the blast radius, and surfacing a hypothesis. This is not a future capability. It is the most immediately high-ROI application of LLMs in production DevOps today.

---

## 1. Core Concept

An **AI incident triage agent** is an autonomous or semi-autonomous system that responds to alerts by:

1. Receiving the incident trigger (PagerDuty webhook, CloudWatch alarm, Prometheus alert)
2. Gathering context — querying logs, metrics, traces, deployment history, runbooks
3. Reasoning across that context to form a root-cause hypothesis
4. Producing a structured triage report
5. (Optionally) taking a first-response action — restarting a pod, draining a node, opening a Jira ticket

The key distinction from traditional alert routing is **reasoning under uncertainty**. A rule-based system can route "CPU > 90%" to the infra team. An AI agent can read 500 log lines, notice that the CPU spike correlates with a specific deployment 12 minutes ago, cross-reference the change with the PR description, and output: *"High confidence this is caused by an N+1 query introduced in commit a3f2bc1 — the order-service is fetching customer records one-by-one inside a loop. Immediate mitigation: rollback the deployment. Fix: batch the customer fetch."*

### The three modes of operation

```
┌──────────────────────────────────────────────────────────┐
│  INFORM mode       │  Agent produces triage report only. │
│  (lowest risk)     │  Human reads and acts.              │
├──────────────────────────────────────────────────────────┤
│  ASSIST mode       │  Agent produces report + draft      │
│  (balanced)        │  remediation commands. Human        │
│                    │  approves before execution.         │
├──────────────────────────────────────────────────────────┤
│  ACT mode          │  Agent executes safe remediations   │
│  (highest risk)    │  automatically (restart pod,        │
│                    │  scale up, clear cache). Humans     │
│                    │  notified, can override.            │
└──────────────────────────────────────────────────────────┘
```

Start in **INFORM** mode. Build trust with the team before moving to ASSIST or ACT.

---

## 2. Production Engineering View

### What the agent needs to be useful

An AI triage agent is only as good as the context it can access. The tool-calling interface is the critical design surface — not the LLM choice.

```
Alert fires
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│                  Triage Agent Orchestrator               │
│                                                         │
│  1. Parse alert payload → extract service, time, type   │
│  2. Invoke tool: get_recent_logs(service, window=10min) │
│  3. Invoke tool: get_metrics(service, window=30min)     │
│  4. Invoke tool: get_recent_deployments(service)        │
│  5. Invoke tool: get_runbook(alert_type)                │
│  6. Invoke tool: get_error_rate_trend(service)          │
│  7. LLM reasons over all gathered context              │
│  8. Produce structured triage report                   │
│  9. POST report to Slack + PagerDuty note              │
└─────────────────────────────────────────────────────────┘
```

### Tool surface — what to give the agent

| Tool | Data source | Why it matters |
|---|---|---|
| `get_recent_logs` | CloudWatch Logs / Splunk / Loki | Error messages, stack traces, timing clues |
| `get_metrics` | CloudWatch / Prometheus / Datadog | CPU, memory, latency, error rate, saturation |
| `get_recent_deployments` | ECS DescribeServices / K8s rollout history / GitHub API | Correlate timing — "did anything change?" |
| `get_runbook` | Confluence / Notion / S3 Markdown files | What the team already knows about this alert type |
| `get_upstream_status` | Health check endpoints / dependency map | Is the problem in this service or upstream? |
| `get_active_incidents` | PagerDuty / OpsGenie API | Is this alert part of a wider outage already being worked? |
| `list_error_patterns` | Log aggregation | Top error messages in window — pattern recognition |
| `get_change_diff` | GitHub / GitLab API | Pull the actual code change that went out recently |

### Why tool design is harder than it looks

**Problem 1: Window mismatch.** The alert fires at T+0. The root cause may have started at T-15m. If your `get_recent_logs` tool only looks back 5 minutes, the agent will never see the original error. Always look back at least 30–60 minutes for logs and metrics.

**Problem 2: Log volume.** A busy service can emit millions of log lines per hour. Passing raw logs to an LLM is expensive and exceeds context limits. Pre-filter: deduplicate identical lines, keep only ERROR/WARN level, extract stack trace roots. Give the agent a summary, not a firehose.

**Problem 3: Hallucination without grounding.** If the agent can't find relevant context, it will sometimes invent a plausible explanation. Every triage report must include a **confidence score** and the **specific evidence** that supports each claim. If the evidence is thin, the report should say so explicitly.

**Problem 4: Time zone and timestamp normalization.** Logs from different services use different timestamp formats and time zones. Normalize everything to UTC before handing to the agent.

---

## 3. Tools and Technologies

| Layer | Tool options |
|---|---|
| LLM backend | Claude claude-opus-4-8 (deep reasoning), Claude Haiku (fast triage), Bedrock-hosted models |
| Agent framework | AWS Strands Agents, LangGraph, custom Python with tool loop |
| Log ingestion | CloudWatch Logs Insights, Splunk REST API, Loki HTTP API, Elasticsearch |
| Metrics | CloudWatch GetMetricData, Prometheus HTTP API, Datadog Metrics API |
| Alert trigger | PagerDuty webhooks, CloudWatch Alarm SNS → Lambda, Prometheus Alertmanager webhook |
| Output delivery | Slack (Block Kit), PagerDuty incident note, Jira ticket, email |
| Deployment history | ECS DescribeServices, `kubectl rollout history`, GitHub Deployments API |
| Runbook store | S3 Markdown files, Confluence API, Notion API |

---

## 4. Architecture Pattern

### End-to-end: CloudWatch Alarm → Lambda Triage Agent → Slack

```
CloudWatch Alarm (fires)
        │
        ▼ SNS notification
┌───────────────────┐
│   AWS Lambda      │  ← triggered by SNS
│   triage_agent.py │
│                   │
│  1. Parse alarm   │
│  2. Call tools    │──── CloudWatch Logs Insights
│     in parallel   │──── CloudWatch GetMetricData
│                   │──── GitHub Deployments API
│  3. LLM call      │──── Claude claude-opus-4-8 (Bedrock)
│     with context  │
│  4. Structure     │
│     output        │
└────────┬──────────┘
         │
         ├──→ Slack #incidents (structured report)
         ├──→ PagerDuty incident note
         └──→ Jira ticket (if P1/P2)
```

### Triage report output schema

```json
{
  "incident_id": "INC-20260614-001",
  "service": "order-service",
  "alert_type": "HighErrorRate",
  "severity": "P2",
  "triage_confidence": "high",
  "triggered_at": "2026-06-14T14:32:00Z",
  "hypothesis": "N+1 query pattern introduced in deployment a3f2bc1 at 14:20 UTC causing database connection pool exhaustion",
  "evidence": [
    "Error rate spiked from 0.1% to 18% at 14:21 UTC",
    "Deployment a3f2bc1 (order-service v2.3.1) rolled out at 14:20 UTC",
    "CloudWatch logs show 'connection pool timeout' errors starting 14:21 UTC",
    "DB connection count (CloudWatch) jumped from 45 to 195 at 14:21 UTC",
    "No upstream services show elevated error rates"
  ],
  "blast_radius": "All orders endpoints — /api/v1/orders/* returning 503",
  "immediate_action": "Rollback order-service to v2.3.0",
  "rollback_command": "aws ecs update-service --cluster prod --service order-service --task-definition order-service:142 --force-new-deployment",
  "follow_up": "Review PR #847 for N+1 query in CustomerRepository.findByOrderIds()",
  "runbook_link": "s3://runbooks/order-service/high-error-rate.md",
  "related_incidents": ["INC-20260601-018 — same symptom, previous release"]
}
```

---

## 5. Common Mistakes

### Mistake 1: Trusting the agent's output without verifying evidence
The agent lists its evidence. Read it. If the evidence says "deployment at 14:20 correlates with alert at 14:21" — verify that timestamp yourself before executing a rollback. An AI can hallucinate a non-existent deployment.

**Fix:** Every triage report must link to the raw evidence (log query link, metric chart URL). Engineers should click through before acting on recommendations.

### Mistake 2: Building ACT mode before INFORM mode earns trust
Teams that skip INFORM and jump straight to automated remediation create an agent that can make an incident worse. A restart loop caused by a misconfigured agent at 3am is harder to diagnose than the original incident.

**Fix:** Run in INFORM mode for 4–6 weeks. Track accuracy: how often was the agent's hypothesis correct? Build the trust baseline before enabling actions.

### Mistake 3: Giving the agent write tools too early
A tool like `restart_service` or `rollback_deployment` must be gated by human approval until the agent's accuracy is proven. The risk is asymmetric: a correct auto-remediation saves 10 minutes; a wrong one creates a new incident.

**Fix:** In ASSIST mode, generate the command but require a human to paste it. In ACT mode, limit to safe actions (scale up, clear cache, force health check) and exclude irreversible ones (rollback, database operations).

### Mistake 4: Not deduplicating or rate-limiting agent invocations
A flapping alert that fires 20 times in 5 minutes will trigger 20 parallel agent runs, each making expensive LLM and API calls. This burns money and floods Slack.

**Fix:** Deduplicate on `(service, alert_type)` with a 5-minute cooldown. Only create a new triage report if the alert clears and re-fires.

### Mistake 5: Ignoring LLM context window limits
A 30-minute CloudWatch Logs Insights query for a busy service might return 50,000 lines. Passing this to Claude hits the context limit and costs a lot.

**Fix:** Pre-process before sending to LLM: top 10 unique error messages by count, summary statistics (total errors, error rate, p95 latency), not raw logs.

### Mistake 6: No fallback when tools fail
If CloudWatch Logs Insights times out during an outage (high load on AWS), the agent gets no log context and produces a useless report.

**Fix:** Build graceful degradation. If a tool returns an error, the agent should note which context is missing and lower its confidence accordingly rather than making up an explanation.

---

## 6. Hands-on Lab

**Goal:** Build a working AI incident triage agent in Python that receives a simulated CloudWatch alarm and produces a structured triage report using Claude.

### Prerequisites

```bash
pip install anthropic boto3 requests python-dotenv --break-system-packages
```

```bash
# .env
ANTHROPIC_API_KEY=your_key_here
AWS_REGION=us-east-1
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
```

### Step 1: Define the tools

```python
# tools.py
import boto3
import json
from datetime import datetime, timedelta, timezone

logs_client = boto3.client('logs', region_name='us-east-1')
cw_client   = boto3.client('cloudwatch', region_name='us-east-1')


def get_recent_logs(service_name: str, log_group: str, window_minutes: int = 30) -> dict:
    """
    Query CloudWatch Logs Insights for recent errors.
    Returns top error patterns, not raw lines.
    """
    end_time   = datetime.now(timezone.utc)
    start_time = end_time - timedelta(minutes=window_minutes)

    query = f"""
        fields @timestamp, @message
        | filter @logStream like '{service_name}'
        | filter @message like /ERROR|WARN|Exception|timeout|5[0-9][0-9]/
        | stats count(*) as occurrences by @message
        | sort occurrences desc
        | limit 20
    """

    try:
        response = logs_client.start_query(
            logGroupName=log_group,
            startTime=int(start_time.timestamp()),
            endTime=int(end_time.timestamp()),
            queryString=query,
        )
        query_id = response['queryId']

        import time
        for _ in range(15):
            result = logs_client.get_query_results(queryId=query_id)
            if result['status'] in ('Complete', 'Failed', 'Cancelled'):
                break
            time.sleep(2)

        rows = result.get('results', [])
        patterns = [
            {
                'message': next((f['value'] for f in row if f['field'] == '@message'), ''),
                'count':   int(next((f['value'] for f in row if f['field'] == 'occurrences'), 0)),
            }
            for row in rows
        ]
        return {'status': 'ok', 'error_patterns': patterns, 'window_minutes': window_minutes}

    except Exception as e:
        return {'status': 'error', 'message': str(e), 'error_patterns': []}


def get_metrics_summary(service_name: str, window_minutes: int = 30) -> dict:
    """
    Get key metrics for the service: error rate, latency, request count.
    Uses CloudWatch GetMetricData.
    """
    end_time   = datetime.now(timezone.utc)
    start_time = end_time - timedelta(minutes=window_minutes)

    try:
        response = cw_client.get_metric_data(
            MetricDataQueries=[
                {
                    'Id': 'error_rate',
                    'Expression': f'(errors / requests) * 100',
                    'Label': 'ErrorRate',
                },
                {
                    'Id': 'errors',
                    'MetricStat': {
                        'Metric': {
                            'Namespace': 'ECS/ContainerInsights',
                            'MetricName': 'TaskCount',
                            'Dimensions': [{'Name': 'ServiceName', 'Value': service_name}],
                        },
                        'Period': 60,
                        'Stat': 'Sum',
                    },
                    'ReturnData': False,
                },
                {
                    'Id': 'requests',
                    'MetricStat': {
                        'Metric': {
                            'Namespace': 'ECS/ContainerInsights',
                            'MetricName': 'RunningTaskCount',
                            'Dimensions': [{'Name': 'ServiceName', 'Value': service_name}],
                        },
                        'Period': 60,
                        'Stat': 'Average',
                    },
                    'ReturnData': False,
                },
            ],
            StartTime=start_time,
            EndTime=end_time,
        )
        return {'status': 'ok', 'metrics': response.get('MetricDataResults', [])}
    except Exception as e:
        return {'status': 'error', 'message': str(e), 'metrics': []}


def get_recent_deployments(service_name: str, cluster: str = 'prod') -> dict:
    """
    Get recent ECS service deployments and their timing.
    """
    ecs_client = boto3.client('ecs', region_name='us-east-1')
    try:
        response = ecs_client.describe_services(
            cluster=cluster,
            services=[service_name],
        )
        service = response['services'][0]
        deployments = [
            {
                'id':           d['id'],
                'status':       d['status'],
                'task_def':     d['taskDefinition'].split('/')[-1],
                'created_at':   d['createdAt'].isoformat(),
                'running_count': d['runningCount'],
                'desired_count': d['desiredCount'],
            }
            for d in service.get('deployments', [])
        ]
        return {'status': 'ok', 'deployments': deployments}
    except Exception as e:
        return {'status': 'error', 'message': str(e), 'deployments': []}
```

### Step 2: Build the triage agent

```python
# triage_agent.py
import anthropic
import json
from tools import get_recent_logs, get_metrics_summary, get_recent_deployments

client = anthropic.Anthropic()

TOOLS = [
    {
        "name": "get_recent_logs",
        "description": "Query CloudWatch Logs for recent error patterns in a service. Returns top error messages by frequency, not raw log lines.",
        "input_schema": {
            "type": "object",
            "properties": {
                "service_name":    {"type": "string", "description": "ECS service name"},
                "log_group":       {"type": "string", "description": "CloudWatch log group path"},
                "window_minutes":  {"type": "integer", "description": "How many minutes to look back (default 30)", "default": 30},
            },
            "required": ["service_name", "log_group"],
        },
    },
    {
        "name": "get_metrics_summary",
        "description": "Retrieve error rate, latency p95, and request count metrics for a service from CloudWatch.",
        "input_schema": {
            "type": "object",
            "properties": {
                "service_name":   {"type": "string"},
                "window_minutes": {"type": "integer", "default": 30},
            },
            "required": ["service_name"],
        },
    },
    {
        "name": "get_recent_deployments",
        "description": "List recent ECS deployments for a service, including task definition versions and timing.",
        "input_schema": {
            "type": "object",
            "properties": {
                "service_name": {"type": "string"},
                "cluster":      {"type": "string", "default": "prod"},
            },
            "required": ["service_name"],
        },
    },
]

TOOL_MAP = {
    "get_recent_logs":       get_recent_logs,
    "get_metrics_summary":   get_metrics_summary,
    "get_recent_deployments": get_recent_deployments,
}


def run_triage_agent(alert_payload: dict) -> dict:
    """
    Run the triage agent for an incoming alert.
    Returns a structured triage report.
    """
    service     = alert_payload.get("service_name", "unknown-service")
    alert_type  = alert_payload.get("alert_type", "Unknown")
    severity    = alert_payload.get("severity", "P3")
    description = alert_payload.get("description", "")
    log_group   = alert_payload.get("log_group", f"/ecs/{service}")
    fired_at    = alert_payload.get("fired_at", "unknown time")

    system_prompt = """You are an expert SRE performing incident triage. You have access to tools 
to query logs, metrics, and deployment history. Your job is to:
1. Gather relevant context using the available tools
2. Identify the most likely root cause
3. Assess the blast radius
4. Recommend immediate action

Always use tools before forming conclusions. If a tool returns an error, note what context is 
missing and lower your confidence accordingly. Never invent evidence.

Return your final triage report as a JSON object with these fields:
- hypothesis (string): your root cause theory
- confidence (string): high/medium/low
- evidence (array of strings): specific observations supporting the hypothesis
- blast_radius (string): what is broken for whom
- immediate_action (string): what to do right now
- follow_up (string): what to investigate after stabilization
- missing_context (array of strings): what you couldn't access"""

    user_message = f"""Triage this incident:

Service: {service}
Alert type: {alert_type}
Severity: {severity}
Fired at: {fired_at}
Description: {description}
Log group: {log_group}

Use your tools to gather context, then produce a structured triage report."""

    messages = [{"role": "user", "content": user_message}]

    print(f"\n🚨 Triaging {alert_type} on {service} ({severity})")
    print("─" * 60)

    # Agentic loop — run until model stops calling tools
    while True:
        response = client.messages.create(
            model="claude-opus-4-8",
            max_tokens=4096,
            system=system_prompt,
            tools=TOOLS,
            messages=messages,
        )

        # Collect any text the model produced this turn
        for block in response.content:
            if hasattr(block, 'text'):
                print(f"[Agent] {block.text[:200]}...")

        if response.stop_reason == "end_turn":
            # Extract the JSON triage report from the final response
            for block in response.content:
                if hasattr(block, 'text'):
                    text = block.text
                    # Find JSON in the response
                    start = text.find('{')
                    end   = text.rfind('}') + 1
                    if start != -1 and end > start:
                        try:
                            return json.loads(text[start:end])
                        except json.JSONDecodeError:
                            pass
            return {"hypothesis": "Could not parse structured report", "confidence": "low"}

        if response.stop_reason == "tool_use":
            # Execute all tool calls the model requested
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    tool_name = block.name
                    tool_args = block.input
                    print(f"  → Calling tool: {tool_name}({tool_args})")

                    fn     = TOOL_MAP.get(tool_name)
                    result = fn(**tool_args) if fn else {"error": f"Unknown tool: {tool_name}"}
                    print(f"  ← Result: {str(result)[:100]}...")

                    tool_results.append({
                        "type":        "tool_result",
                        "tool_use_id": block.id,
                        "content":     json.dumps(result),
                    })

            # Append model turn + tool results to message history
            messages.append({"role": "assistant", "content": response.content})
            messages.append({"role": "user",      "content": tool_results})
```

### Step 3: Simulate an alert and run triage

```python
# simulate_incident.py
import json
from triage_agent import run_triage_agent

# Simulated CloudWatch alarm payload (as you'd receive from SNS)
alert = {
    "service_name":  "order-service",
    "alert_type":    "HighErrorRate",
    "severity":      "P2",
    "fired_at":      "2026-06-14T14:32:00Z",
    "description":   "Error rate exceeded 10% for 5 consecutive minutes. Current rate: 18.3%",
    "log_group":     "/ecs/prod/order-service",
    "cluster":       "prod",
}

report = run_triage_agent(alert)

print("\n" + "=" * 60)
print("TRIAGE REPORT")
print("=" * 60)
print(json.dumps(report, indent=2))

# Format for Slack
slack_message = {
    "blocks": [
        {
            "type": "header",
            "text": {"type": "plain_text", "text": f"🚨 AI Triage: {alert['alert_type']} — {alert['service_name']}"}
        },
        {
            "type": "section",
            "fields": [
                {"type": "mrkdwn", "text": f"*Severity:* {alert['severity']}"},
                {"type": "mrkdwn", "text": f"*Confidence:* {report.get('confidence', 'N/A')}"},
                {"type": "mrkdwn", "text": f"*Hypothesis:*\n{report.get('hypothesis', 'N/A')}"},
                {"type": "mrkdwn", "text": f"*Blast Radius:*\n{report.get('blast_radius', 'N/A')}"},
            ]
        },
        {
            "type": "section",
            "text": {"type": "mrkdwn", "text": f"*Immediate action:*\n`{report.get('immediate_action', 'N/A')}`"}
        },
        {
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": "*Evidence:*\n" + "\n".join(f"• {e}" for e in report.get('evidence', []))
            }
        },
    ]
}

print("\nSlack payload:")
print(json.dumps(slack_message, indent=2))
```

### Step 4: Run it

```bash
python simulate_incident.py
```

**Expected console output:**

```
🚨 Triaging HighErrorRate on order-service (P2)
────────────────────────────────────────────
[Agent] I'll gather context about this incident by checking logs, metrics, and recent deployments...
  → Calling tool: get_recent_deployments({'service_name': 'order-service', 'cluster': 'prod'})
  ← Result: {'status': 'ok', 'deployments': [{'id': 'ecs-svc-...', 'status': 'PRIMARY', ...}]}
  → Calling tool: get_recent_logs({'service_name': 'order-service', 'log_group': '/ecs/prod/order-service', 'window_minutes': 30})
  ← Result: {'status': 'ok', 'error_patterns': [{'message': 'connection pool timeout', 'count': 847}...]}
  → Calling tool: get_metrics_summary({'service_name': 'order-service', 'window_minutes': 30})
  ← Result: {'status': 'ok', 'metrics': [...]}
[Agent] Based on the gathered evidence, here is the triage report: {...}
```

---

## 7. Interview Talking Points

**"How would you design an AI incident triage system?"**

> "I'd start with the tool surface — what context can the agent actually access: logs, metrics, deployment history, upstream health, and runbooks. The LLM is just the reasoning layer; the tools are what make it useful. I'd design three operating modes: INFORM (report only), ASSIST (report plus draft commands), and ACT (automated safe remediations). I'd always deploy INFORM first, run it for 4–6 weeks, track accuracy against what engineers actually found, and only expand capabilities after trust is established."

**"What are the failure modes of AI triage agents?"**

> "The main ones: hallucinated evidence — the model invents a plausible root cause without grounding it in actual data, which is why every report must link to the specific log query or metric that supports it. Context gaps — if CloudWatch Logs Insights is slow during the outage, the agent has no log context and should say so rather than guess. Alert storms — a flapping alert fires 20 times and you trigger 20 parallel agent runs; you need deduplication and cooldown windows. And scope creep in ACT mode — the agent executes a rollback based on a wrong hypothesis and creates a second incident. Safe remediation lists need to be explicitly defined and audited."

**"How is this different from PagerDuty's AI features?"**

> "PagerDuty and similar tools offer pre-built AI triage, which is great for out-of-the-box use. Building your own agent gives you control over what tools the agent can call, how context is pre-processed before hitting the LLM, what your specific runbooks say, and the ability to integrate with internal systems that no SaaS product has connectors for. The custom approach also lets you tune accuracy for your specific service topology rather than relying on a general model."

**"How do you handle LLM hallucination in production triage?"**

> "Require citations. Every claim in the triage report must point to a specific evidence item that came from a tool call, not from the model's training data. The evidence array in the output schema is the key control — if the agent says 'deployment at 14:20 caused the error rate spike,' the evidence array must include the specific deployment timestamp and the specific error rate chart that support it. Engineers are trained to verify those links before acting."

**"What's your LLM selection criteria for triage?"**

> "For full root-cause reasoning on complex incidents I want a frontier model — claude-opus-4-8 or equivalent — because the quality of reasoning matters more than cost when someone's P1 is burning. For high-volume triage at low severity (every P4 alert, background categorization) I use a faster, cheaper model like Claude Haiku or a fine-tuned smaller model. The key is that the tool interface is identical regardless of which model is behind it — you can swap models without changing the agent architecture."

---

## 8. AI DevOps / AIOps / LLMOps Angle

### This IS the AIOps angle

This file is the AIOps use case. Additional depth:

**LLMOps concern: prompt stability.** Your system prompt is effectively production configuration. Version-control it. Track when it changes and what changed in triage quality afterwards. A casual edit to the system prompt can degrade accuracy.

**Evaluation pipeline.** Build a golden dataset: 30 real incidents with known root causes. Run the triage agent against them weekly. Track:
- Hypothesis accuracy (correct root cause identified)
- Evidence relevance (did the agent surface the right signals)
- Actionability (was the immediate action correct and executable)

```python
# eval_triage.py — weekly accuracy tracking
GOLDEN_INCIDENTS = [
    {
        "alert": {"service_name": "order-service", "alert_type": "HighErrorRate", ...},
        "expected_hypothesis": "N+1 query in CustomerRepository",
        "expected_action": "rollback",
    },
    # ... 29 more
]

def evaluate_accuracy(results: list[dict]) -> dict:
    correct = sum(
        1 for r, g in zip(results, GOLDEN_INCIDENTS)
        if g["expected_hypothesis"].lower() in r.get("hypothesis", "").lower()
    )
    return {
        "total": len(results),
        "correct": correct,
        "accuracy": f"{correct / len(results) * 100:.1f}%"
    }
```

**Agent memory.** A triage agent with memory of past incidents is dramatically more useful. If the same N+1 query issue fired last month, the agent should say so: "High confidence — this matches INC-20260601-018 which had the same root cause." See upcoming file: `agent-memory-design-for-devops-workflows.md`.

---

## 9. GitHub or LinkedIn Content Angle

### Git commit message

```
docs: add AI incident triage agent complete guide

Covers triage agent architecture, tool design, three operating modes
(inform/assist/act), production failure modes, and a working Python
implementation using Claude with CloudWatch + ECS tool integrations.

Includes eval framework for tracking triage accuracy over time.
```

**Thread structure:**
1. "Traditional triage: 20 minutes to understand what's happening"
2. "AI triage: 8 seconds to a structured hypothesis with evidence"
3. "The three modes: INFORM → ASSIST → ACT (never skip INFORM first)"
4. "The tool surface is everything — the LLM is just the reasoning layer"
5. "The failure modes that will bite you: hallucination, context gaps, alert storms"
6. "How to track accuracy before you trust it with automated remediations"

**CTA:** "Full Python implementation using Claude + CloudWatch + ECS in the repo. Link in comments."

**Tags:** #AIOps #IncidentResponse #SRE #AIDevOps #CloudEngineering #LLMOps

---

## 10. What I Should Practice Next

**Immediately:**
- Wire the lab above to a real CloudWatch alarm using SNS → Lambda
- Run INFORM mode on one real service for two weeks and track accuracy manually
- Build the golden dataset: capture 10 real incidents this month with their actual root causes

