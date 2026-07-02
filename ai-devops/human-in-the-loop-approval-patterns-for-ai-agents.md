# Human-in-the-Loop Approval Patterns for AI Agents

**Category:** AI DevOps / AIOps / LLMOps  
**Why this matters:** As AI agents take on more autonomous actions in production environments — triggering rollbacks, scaling infrastructure, patching services — blind trust is an outage waiting to happen. Human-in-the-loop (HITL) patterns give teams the control surface to approve, reject, or escalate high-risk agent decisions without sacrificing the speed benefits of automation. Getting this right is the difference between an AI DevOps system your organization will adopt and one that gets shut down after the first incident.

---

## 1. Core Concept

Human-in-the-loop (HITL) in the context of AI agents means inserting a human approval or review checkpoint at critical decision points in an agentic workflow — before the agent takes an irreversible or high-risk action.

This is not the same as human-in-the-loop in ML training (active learning). In DevOps agentic systems, HITL is an **operational control pattern** that answers: *"What must a human approve before the agent acts?"*

### Key design axes

**Risk-gated triggering** — not every agent action needs human approval. The HITL layer should be proportional to blast radius:

| Risk Level | Example Action | Approval Needed? |
|---|---|---|
| Low | Read CloudWatch metrics | No |
| Medium | Scale ECS task count up by 2 | Optional (auto with audit log) |
| High | Rolling restart of production service | Yes — Slack approval |
| Critical | Terminate instances, rollback DB migration | Yes — async with timeout + escalation |

**Approval channel design** — where does the human respond? Common patterns:
- Slack interactive message (button: Approve / Reject / Escalate)
- GitHub PR comment bot (AI opens PR, human merges)
- PagerDuty incident acknowledgment as implicit approval
- Custom web UI for platform teams
- Email with magic link (lower-trust environments)

**Timeout and fallback** — what happens if no human responds? The agent must have a safe default: fail closed (do nothing), escalate to on-call, or auto-approve with elevated alerting.

**Audit trail** — every HITL decision must be recorded: who approved, when, the agent's reasoning, and the action taken. This is the compliance artifact.

---

## 2. Production Engineering View

Production HITL systems must handle asynchronous communication between an AI agent (synchronous process) and a human (asynchronous responder).

### The async handshake problem

Agents using LLMs typically run in a request-response model. HITL breaks this because:
- Agent calls a tool → needs approval → must **pause and wait**
- Human may take seconds or 20 minutes
- Agent process cannot block a thread for 20 minutes

**Solution:** Decouple the approval workflow using a **durable state store** (DynamoDB, Redis, or Postgres) and a **callback mechanism** (webhook, polling, or event bus).

```
Agent → writes pending approval to DB → sends Slack message → suspends
Human clicks Approve → Slack webhook fires → updates DB record
Agent wakes via polling or event → reads approval → executes action
```

### Approval state machine

Every pending action should be modeled as a state machine:

```
PENDING → APPROVED → EXECUTING → COMPLETED
       → REJECTED → ABORTED
       → TIMED_OUT → ESCALATED
       → ESCALATED → APPROVED (by senior on-call)
                   → REJECTED
```

### Idempotency is non-negotiable

If the agent retries an approved action (network blip, restart), it must not double-execute. Every action must carry a unique idempotency key tracked in the state store.

### Who can approve what

Implement an authorization layer on the approval endpoint:
- Level 1: Any on-call engineer (scale up, restart)
- Level 2: Team lead or SRE lead (rollback, data ops)
- Level 3: Incident commander or VP (production DB changes, multi-region failover)

---

## 3. Tools and Technologies

**Orchestration and state:**
- **AWS Step Functions** — native HITL with `waitForTaskToken` pattern; pauses execution for days if needed
- **Temporal** — durable workflow engine with built-in signal/await for HITL
- **Prefect** — Python-native workflows with manual approval steps
- **Redis / DynamoDB** — lightweight state store for custom implementations

**Notification channels:**
- **Slack Bolt (Python)** — interactive messages with buttons, block kit payloads
- **PagerDuty Events API** — route AI agent alerts through existing on-call rotation
- **AWS SNS + Lambda** — decouple notification delivery from agent logic

**Audit and compliance:**
- **AWS CloudTrail** — if actions invoke AWS APIs
- **Splunk HEC** — stream approval events to SIEM
- **OpenTelemetry spans** — trace the full agent → approval → execution chain

**Agent frameworks with HITL support:**
- **Anthropic Claude + tool_use** — pause after tool call, inject human result
- **LangGraph** — interrupt nodes for human-in-the-loop breakpoints
- **Strands Agents (AWS)** — tool confirmation callbacks
- **AutoGen** — human proxy agent pattern

---

## 4. Architecture Pattern

### Pattern: Slack-Gated Agent Action with Durable State

```
┌─────────────────────────────────────────────────────────┐
│                    AI Agent (Lambda / ECS)               │
│                                                         │
│  detect_anomaly() → assess_risk() → [HITL GATE]         │
│                                          │               │
│                              risk_score > threshold?     │
│                                   YES ↓                  │
│                        create_approval_request()         │
│                        write to DynamoDB (PENDING)       │
│                        send Slack interactive msg        │
│                        suspend / return token            │
└───────────────────────────────────────────────────────┬─┘
                                                        │
              ┌─────────────────────────────────────────┘
              │   DynamoDB: approvals table
              │   {token, action, risk, status, approver, ts}
              │
┌─────────────▼──────────────────┐
│       Slack Bolt App (Lambda)   │
│                                 │
│  /hitl-callback endpoint        │
│  validates user role            │
│  updates DynamoDB → APPROVED    │
│  publishes to EventBridge       │
└─────────────┬───────────────────┘
              │
┌─────────────▼──────────────────┐
│    Agent Execution Lambda       │
│    triggered by EventBridge     │
│    reads approved action        │
│    executes with idempotency    │
│    writes audit log → Splunk    │
└─────────────────────────────────┘
```

### Pattern: AWS Step Functions `waitForTaskToken`

For enterprise use cases where the workflow spans hours and must survive Lambda cold starts:

```yaml
# Step Functions state definition
WaitForApproval:
  Type: Task
  Resource: arn:aws:states:::sqs:sendMessage.waitForTaskToken
  Parameters:
    QueueUrl: !Ref ApprovalQueue
    MessageBody:
      TaskToken.$: $$.Task.Token
      Action.$: $.proposed_action
      RiskScore.$: $.risk_score
  HeartbeatSeconds: 3600   # 1 hour timeout
  Next: ExecuteAction
```

The task token is embedded in the Slack message. When the human approves, the callback sends `SendTaskSuccess` with the token, resuming the workflow.

---

## 5. Common Mistakes

**1. Blocking the agent thread synchronously**  
Polling in a loop (`while not approved: sleep(5)`) inside a Lambda is wasteful and fragile. Use event-driven callbacks or Step Functions task tokens instead.

**2. No timeout handling**  
If no human responds in 30 minutes, the agent silently does nothing. Define explicit timeout behavior: auto-escalate to the next on-call tier, or safe-abort with a notification.

**3. Approval fatigue**  
Routing every trivial action through HITL trains humans to click Approve without reading. Define a clear risk rubric and only HITL for genuinely high-impact actions. If >80% of requests are approved without review, the threshold is too low.

**4. Missing authorization on the callback**  
Any engineer who gets the Slack message can approve by default. Implement role checks: validate the approver's Slack user ID against your LDAP/IAM group for that action class.

**5. No idempotency on execution**  
Network errors between approval and execution can cause the agent to re-trigger the same action. Always generate a unique action ID, check it before executing, and mark it complete atomically.

**6. Approval token leakage**  
Slack messages are searchable by the whole workspace. Never embed the raw task token in the message — store it server-side and use a short-lived reference ID in the message.

**7. No audit trail**  
Who approved the 3am production rollback? You'll need this for the postmortem and potentially for compliance. Log: action, risk score, agent reasoning, approver, timestamp, outcome.

---

## 6. Hands-on Lab

### Build a Slack-Gated HITL system for an ECS restart agent

**Prerequisites:** Python 3.11+, AWS credentials, Slack app with Bolt, boto3

#### Step 1: Install dependencies

```bash
pip install anthropic slack-bolt boto3 flask
```

#### Step 2: DynamoDB approvals table

```bash
aws dynamodb create-table \
  --table-name hitl-approvals \
  --attribute-definitions AttributeName=token,AttributeType=S \
  --key-schema AttributeName=token,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

#### Step 3: Core HITL agent with Anthropic SDK

```python
import anthropic
import boto3
import uuid
import json
import time
from slack_sdk import WebClient
from slack_sdk.models.blocks import (
    SectionBlock, ActionsBlock, ButtonElement, MarkdownTextObject
)

client = anthropic.Anthropic()
dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table("hitl-approvals")
slack = WebClient(token="xoxb-your-bot-token")
APPROVAL_CHANNEL = "#platform-approvals"


def assess_risk(action: dict) -> float:
    """Use Claude to score the risk of a proposed action."""
    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=256,
        messages=[{
            "role": "user",
            "content": f"""You are a DevOps risk assessor. Score the following action 0.0–1.0 for blast radius risk.
Action: {json.dumps(action)}
Reply with only a JSON object: {{"score": float, "reason": "string"}}"""
        }]
    )
    result = json.loads(response.content[0].text)
    return result["score"], result["reason"]


def request_human_approval(action: dict, risk_score: float, reason: str) -> str:
    """Write pending approval to DynamoDB and notify Slack. Returns approval token."""
    token = str(uuid.uuid4())
    
    # Store in DynamoDB
    table.put_item(Item={
        "token": token,
        "action": json.dumps(action),
        "risk_score": str(risk_score),
        "reason": reason,
        "status": "PENDING",
        "created_at": int(time.time()),
        "expires_at": int(time.time()) + 1800  # 30 min TTL
    })
    
    # Send Slack interactive message
    slack.chat_postMessage(
        channel=APPROVAL_CHANNEL,
        blocks=[
            SectionBlock(text=MarkdownTextObject(
                text=f"*🤖 AI Agent Approval Request*\n"
                     f"*Action:* `{action['type']}` on `{action['target']}`\n"
                     f"*Risk Score:* {risk_score:.2f}\n"
                     f"*Reason:* {reason}"
            )),
            ActionsBlock(elements=[
                ButtonElement(
                    text="✅ Approve",
                    action_id="hitl_approve",
                    value=token,
                    style="primary"
                ),
                ButtonElement(
                    text="❌ Reject",
                    action_id="hitl_reject",
                    value=token,
                    style="danger"
                )
            ])
        ]
    )
    return token


def wait_for_approval(token: str, timeout_seconds: int = 1800) -> str:
    """Poll DynamoDB for approval decision. Returns 'APPROVED' or 'REJECTED'."""
    deadline = time.time() + timeout_seconds
    while time.time() < deadline:
        response = table.get_item(Key={"token": token})
        item = response.get("Item", {})
        status = item.get("status", "PENDING")
        if status in ("APPROVED", "REJECTED"):
            return status
        time.sleep(10)  # Poll every 10 seconds
    # Timeout: mark as TIMED_OUT
    table.update_item(
        Key={"token": token},
        UpdateExpression="SET #s = :s",
        ExpressionAttributeNames={"#s": "status"},
        ExpressionAttributeValues={":s": "TIMED_OUT"}
    )
    return "TIMED_OUT"


def execute_ecs_restart(service_name: str, cluster: str):
    """Execute the approved ECS service restart."""
    ecs = boto3.client("ecs")
    ecs.update_service(
        cluster=cluster,
        service=service_name,
        forceNewDeployment=True
    )
    print(f"✅ Restarted ECS service {service_name} in {cluster}")


# --- Main agent workflow ---
def agent_workflow(service_name: str, cluster: str):
    action = {
        "type": "ecs_force_restart",
        "target": f"{cluster}/{service_name}",
        "triggered_by": "high_error_rate_anomaly"
    }

    risk_score, reason = assess_risk(action)
    print(f"Risk score: {risk_score:.2f} — {reason}")

    if risk_score < 0.4:
        print("Low risk — executing automatically")
        execute_ecs_restart(service_name, cluster)
    else:
        print(f"Risk {risk_score:.2f} > 0.4 — requesting human approval")
        token = request_human_approval(action, risk_score, reason)
        status = wait_for_approval(token)

        if status == "APPROVED":
            execute_ecs_restart(service_name, cluster)
        elif status == "REJECTED":
            print("❌ Action rejected by human approver")
        else:
            print("⏰ Approval timed out — escalating to on-call")


if __name__ == "__main__":
    agent_workflow("payments-api", "production")
```

#### Step 4: Slack callback handler (Flask)

```python
from flask import Flask, request
from slack_bolt.adapter.flask import SlackRequestHandler
from slack_bolt import App

bolt_app = App(token="xoxb-your-bot-token", signing_secret="your-signing-secret")
handler = SlackRequestHandler(bolt_app)
flask_app = Flask(__name__)


@bolt_app.action("hitl_approve")
def handle_approve(ack, body, client):
    ack()
    token = body["actions"][0]["value"]
    user = body["user"]["id"]
    
    # TODO: check user is in approved-approvers group
    table.update_item(
        Key={"token": token},
        UpdateExpression="SET #s = :s, approver = :u, approved_at = :t",
        ExpressionAttributeNames={"#s": "status"},
        ExpressionAttributeValues={":s": "APPROVED", ":u": user, ":t": int(time.time())}
    )
    client.chat_postMessage(channel=APPROVAL_CHANNEL, text=f"✅ Approved by <@{user}>")


@bolt_app.action("hitl_reject")
def handle_reject(ack, body, client):
    ack()
    token = body["actions"][0]["value"]
    user = body["user"]["id"]
    
    table.update_item(
        Key={"token": token},
        UpdateExpression="SET #s = :s, approver = :u",
        ExpressionAttributeNames={"#s": "status"},
        ExpressionAttributeValues={":s": "REJECTED", ":u": user}
    )
    client.chat_postMessage(channel=APPROVAL_CHANNEL, text=f"❌ Rejected by <@{user}>")


@flask_app.route("/slack/events", methods=["POST"])
def slack_events():
    return handler.handle(request)


if __name__ == "__main__":
    flask_app.run(port=3000)
```

#### Test it

```bash
# Trigger the agent
python agent_workflow.py

# In Slack: #platform-approvals will show the approval card
# Click Approve → agent proceeds to restart ECS
# Click Reject → agent aborts and logs the decision
```

---

## Interview Talking Points

**How do you design HITL for AI agents in production without creating bottlenecks?**

The key insight is that HITL should be risk-proportional, not action-proportional. A naive implementation routes every agent action through human review, which creates approval fatigue and negates the value of automation. The right design starts with a risk scoring model — I've used Claude itself to score proposed actions against a blast-radius rubric — and only gates actions above a defined threshold. Low-risk actions (read-only, reversible, bounded scope) run autonomously with just an audit log. High-risk actions (force deployments, data operations, cross-region failover) pause and notify the on-call engineer via Slack or PagerDuty with full context.

The second challenge is the async handshake. Agents are synchronous processes, but humans are asynchronous responders. The production pattern I use is a durable state store — DynamoDB for lightweight cases, Step Functions `waitForTaskToken` for enterprise workflows — where the agent writes a pending record and suspends. A Slack Bolt callback handler receives the human's decision, updates the record, and triggers the agent's next execution via EventBridge or a task token callback. This pattern survives Lambda restarts, scales horizontally, and provides a natural audit trail.

The third piece is authorization. Approval channels must validate that the approving human has the right to approve that class of action. Mapping Slack user IDs to IAM or LDAP groups and checking at callback time is the pattern I use, with the ruleset stored in a config file that's version-controlled and reviewed like any other policy.

**What happens when no human responds to an approval request?**

This is the timeout and escalation design. Every pending approval has a TTL. On expiry, the system has three options: fail-closed (abort the action and notify), auto-escalate to a senior on-call tier, or in very low-risk cases auto-approve with elevated alerting. I always default to fail-closed for anything touching production data. The escalation path should mirror your existing incident response runbook — if the primary on-call doesn't respond in 15 minutes, page the secondary. The AI agent shouldn't be smarter than your existing escalation policy.

---

##  AI DevOps / AIOps / LLMOps Angle

The HITL pattern pairs naturally with **agentic AI** frameworks where the agent uses tools. In Claude's tool_use API, you can model the approval request itself as a tool call — the agent calls `request_human_approval(action, risk_score)` and the tool handler is the HITL infrastructure. This makes the agent's decision to seek approval explicit in the trace, observable via OpenTelemetry spans, and testable in isolation.

The deeper LLMOps opportunity is using the approval decision dataset to continuously improve the risk scoring model. Every human approval or rejection is a labeled example: the action, the risk score the model assigned, and the ground truth (approved = lower risk than model thought, rejected = higher). Over time, fine-tuning or few-shot prompting on this dataset makes the model's risk assessments more calibrated, reducing the number of actions that need human review.

```python
import anthropic
import json

client = anthropic.Anthropic()

def score_action_risk_with_history(action: dict, approval_history: list[dict]) -> dict:
    """
    Score action risk using historical approval decisions as few-shot examples.
    approval_history: list of {action, risk_score, human_decision, notes}
    """
    # Build few-shot examples from recent history
    examples = "\n".join([
        f"Action: {h['action']['type']} on {h['action']['target']}\n"
        f"Score: {h['risk_score']}\nHuman decision: {h['human_decision']}\n"
        for h in approval_history[-5:]  # last 5 decisions
    ])
    
    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=512,
        system=(
            "You are a DevOps risk assessor. Use the historical decisions below "
            "to calibrate your risk scoring. Score 0.0 = safe to automate, "
            "1.0 = requires senior human approval.\n\n"
            f"Recent approval history:\n{examples}"
        ),
        messages=[{
            "role": "user",
            "content": (
                f"Score this proposed action:\n{json.dumps(action, indent=2)}\n\n"
                "Reply with JSON: {\"score\": float, \"reason\": str, "
                "\"similar_to\": str}"
            )
        }]
    )
    
    return json.loads(response.content[0].text)


# Example usage
history = [
    {"action": {"type": "ecs_restart", "target": "staging/api"}, 
     "risk_score": 0.3, "human_decision": "AUTO_APPROVED", "notes": "staging only"},
    {"action": {"type": "ecs_restart", "target": "prod/payments"}, 
     "risk_score": 0.7, "human_decision": "APPROVED", "notes": "business hours"},
    {"action": {"type": "rds_failover", "target": "prod/main-db"}, 
     "risk_score": 0.9, "human_decision": "ESCALATED", "notes": "required VP approval"},
]

new_action = {"type": "ecs_restart", "target": "prod/search-api"}
result = score_action_risk_with_history(new_action, history)
print(json.dumps(result, indent=2))
# Expected: score ~0.7, similar to prod/payments restart
```


---

##  What I Should Practice Next

Build the complete end-to-end flow locally: a Python agent that detects high ECS error rates (mock CloudWatch data), calls Claude to score the action, writes to DynamoDB, sends a Slack message with Approve/Reject buttons, waits for the callback, and executes or aborts. Run it with ngrok to expose the Slack callback endpoint.

Then extend it in two directions. First, swap the polling mechanism for Step Functions `waitForTaskToken` — this is the enterprise pattern and the one you'll explain in FAANG-level design interviews. Second, add the approval history feedback loop: after each decision, append to a JSONL file and use it as few-shot context in the next risk scoring call. After 20-30 decisions, compare the model's scores against human decisions to see if calibration is improving.

Topics to go deeper: **Temporal workflows for durable HITL** (more powerful than Step Functions for complex multi-step approvals), **LangGraph interrupt nodes** (declarative HITL inside agent graphs), and **policy-as-code for approval authorization** (using OPA or Cedar to define who can approve what, version-controlled and auditable).

---

**Suggested git commit message:**  
`docs: add HITL approval patterns for AI DevOps agents — Slack, DynamoDB, Step Functions, risk-scored gates`
