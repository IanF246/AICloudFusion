# Lab 10C: Make the Bot Self-Healing — Fallback & Graceful Degradation

**Session:** 10 — Chatbot Observability  
**Track:** Solutions Architecture  
**Difficulty:** Advanced  
**Estimated Time:** 45–55 minutes  
**Target Cert:** AWS Solutions Architect – Associate (SAA)

---

## Overview

In Lab 10B, when the external API got slow, your chatbot got slow too — it still worked, but users waited 3+ seconds. In production, that's unacceptable. What if the API goes down entirely? Your bot would fail completely.

In this lab you'll make the chatbot **resilient**: when the external API is slow or down, it automatically switches to a **fallback response** instead of failing or hanging. The bot stays fast and responsive even when its dependency breaks.

By the end of this lab you will have:

1. **An SSM Parameter** that controls whether the bot uses the live API or a fallback
2. **A resilient handler** that checks the parameter and returns a cached response when the API is unhealthy
3. **Three observable states** on your dashboard: healthy requests, fallback requests, and failed requests
4. **Proof it works** — break the API, bot stays fast (fallback), fix it, bot returns to live mode

**The key lesson:** Good architecture doesn't just *detect* problems — it *tolerates* them. The alarm tells you something's wrong; the fallback keeps users happy while you fix it.

---

## Prerequisites

- ✅ Completed **Lab 10A** — the `workshop-chatbot-lab10` function must be deployed
- ✅ Completed **Lab 10B** — metric filters, dashboard, and alarm should exist
- ✅ AWS CLI authenticated

> **📌 If you cleaned up Lab 10B:** You need the function, role, metric filters, and dashboard back. Follow Lab 10A Steps 2–4, then Lab 10B Steps 2–6 to recreate everything.

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| AWS Lambda | Function execution | Always Free (1M requests/month) |
| AWS Systems Manager Parameter Store | Configuration storage | Always Free (standard parameters) |
| CloudWatch | Metrics + dashboard | Always Free (within limits) |

**Estimated cost for this lab: $0.00**

---

## Concepts

**Graceful Degradation** — when a system continues to function (perhaps with reduced features) even when a component fails. Instead of crashing when the trivia API is down, the chatbot returns a pre-stored response. Users get *something* rather than an error. Think of it like a restaurant that runs out of fresh pasta and offers dried pasta instead of closing.

**Fallback Response** — a pre-prepared answer returned when the primary data source is unavailable. It's not as good as the live response (the trivia question won't be random), but it's infinitely better than an error message or a 10-second timeout.

**SSM Parameter Store** — a free AWS service that stores configuration values (strings, numbers, secrets). Your Lambda reads a parameter at runtime to decide whether to use the live API or the fallback. This lets you flip the switch *without redeploying code* — change the parameter and the next invocation uses the new behaviour.

**Circuit Breaker Pattern** — an architecture pattern where you "open the circuit" (stop calling a failing dependency) when it's unhealthy, and "close the circuit" (resume calling it) when it recovers. The SSM parameter acts as a manual circuit breaker in this lab.

---

## ⚠️ Reminder: How to Read Commands

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account number | `123456789012` |

---

## Lab Steps

### Step 1: Set Up

**Step 1a:** Set your AWS profile and navigate to your project folder.

**Windows (PowerShell):**
```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
cd ~\Desktop\workshop-lab-10a
pwd
```

**macOS / Linux:**
```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
cd ~/Desktop/workshop-lab-10a
pwd
```

> **⚠️ This setting is lost if you close your terminal.**

---

### Step 2: Create the Fallback Mode Parameter in SSM

This parameter controls whether the bot uses the live API or a fallback. You'll flip it without redeploying code.

📋 Copy and paste:

```
aws ssm put-parameter --name "/workshop/chatbot/mode" --type String --value "live" --region us-east-1
```

**✅ You should see** JSON with `"Version": 1`.

> **What does this do?** Creates a parameter at the path `/workshop/chatbot/mode` with the value `"live"`. Your Lambda will read this parameter on every invocation. When it's `"live"`, the bot calls the trivia API normally. When it's `"fallback"`, the bot returns a pre-stored response without calling the API at all.

---

### Step 3: Give the Lambda Role Permission to Read SSM

📋 Copy and paste:

```
aws iam put-role-policy --role-name workshop-lab10-lambda-role --policy-name ssm-read --policy-document "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Action\":[\"ssm:GetParameter\"],\"Resource\":\"arn:aws:ssm:us-east-1:*:parameter/workshop/chatbot/*\"}]}"
```

**✅ No output means success.**

> **💡 On Mac/Linux**, the inline JSON with escaped quotes works. **On Windows**, if this fails, create a file called `ssm-policy.json` with:
> ```json
> {
>     "Version": "2012-10-17",
>     "Statement": [
>         {
>             "Effect": "Allow",
>             "Action": ["ssm:GetParameter"],
>             "Resource": "arn:aws:ssm:us-east-1:*:parameter/workshop/chatbot/*"
>         }
>     ]
> }
> ```
> Then run: `aws iam put-role-policy --role-name workshop-lab10-lambda-role --policy-name ssm-read --policy-document file://ssm-policy.json`

---

### Step 4: Create the Resilient Handler

This version checks the SSM parameter on every request. If mode is `"fallback"`, it skips the API entirely and returns a cached response.

**Step 4a:** Open your text editor → new file. 📋 Copy and paste:

```python
import json
import logging
import time
import urllib.request
import urllib.error
import boto3

logger = logging.getLogger()
logger.setLevel(logging.INFO)

TRIVIA_API_URL = "https://opentdb.com/api.php?amount=1&type=multiple"
SSM_PARAMETER = "/workshop/chatbot/mode"

ssm_client = boto3.client("ssm", region_name="us-east-1")

FALLBACK_RESPONSE = (
    "Category: General Knowledge\n"
    "Question: What is the largest planet in our solar system?\n"
    "Answer: Jupiter"
)

def get_mode():
    """Check SSM Parameter Store for current mode (live or fallback)."""
    try:
        result = ssm_client.get_parameter(Name=SSM_PARAMETER)
        return result["Parameter"]["Value"]
    except Exception:
        return "live"  # Default to live if parameter can't be read

def lambda_handler(event, context):
    """Resilient chatbot with fallback mode."""
    start_time = time.time()
    request_id = context.aws_request_id

    body = {}
    if event.get("body"):
        try:
            body = json.loads(event["body"])
        except (json.JSONDecodeError, TypeError):
            body = {}

    user_message = body.get("message", "Give me a trivia question")
    mode = get_mode()

    logger.info(json.dumps({
        "level": "INFO",
        "event": "request_received",
        "request_id": request_id,
        "user_message": user_message,
        "mode": mode
    }))

    if mode == "fallback":
        # Skip the API entirely — return cached response
        api_latency_ms = 0
        logger.info(json.dumps({
            "level": "INFO",
            "event": "fallback_used",
            "request_id": request_id,
            "reason": "Mode set to fallback via SSM parameter"
        }))
        bot_response = FALLBACK_RESPONSE
    else:
        # Call external trivia API (live mode)
        api_start = time.time()
        try:
            req = urllib.request.Request(TRIVIA_API_URL)
            with urllib.request.urlopen(req, timeout=5) as response:
                api_data = json.loads(response.read().decode())
                api_status = response.status
            api_latency_ms = round((time.time() - api_start) * 1000, 2)

            logger.info(json.dumps({
                "level": "INFO",
                "event": "api_call_success",
                "request_id": request_id,
                "api_url": TRIVIA_API_URL,
                "api_status": api_status,
                "api_latency_ms": api_latency_ms
            }))

            question_data = api_data["results"][0]
            question = question_data["question"]
            correct = question_data["correct_answer"]
            category = question_data["category"]

            bot_response = (
                f"Category: {category}\n"
                f"Question: {question}\n"
                f"Answer: {correct}"
            )

        except (urllib.error.URLError, urllib.error.HTTPError, Exception) as e:
            api_latency_ms = round((time.time() - api_start) * 1000, 2)
            logger.error(json.dumps({
                "level": "ERROR",
                "event": "api_call_failed",
                "request_id": request_id,
                "api_url": TRIVIA_API_URL,
                "api_latency_ms": api_latency_ms,
                "error": str(e)
            }))
            bot_response = "Sorry, I couldn't fetch a trivia question right now. Try again later!"

    total_duration_ms = round((time.time() - start_time) * 1000, 2)
    logger.info(json.dumps({
        "level": "INFO",
        "event": "request_completed",
        "request_id": request_id,
        "total_duration_ms": total_duration_ms,
        "api_latency_ms": api_latency_ms,
        "mode": mode
    }))

    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({
            "response": bot_response,
            "metadata": {
                "api_latency_ms": api_latency_ms,
                "total_duration_ms": total_duration_ms,
                "mode": mode
            }
        })
    }
```

**Step 4b:** Save as `handler.py` in your `workshop-lab-10a` folder (overwriting the previous version).

> **What's new in this version?**
> - It imports `boto3` and reads the SSM parameter on every request
> - If mode is `"fallback"`, it returns a pre-stored Jupiter trivia question instantly (0ms API latency)
> - If mode is `"live"`, it calls the real API as before
> - It logs which mode was used (`"mode": "live"` or `"mode": "fallback"`) and a `fallback_used` event
> - The fallback path is **instant** — no network call, no delay, no risk of failure

---

### Step 5: Deploy the Resilient Version

> **⚠️ Verify you're in the `workshop-lab-10a` folder** (`pwd`).

**Windows (PowerShell):**
```powershell
Compress-Archive -Path handler.py -DestinationPath function.zip -Force
aws lambda update-function-code --function-name workshop-chatbot-lab10 --zip-file fileb://function.zip --region us-east-1 --query "LastUpdateStatus" --output text
```

**macOS / Linux:**
```bash
zip function.zip handler.py
aws lambda update-function-code --function-name workshop-chatbot-lab10 --zip-file fileb://function.zip --region us-east-1 --query "LastUpdateStatus" --output text
```

**✅ You should see:** `InProgress` or `Successful`

---

### Step 6: Test in Live Mode

The parameter is currently set to `"live"`, so the bot should work normally.

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-chatbot-lab10 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json; Get-Content response.json
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-chatbot-lab10 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json && cat response.json
```

**✅ You should see** a random trivia question with `"mode": "live"` in the metadata. API latency should be ~300–600ms (normal).

---

### Step 7: Switch to Fallback Mode

Now simulate the scenario: the external API is degraded, and you (or automation) flips the switch to fallback mode.

📋 Copy and paste:

```
aws ssm put-parameter --name "/workshop/chatbot/mode" --type String --value "fallback" --overwrite --region us-east-1
```

**✅ You should see** `"Version": 2`.

---

### Step 8: Test in Fallback Mode

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-chatbot-lab10 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json; Get-Content response.json
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-chatbot-lab10 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json && cat response.json
```

**✅ You should see:**
- The response contains the **Jupiter question** (the fallback)
- `"mode": "fallback"` in metadata
- `"api_latency_ms": 0` — no API call was made!
- `"total_duration_ms"` should be very low (under 100ms)

> **🎯 The bot is instant!** It didn't call the external API at all. Users get a response in milliseconds instead of waiting 3+ seconds for a degraded API. The question is always the same (not random), but that's far better than an error or a 10-second timeout.

**Step 8a:** Invoke a few more times to generate dashboard data:

**Windows (PowerShell):**
```powershell
1..5 | ForEach-Object { aws lambda invoke --function-name workshop-chatbot-lab10 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json | Out-Null; Write-Host "Fallback invocation $_ done" }
```

**macOS / Linux:**
```bash
for i in 1 2 3 4 5; do aws lambda invoke --function-name workshop-chatbot-lab10 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json > /dev/null 2>&1; echo "Fallback invocation $i done"; done
```

---

### Step 9: Switch Back to Live Mode

📋 Copy and paste:

```
aws ssm put-parameter --name "/workshop/chatbot/mode" --type String --value "live" --overwrite --region us-east-1
```

**✅ You should see** `"Version": 3`.

Invoke once more to confirm it's back to live random questions:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-chatbot-lab10 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json; Get-Content response.json
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-chatbot-lab10 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json && cat response.json
```

**✅ You should see** a random trivia question again (not Jupiter) with `"mode": "live"`.

---

### Step 10: ✅ Console Checkpoint — See All Three States in the Dashboard

**Step 10a:** Open the CloudWatch Console → **Dashboards** → **workshop-chatbot-dashboard**.

**✅ You should see the story in the graphs:**
- **External API Latency:** High when in live mode (300–600ms), zero during fallback mode
- **Chatbot Invocations:** Steady throughout — the bot never stopped serving requests
- **Lambda Errors:** Zero — the bot never crashed, even when the API was "degraded"

> **💡 The key observation:** During fallback mode, the bot kept running, users got instant responses, and NO errors occurred. Compare this to Lab 10B where the slow API made every user wait 3+ seconds. Fallback is the difference between "users have a bad experience" and "users don't even notice the problem."

**Step 10b:** Check the logs to see the `fallback_used` events. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
$endTime = [int64](([datetime]::UtcNow) - ([datetime]'1970-01-01')).TotalSeconds
$startTime = $endTime - 1800
$queryId = aws logs start-query --log-group-name "/aws/lambda/workshop-chatbot-lab10" --start-time $startTime --end-time $endTime --query-string "fields @timestamp, @message | filter @message like /fallback_used/ | sort @timestamp desc | limit 5" --region us-east-1 --query "queryId" --output text
Start-Sleep 5
aws logs get-query-results --query-id $queryId --region us-east-1 --query "results[*][*].value" --output text
```

**macOS / Linux:**
```bash
QUERY_ID=$(aws logs start-query --log-group-name "/aws/lambda/workshop-chatbot-lab10" --start-time $(date -u -d '30 minutes ago' +%s) --end-time $(date -u +%s) --query-string "fields @timestamp, @message | filter @message like /fallback_used/ | sort @timestamp desc | limit 5" --region us-east-1 --query "queryId" --output text)
sleep 5
aws logs get-query-results --query-id "$QUERY_ID" --region us-east-1 --query "results[*][*].value" --output text
```

**✅ You should see** entries showing `"event": "fallback_used"` with timestamps from when you had the parameter set to "fallback."

---

## What You Just Did

| What You Built | Why It Matters |
|---------------|----------------|
| SSM Parameter as a mode switch | Change application behaviour without redeploying code |
| Fallback response path | Bot stays fast and responsive even when the API is down |
| Graceful degradation | Users get *something* (cached response) rather than an error or timeout |
| Observable mode in logs | You can see exactly when the bot was in fallback mode and why |
| Zero-downtime mode switching | Flip a parameter → immediate effect → no deploy needed |

**Key takeaways:**
- **Detection is not enough.** Lab 10B taught you to detect slow dependencies. This lab taught you to *tolerate* them.
- **Graceful degradation > hard failure.** A cached response is always better than an error.
- **SSM Parameter Store** lets you change behaviour at runtime without redeploying — essential for incident response ("flip to fallback NOW while we investigate")
- **The three observable states:** live (normal), fallback (degraded but functional), failed (error). Good dashboards show all three.
- This pattern is how real production systems handle unreliable dependencies — Netflix, Spotify, and every major service uses circuit breakers and fallbacks.

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

The SAA exam tests:
- SSM Parameter Store as a runtime configuration service
- Graceful degradation and fault-tolerant architecture
- Resilience pillar of the Well-Architected Framework
- Loose coupling between services (not failing when a dependency fails)
- Circuit breaker and fallback patterns

**Sample question type:** "An application depends on a third-party API that occasionally becomes unavailable. How should the architect design for resilience?"  
**Answer:** Implement a fallback mechanism that returns cached or default responses when the primary API is unavailable. Use a configuration parameter (like SSM Parameter Store) to switch between live and fallback modes, allowing runtime control without redeployment.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `AccessDeniedException` reading SSM parameter | Lambda role doesn't have `ssm:GetParameter` permission | Re-run Step 3 to attach the SSM policy to the role |
| Bot still returns random questions in fallback mode | Old code is deployed (doesn't read SSM) | Verify `handler.py` has the `get_mode()` function and `boto3` import, re-zip, re-deploy |
| `No module named 'boto3'` | Lambda runtime issue | `boto3` is pre-installed in Python 3.12 Lambda runtime — this shouldn't happen. Verify `--runtime python3.12` |
| Parameter not found | Wrong parameter name or region | Verify with `aws ssm get-parameter --name "/workshop/chatbot/mode" --region us-east-1` |
| Fallback response shows `api_latency_ms` not 0 | The code took the live path despite fallback mode | Check the SSM parameter value: `aws ssm get-parameter --name "/workshop/chatbot/mode" --region us-east-1` — it should show `"Value": "fallback"` |
| `put-role-policy` fails on Windows with JSON | PowerShell mangling inline JSON | Use the `file://ssm-policy.json` approach described in Step 3's Windows note |

---

## Cleanup

**Clean up everything from Session 10 (all three labs):**

**Step 1:** Delete the SSM parameter:
```
aws ssm delete-parameter --name "/workshop/chatbot/mode" --region us-east-1
```

**Step 2:** Delete the alarm and dashboard:
```
aws cloudwatch delete-alarms --alarm-names chatbot-api-slow --region us-east-1
aws cloudwatch delete-dashboards --dashboard-names workshop-chatbot-dashboard --region us-east-1
```

**Step 3:** Delete the metric filters:
```
aws logs delete-metric-filter --log-group-name "/aws/lambda/workshop-chatbot-lab10" --filter-name api-latency --region us-east-1
aws logs delete-metric-filter --log-group-name "/aws/lambda/workshop-chatbot-lab10" --filter-name api-errors --region us-east-1
```

**Step 4:** Delete the Lambda function and logs:
```
aws lambda delete-function --function-name workshop-chatbot-lab10 --region us-east-1
aws logs delete-log-group --log-group-name /aws/lambda/workshop-chatbot-lab10 --region us-east-1
```

**Step 5:** Remove the Lambda role:
```
aws iam delete-role-policy --role-name workshop-lab10-lambda-role --policy-name ssm-read
aws iam detach-role-policy --role-name workshop-lab10-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name workshop-lab10-lambda-role
```

**Step 6:** Delete the project folder:

**Windows (PowerShell):**
```powershell
cd ~\Desktop; Remove-Item -Recurse -Force workshop-lab-10a
```

**macOS / Linux:**
```bash
cd ~/Desktop && rm -rf workshop-lab-10a
```

---

## Help

If you get stuck, post in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** you received
4. Your **operating system** (Windows / Mac / Linux)
