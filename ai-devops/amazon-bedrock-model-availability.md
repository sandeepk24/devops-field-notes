# Amazon Bedrock: Why Your Model Isn't Available (And How to Fix It)

> 🧑‍💻 **Who this is for:** Cloud Engineers, DevOps Engineers, and Senior Engineers who are building or maintaining GenAI / Agentic AI apps on AWS using Amazon Bedrock.

---

## The Story Behind This Guide

You're building a GenAI app. Everything looks great — your code is clean, your IAM policy is in place, and you hit run. Then you get slapped with:

```
Model not available in this region
```

You stare at it. You *definitely* set up Bedrock. You *definitely* have IAM permissions. What gives?

Nine times out of ten, the answer is **region mismatch** — your IAM policy only allows models in `us-east-1`, but your app is pointing somewhere else, or the model you want simply isn't available in that region yet.

This guide explains *why* that happens, how Bedrock actually works under the hood, and how to design your setup so this never bites you again.

---

## First, Let's Clear Up a Big Misconception

**Bedrock is NOT a global API.**

A lot of people assume Amazon Bedrock works like a big cloud brain sitting somewhere that you just "call." It doesn't. Bedrock is a **regional AWS service** — just like RDS, SageMaker, or Lambda. Each AWS region runs its own completely independent copy of Bedrock, with its own set of available models.

This means:

- Calling `us-east-1` Bedrock and calling `us-west-2` Bedrock are two different things
- A model available in one region may not exist in another
- Your IAM policy must explicitly allow the region(s) you're calling

There's no invisible cross-region routing happening behind the scenes (unless you explicitly set that up — more on that later).

---

## How Bedrock's Architecture Actually Works

Here's a simplified view of what happens when your app calls Bedrock:

```
┌─────────────────────────────────────────────────┐
│              Your Application                    │
│     (Lambda, ECS, EC2, local dev, etc.)          │
└────────────────────┬────────────────────────────┘
                     │
                     │  HTTPS request to:
                     │  bedrock-runtime.us-east-1.amazonaws.com
                     ▼
┌─────────────────────────────────────────────────┐
│         Bedrock Regional Endpoint (us-east-1)    │
│                                                  │
│  ┌──────────────┐  ┌─────────────┐  ┌─────────┐ │
│  │ Claude 3.5   │  │ Meta Llama  │  │  Titan  │ │
│  │ Sonnet/Haiku │  │   3.x       │  │  Text   │ │
│  └──────────────┘  └─────────────┘  └─────────┘ │
│                                                  │
│  ┌──────────────┐  ┌─────────────┐  ┌─────────┐ │
│  │   Cohere     │  │  AI21 Labs  │  │Stability│ │
│  │   Models     │  │   Models    │  │   AI    │ │
│  └──────────────┘  └─────────────┘  └─────────┘ │
└─────────────────────────────────────────────────┘
```

Your request lands at a specific regional endpoint. That endpoint has a specific set of models available. If the model you're asking for isn't in that region — or if your IAM policy doesn't allow that region — the request fails.

### Why is Bedrock Regional Instead of Global?

This is intentional, not a limitation. Here's why AWS built it this way:

**1. Data Residency & Compliance**
Many companies (especially in healthcare, finance, and government) have legal requirements that their data never leaves a specific geographic boundary. If AWS silently routed your prompt to another country's region, you'd have a compliance violation before you even knew it happened. Regional isolation protects you.

**2. Capacity Planning**
Running large language models requires specialized hardware — NVIDIA GPUs, AWS Inferentia, AWS Trainium. AWS provisions this capacity per region. A model being "available" in a region literally means AWS has deployed enough inference hardware there to serve it reliably.

**3. Model Provider Agreements**
Anthropic, Meta, Cohere, and other providers have their own rollout schedules and licensing agreements. A new model might launch in `us-east-1` first, then expand to other regions over weeks or months. AWS can't just flip a global switch — they have to deploy infrastructure region by region.

---

## Model Availability Varies by Region — Always Check First

Not every model is available in every region. Here's a rough sense of what that looks like (always verify with the AWS docs or the CLI since this changes frequently):

| Model Family       | us-east-1 | us-west-2 | eu-west-1 | ap-southeast-1 |
|--------------------|-----------|-----------|-----------|----------------|
| Claude 3.5 Sonnet  | ✅        | ✅        | ✅        | ✅             |
| Claude 3 Opus      | ✅        | ✅        | ❌        | ❌             |
| Llama 3.x          | ✅        | ✅        | ✅        | ✅             |
| Amazon Titan Text  | ✅        | ✅        | ✅        | ✅             |
| Stability SDXL     | ✅        | ✅        | ❌        | ❌             |

> ⚠️ **Don't guess.** Always check programmatically before building around a specific model.

### Check What's Available in Your Region

```python
import boto3

client = boto3.client('bedrock', region_name='us-east-1')

response = client.list_foundation_models()
for model in response['modelSummaries']:
    print(f"{model['modelId']:55s}  Status: {model['modelLifecycle']['status']}")
```

### Check if a Specific Model is Available

```python
response = client.get_foundation_model(
    modelIdentifier='anthropic.claude-3-5-sonnet-20241022-v2:0'
)
print(response['modelDetails']['modelLifecycle']['status'])
# "ACTIVE" = you're good to go
# Anything else = it's not ready
```

### ⚠️ One More Gotcha: You Also Need to Enable Model Access

Even if a model exists in a region and your IAM looks right, you still won't be able to use it unless you've **manually enabled model access** in the AWS Bedrock console (or via API). This is a separate step that a lot of people miss.

Go to **AWS Console → Amazon Bedrock → Model access** and enable the models you need. You'll have to do this per-account, per-region.

---

## The IAM Side: What Your Policy Should Actually Look Like

This is where the original problem usually lives. Here's a policy that covers you properly for multi-region access:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "BedrockInvokeModels",
            "Effect": "Allow",
            "Action": [
                "bedrock:InvokeModel",
                "bedrock:InvokeModelWithResponseStream"
            ],
            "Resource": [
                "arn:aws:bedrock:us-east-1::foundation-model/*",
                "arn:aws:bedrock:us-west-2::foundation-model/*",
                "arn:aws:bedrock:us-east-1:*:inference-profile/us.*",
                "arn:aws:bedrock:us-west-2:*:inference-profile/us.*"
            ]
        },
        {
            "Sid": "BedrockReadOnly",
            "Effect": "Allow",
            "Action": [
                "bedrock:ListFoundationModels",
                "bedrock:GetFoundationModel",
                "bedrock:ListInferenceProfiles"
            ],
            "Resource": "*"
        }
    ]
}
```

**The rule to remember:** The region in the ARN (`arn:aws:bedrock:us-east-1::...`) must match the region your boto3 client is pointed at. Mismatch between these two = access denied, and the error message won't always make it obvious why.

**If you're using cross-region inference profiles** (covered below), you also need permissions for the underlying foundation model ARNs in *every* region the profile might route to.

---

## Three Patterns for Calling Bedrock in Production

Here are the three ways teams typically set up Bedrock, from simplest to most resilient.

---

### Pattern 1: Single Region, Fixed Model

This is the "get it working" approach. Good for dev. Fragile for prod.

```python
import boto3
import json

bedrock_runtime = boto3.client(
    'bedrock-runtime',
    region_name='us-east-1'  # pinned to one region
)

response = bedrock_runtime.invoke_model(
    modelId='anthropic.claude-3-5-sonnet-20241022-v2:0',
    contentType='application/json',
    accept='application/json',
    body=json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 4096,
        "messages": [{"role": "user", "content": "Fix this code..."}]
    })
)

result = json.loads(response['body'].read())
print(result['content'][0]['text'])
```

**When it's fine:** Prototyping, internal tools, single-region apps with no HA requirements.

**The risk:** If that region has a capacity issue or the model is temporarily unavailable, your app goes down completely. There's no safety net.

---

### Pattern 2: Primary + Fallback Region (Recommended for Production)

This is what most production teams should be running. You pick a primary region, and if it fails, you automatically retry in a fallback region.

```python
import boto3
import json
from botocore.exceptions import ClientError

# Define your preference order up front — make this configurable via env vars
MODEL_CONFIG = {
    "primary": {
        "region": "us-east-1",
        "model_id": "anthropic.claude-3-5-sonnet-20241022-v2:0"
    },
    "fallback": {
        "region": "us-west-2",
        "model_id": "anthropic.claude-3-5-sonnet-20241022-v2:0"
    }
}

def invoke_with_fallback(prompt: str) -> dict:
    """
    Try the primary region first.
    If it's throttled or unavailable, automatically fall back to the secondary.
    """

    for tier in ["primary", "fallback"]:
        region = MODEL_CONFIG[tier]["region"]
        model_id = MODEL_CONFIG[tier]["model_id"]

        try:
            client = boto3.client('bedrock-runtime', region_name=region)
            response = client.invoke_model(
                modelId=model_id,
                contentType='application/json',
                accept='application/json',
                body=json.dumps({
                    "anthropic_version": "bedrock-2023-05-31",
                    "max_tokens": 4096,
                    "messages": [{"role": "user", "content": prompt}]
                })
            )
            result = json.loads(response['body'].read())

            # Track which region actually served the request (useful for debugging)
            result['_served_from'] = region
            return result

        except ClientError as e:
            error_code = e.response['Error']['Code']

            # These are the errors worth retrying in another region
            retryable_errors = (
                'ThrottlingException',
                'ServiceUnavailableException',
                'ModelNotReadyException'
            )

            if error_code in retryable_errors:
                print(f"[WARN] {tier} region {region} unavailable ({error_code}), trying fallback...")
                continue
            else:
                # Something else is wrong (bad request, auth issue, etc.)
                # Don't silently retry — raise and let the caller handle it
                raise

    raise RuntimeError("All configured Bedrock regions exhausted. Check model availability and IAM permissions.")


# Usage
result = invoke_with_fallback("Help me debug this Python function...")
print(f"Response from: {result['_served_from']}")
print(result['content'][0]['text'])
```

**When to use this:** Any production workload. It's straightforward, you stay in control of where your data goes, and it gives you resilience without much added complexity.

---

### Pattern 3: Cross-Region Inference Profiles (AWS-Managed Routing)

This is the "let AWS handle it" approach. Instead of calling a model directly, you call an **inference profile** — and AWS automatically routes your request to whichever region has available capacity, within a defined geographic boundary.

```python
# Notice the model ID — it starts with "us." instead of a direct model name.
# That "us." prefix is what makes it a cross-region inference profile.

response = bedrock_runtime.invoke_model(
    modelId='us.anthropic.claude-3-5-sonnet-20241022-v2:0',
    #        ^^ "us." = inference profile covering US regions
    contentType='application/json',
    accept='application/json',
    body=json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 4096,
        "messages": [{"role": "user", "content": prompt}]
    })
)
```

Here's what happens under the hood:

```
┌─────────────────┐
│   Your App      │──▶  bedrock-runtime.us-east-1.amazonaws.com
└─────────────────┘
                              │
                              ▼
                 ┌────────────────────────┐
                 │   Inference Profile     │
                 │   "us.anthropic..."     │
                 │                        │
                 │  AWS routes to region  │
                 │  with free capacity:   │
                 │                        │
                 │  ✅  us-east-1         │
                 │  ✅  us-east-2         │
                 │  ✅  us-west-2         │
                 └────────────────────────┘
```

**The prefixes and what they mean:**
- `us.` — Routes within US regions only
- `eu.` — Routes within EU regions only

Your data never crosses outside the geographic boundary of the prefix. So `us.` profiles stay in the US, `eu.` profiles stay in Europe. This is what makes it compliant for most data residency requirements while still giving you multi-region resilience.

**To see what inference profiles are available in your account:**

```python
client = boto3.client('bedrock', region_name='us-east-1')
response = client.list_inference_profiles()

for profile in response['inferenceProfileSummaries']:
    print(
        f"ID: {profile['inferenceProfileId']}\n"
        f"Type: {profile['type']}\n"
        f"Status: {profile['status']}\n"
    )
```

---

## How to Decide Which Pattern to Use

Not every team needs cross-region inference. Here's a straightforward decision tree:

```
Do you have strict data residency / compliance requirements?
│
├── YES → Use Pattern 2 (Primary + Fallback).
│         Only allow the specific regions your compliance
│         team approves. If you use cross-region profiles,
│         make sure the prefix (us. / eu.) matches your
│         required geographic boundary.
│
└── NO → Do you need high availability or burst capacity handling?
         │
         ├── YES → Use Pattern 3 (Cross-Region Inference Profiles).
         │         Let AWS handle the routing. Just make sure
         │         your IAM covers all regions the profile
         │         might route to.
         │
         └── NO → Pattern 1 is fine for now.
                  But add monitoring and a runbook for
                  when the region has issues.
```

---

## The Pre-Deployment Checklist

Before you push any Bedrock-dependent code to production, run through this list. These are the most common failure points, in order of how often they cause problems:

| # | Check | How to Verify |
|---|-------|---------------|
| 1 | Model exists in your target region | `list_foundation_models()` in target region |
| 2 | Model access is enabled for your account | AWS Console → Bedrock → Model access |
| 3 | IAM policy resource ARNs include the correct region | Audit your policy, check region in ARN |
| 4 | boto3 client region matches IAM policy region | Check your client initialization |
| 5 | Inference profiles exist (if using them) | `list_inference_profiles()` |
| 6 | Service Quotas are sufficient | AWS Console → Service Quotas → Bedrock |

---

## Quick Reference: Common Errors and What They Actually Mean

| Error | What's Actually Wrong |
|-------|----------------------|
| `AccessDeniedException` | IAM policy doesn't cover this region or model ARN |
| `ValidationException: model not found` | Model isn't available in this region, or the model ID is wrong |
| `ResourceNotFoundException` | Model access not enabled for your account in this region |
| `ThrottlingException` | You've hit your quota for invocations per minute in this region |
| `ModelNotReadyException` | Model is available in the region but not ready to serve requests yet (usually temporary) |

---

## Summary

Here's the whole thing in plain English:

1. **Bedrock is regional** — there's no global endpoint. Each region is its own independent deployment.
2. **Not every model is in every region** — check before you build around a specific model.
3. **You need to enable model access** per account, per region — it's a separate step from IAM.
4. **Your IAM resource ARN region must match your client region** — this is the silent killer.
5. **For production**, use either the primary + fallback pattern (Pattern 2) or cross-region inference profiles (Pattern 3) depending on your HA and compliance requirements.
6. **Cross-region inference profiles** are AWS-managed multi-region routing. They stay within a geographic boundary (`us.` or `eu.`) so they're still compliant for most data residency requirements.

---

## Further Reading

- [Amazon Bedrock Model IDs and Availability](https://docs.aws.amazon.com/bedrock/latest/userguide/model-ids.html)
- [Cross-Region Inference in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles.html)
- [Setting Up IAM for Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/security-iam.html)
- [Amazon Bedrock Quotas](https://docs.aws.amazon.com/bedrock/latest/userguide/quotas.html)

---

*Found an error or want to add something? PRs welcome.*
