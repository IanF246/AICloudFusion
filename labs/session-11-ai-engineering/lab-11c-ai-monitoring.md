# Lab 11C: Production AI — Token Budgets, Cost Monitoring & Guardrails

**Session:** 11 — AI Engineering  
**Track:** AI Engineering  
**Difficulty:** Advanced  
**Estimated Time:** 45–55 minutes  
**Target Cert:** AWS Certified AI Practitioner

---

## Overview

Your AI chatbot works and has a personality. But in production, you need to answer harder questions: "How much is this costing me?", "What if a user sends a 10,000-word prompt?", and "How do I prevent the AI from saying something inappropriate?"

In this lab you'll add production-grade controls:

1. **A token budget** — reject requests that would burn too many tokens
2. **A metric filter** extracting token usage from logs into a CloudWatch custom metric
3. **A cost alarm** that fires if token usage spikes unexpectedly
4. **An AI dashboard** showing: requests, latency, tokens per request, and estimated cost
5. **Input validation** — reject prompts that are too long before they reach Bedrock

**The lesson:** Building AI is easy. Operating AI responsibly (cost-controlled, monitored, safe) is the hard part — and the part that separates hobbyists from professionals.

---

## Prerequisites

- ✅ Completed **Lab 11A** — the `workshop-ai-chatbot-lab11` function and role must exist
- ✅ AWS CLI authenticated

> **📌 If you cleaned up:** Follow Lab 11A Steps 3–5 to recreate everything.

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| AWS Lambda | Function execution | Always Free |
| Amazon Bedrock (Nova Micro) | AI inference | ~$0.01–0.02 for this lab |
| CloudWatch Custom Metrics | Token tracking metric | Always Free (10 metrics) |
| CloudWatch Dashboard | AI operations view | Always Free (3 dashboards) |

**⚠️ Estimated cost for this lab: under $0.03**

---

## Concepts

**Token Budget** — a limit you enforce in code: "if this request would use more than X input tokens, reject it before calling Bedrock." This prevents runaway costs from users pasting entire documents as prompts.

**Input Validation** — checking the user's input BEFORE calling the AI model. If the prompt is too long, contains forbidden content, or is empty, reject it immediately (no Bedrock call = no cost).

**Cost Estimation** — calculating approximate cost from token counts. Nova Micro: ~$0.035/million input tokens, ~$0.14/million output tokens. Your logs capture tokens per request → you can estimate cost per request and total daily cost.

**AI Operations Dashboard** — a CloudWatch dashboard designed for AI workloads. Beyond standard Lambda metrics (invocations, errors, duration), it shows: tokens per request, total tokens over time, and estimated cost.

---

## ⚠️ Reminder: How to Read Commands

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account number | `123456789012` |

---

## Lab Steps

### Step 1: Set Up

**Step 1a:** Set your profile and navigate to your project folder.

**Windows (PowerShell):**
```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
cd ~\Desktop\workshop-lab-11a
```

**macOS / Linux:**
```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
cd ~/Desktop/workshop-lab-11a
```

---

### Step 2: Add Input Validation and Token Budget

You'll update the Lambda to reject requests that are too long BEFORE calling Bedrock.

**Step 2a:** Open `handler.py` and add this validation block AFTER the `user_message = ...` line and BEFORE the `# Call Bedrock` comment:

```python
    # Input validation — reject prompts that are too long
    MAX_INPUT_CHARS = 500  # ~125 tokens max input
    if len(user_message) > MAX_INPUT_CHARS:
        logger.warning(json.dumps({
            "level": "WARNING",
            "event": "input_rejected",
            "request_id": request_id,
            "reason": "prompt_too_long",
            "input_length": len(user_message),
            "max_allowed": MAX_INPUT_CHARS
        }))
        return {
            "statusCode": 400,
            "headers": {
                "Content-Type": "application/json",
                "Access-Control-Allow-Origin": "*"
            },
            "body": json.dumps({
                "error": "Your message is too long. Please keep it under 500 characters.",
                "input_length": len(user_message),
                "max_allowed": MAX_INPUT_CHARS
            })
        }
```

**Step 2b:** Save `handler.py`.

> **What does this do?** If a user sends a prompt longer than 500 characters (~125 tokens), the function immediately returns an error WITHOUT calling Bedrock. No AI call = no cost. This is your first line of defence against prompt injection attacks and runaway costs.

**Step 2c:** Re-deploy:

**Windows (PowerShell):**
```powershell
Compress-Archive -Path handler.py -DestinationPath function.zip -Force
aws lambda update-function-code --function-name workshop-ai-chatbot-lab11 --zip-file fileb://function.zip --region us-east-1 --query "LastUpdateStatus" --output text
```

**macOS / Linux:**
```bash
zip function.zip handler.py
aws lambda update-function-code --function-name workshop-ai-chatbot-lab11 --zip-file fileb://function.zip --region us-east-1 --query "LastUpdateStatus" --output text
```

---

### Step 3: Test the Token Budget

**Step 3a:** Test with a normal short message. Edit `payload.json`:

```json
{"body": "{\"message\": \"What is S3?\"}"}
```

Invoke — **✅ should work normally** (short prompt passes validation).

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-ai-chatbot-lab11 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json; Get-Content response.json
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-ai-chatbot-lab11 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json && cat response.json
```

**Step 3b:** Test with a long message. Edit `payload.json` to include a very long prompt (copy this entire block):

```json
{"body": "{\"message\": \"Please provide an extremely detailed and comprehensive explanation of every single AWS service that exists, including all their features, pricing tiers, use cases, integrations, and historical context. I want you to cover compute, storage, networking, databases, analytics, machine learning, security, developer tools, and management services. For each one, explain the architecture, how it compares to competitors, and give me five real-world case studies. This should be at least 5000 words long with citations and references.\"}"}
```

Save and invoke.

**✅ You should see** a `400` status code response:
```json
{"error": "Your message is too long. Please keep it under 500 characters.", "input_length": 5XX, "max_allowed": 500}
```

> **🎯 The validation rejected the request before it reached Bedrock.** Zero tokens consumed, zero cost incurred. In production, this prevents users from pasting entire documents into your chatbot and burning your token budget.

---

### Step 4: Create a Token Usage Metric Filter

Like Session 10, you'll extract custom metrics from logs — but this time tracking **token usage** instead of API latency.

**Step 4a:** Create the metric transformation file. Open your text editor → new file. 📋 Copy and paste:

```json
[
    {
        "metricName": "TotalTokensUsed",
        "metricNamespace": "WorkshopAIChatbot",
        "metricValue": "$.total_tokens",
        "defaultValue": 0
    }
]
```

**Step 4b:** Save as `token-transform.json` in your `workshop-lab-11a` folder.

**Step 4c:** Create the metric filter. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws logs put-metric-filter --log-group-name "/aws/lambda/workshop-ai-chatbot-lab11" --filter-name token-usage --filter-pattern "{ $.event = ""bedrock_call_success"" }" --metric-transformations file://token-transform.json --region us-east-1
```

**macOS / Linux:**
```bash
aws logs put-metric-filter --log-group-name "/aws/lambda/workshop-ai-chatbot-lab11" --filter-name token-usage --filter-pattern '{ $.event = "bedrock_call_success" }' --metric-transformations file://token-transform.json --region us-east-1
```

**✅ No output means success.**

**Step 4d:** Invoke the chatbot a few times to generate metric data. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
1..5 | ForEach-Object { aws lambda invoke --function-name workshop-ai-chatbot-lab11 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json | Out-Null; Write-Host "Invoke $_ done" }
```

**macOS / Linux:**
```bash
for i in 1 2 3 4 5; do aws lambda invoke --function-name workshop-ai-chatbot-lab11 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json > /dev/null 2>&1; echo "Invoke $i done"; done
```

> **💡 Wait 2 minutes** for the metric filter to process the logs.

---

### Step 5: Create an AI Operations Dashboard

**Step 5a:** Create the dashboard definition. Open your text editor → new file. 📋 Copy and paste:

```json
{
    "widgets": [
        {
            "type": "metric",
            "x": 0,
            "y": 0,
            "width": 12,
            "height": 6,
            "properties": {
                "title": "AI Chatbot Invocations",
                "metrics": [["AWS/Lambda", "Invocations", "FunctionName", "workshop-ai-chatbot-lab11"]],
                "period": 60,
                "stat": "Sum",
                "region": "us-east-1"
            }
        },
        {
            "type": "metric",
            "x": 12,
            "y": 0,
            "width": 12,
            "height": 6,
            "properties": {
                "title": "Bedrock Latency (Lambda Duration ms)",
                "metrics": [["AWS/Lambda", "Duration", "FunctionName", "workshop-ai-chatbot-lab11"]],
                "period": 60,
                "stat": "Average",
                "region": "us-east-1"
            }
        },
        {
            "type": "metric",
            "x": 0,
            "y": 6,
            "width": 12,
            "height": 6,
            "properties": {
                "title": "Total Tokens Per Request",
                "metrics": [["WorkshopAIChatbot", "TotalTokensUsed"]],
                "period": 60,
                "stat": "Average",
                "region": "us-east-1"
            }
        },
        {
            "type": "metric",
            "x": 12,
            "y": 6,
            "width": 12,
            "height": 6,
            "properties": {
                "title": "Total Tokens (Cumulative)",
                "metrics": [["WorkshopAIChatbot", "TotalTokensUsed"]],
                "period": 300,
                "stat": "Sum",
                "region": "us-east-1"
            }
        }
    ]
}
```

**Step 5b:** Save as `dashboard.json` in your `workshop-lab-11a` folder.

**Step 5c:** Create the dashboard. 📋 Copy and paste (from the `workshop-lab-11a` folder):

```
aws cloudwatch put-dashboard --dashboard-name workshop-ai-chatbot-dashboard --dashboard-body file://dashboard.json --region us-east-1
```

**✅ You should see** `"DashboardValidationMessages": []`

**Step 5d: ✅ Console Checkpoint** — Open CloudWatch → Dashboards → `workshop-ai-chatbot-dashboard`. You should see four panels: invocations, latency, tokens per request, and cumulative tokens.

---

### Step 6: Create a Token Spike Alarm

Create an alarm that fires if token usage per request spikes (could indicate someone bypassing your input validation or a prompt injection attack).

📋 Copy and paste:

```
aws cloudwatch put-metric-alarm --alarm-name ai-chatbot-token-spike --metric-name TotalTokensUsed --namespace "WorkshopAIChatbot" --statistic Maximum --period 60 --threshold 500 --comparison-operator GreaterThanThreshold --evaluation-periods 1 --alarm-description "AI chatbot token usage spike - possible long prompt or injection" --region us-east-1
```

**✅ No output means success.**

> **What does this catch?** If any single request uses more than 500 tokens, the alarm fires. Your input validation (Step 2) should prevent this — but if someone finds a way around it, the alarm catches it as a second layer of defence.

---

### Step 7: Test the Full Stack

**Step 7a:** Send a normal request and verify everything works end-to-end:

```json
{"body": "{\"message\": \"Explain VPC in simple terms\"}"}
```

Save to `payload.json`, invoke, and confirm:
- ✅ AI responds correctly
- ✅ Token count is reasonable (< 200)
- ✅ Latency shown in response metadata

**Step 7b:** Check the alarm is healthy:

```
aws cloudwatch describe-alarms --alarm-names ai-chatbot-token-spike --region us-east-1 --query "MetricAlarms[0].[AlarmName,StateValue]" --output text
```

**✅ You should see:** `ai-chatbot-token-spike    OK` (or `INSUFFICIENT_DATA` if not enough data points yet)

---

## What You Just Did

| What You Built | Why It Matters |
|---------------|----------------|
| Input validation (500-char limit) | Prevents cost runaway from long prompts — rejects before calling Bedrock |
| Token usage metric filter | Tracks AI cost per-request automatically from logs |
| AI operations dashboard | Single view of chatbot health: traffic, speed, and token spend |
| Token spike alarm | Detects anomalous usage (possible abuse or prompt injection) |
| Defence in depth for AI | Validation (prevent) + monitoring (detect) + alarm (alert) |

**Key takeaways:**
- **Validate inputs BEFORE calling the AI** — the cheapest request is one you never make
- **Track tokens like you track money** — because they ARE money
- **Set hard limits (maxTokens) AND soft limits (input validation)** — belt and suspenders
- **AI observability = standard observability + token metrics** — the same patterns from Sessions 9–10 apply
- **In production, you'd add:** rate limiting per user, daily budget caps, and content filtering (Bedrock Guardrails)

---

## Cert Prep Callout

**Target Certification:** AWS Certified AI Practitioner

The AI Practitioner exam tests:
- Cost management for AI applications (token budgeting, monitoring)
- Input validation and guardrails for responsible AI
- CloudWatch integration with AI workloads
- Operational best practices for generative AI applications
- Security considerations: prompt injection, input sanitization

**Sample question type:** "A company's AI chatbot is generating unexpectedly high costs. What should they implement to control spending?"  
**Answer:** Implement input validation to reject overly long prompts before they reach the model, set maxTokens limits on responses, create CloudWatch metric filters to track token usage, and set alarms on token thresholds to detect spending spikes early.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| Metric filter not producing data | Filters only process new logs | Invoke the function several more times and wait 2 minutes |
| Filter pattern error on Windows | PowerShell quote issue | Use `""` (double-double-quotes): `"{ $.event = ""bedrock_call_success"" }"` |
| Dashboard shows "No data" | Metrics haven't published yet | Wait 2 minutes after invocations; set time range to "Last 1 hour" |
| Input validation not working | Old code still deployed | Verify `handler.py` has the `MAX_INPUT_CHARS` block, re-zip, re-deploy |
| Alarm in INSUFFICIENT_DATA | Not enough data points | Invoke several more times and wait for metric filter to process |

---

## Cleanup

**Clean up everything from Session 11:**

```
aws cloudwatch delete-alarms --alarm-names ai-chatbot-token-spike --region us-east-1
aws cloudwatch delete-dashboards --dashboard-names workshop-ai-chatbot-dashboard --region us-east-1
aws logs delete-metric-filter --log-group-name "/aws/lambda/workshop-ai-chatbot-lab11" --filter-name token-usage --region us-east-1
aws lambda delete-function --function-name workshop-ai-chatbot-lab11 --region us-east-1
aws logs delete-log-group --log-group-name /aws/lambda/workshop-ai-chatbot-lab11 --region us-east-1
aws iam delete-role-policy --role-name workshop-lab11-lambda-role --policy-name bedrock-invoke
aws iam detach-role-policy --role-name workshop-lab11-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name workshop-lab11-lambda-role
```

Delete the project folder:

**Windows (PowerShell):**
```powershell
cd ~\Desktop; Remove-Item -Recurse -Force workshop-lab-11a
```

**macOS / Linux:**
```bash
cd ~/Desktop && rm -rf workshop-lab-11a
```

---

## Help

If you get stuck, post in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** you received
4. Your **operating system** (Windows / Mac / Linux)
