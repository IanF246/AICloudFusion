# Lab 10B: Monitor Your Bot's Dependencies — Custom Metrics & Dashboards

**Session:** 10 — Chatbot Observability  
**Track:** Solutions Architecture  
**Difficulty:** Intermediate  
**Estimated Time:** 40–50 minutes  
**Target Cert:** AWS Solutions Architect – Associate (SAA)

---

## Overview

Your chatbot works — but how fast is it? And what happens when the trivia API it depends on gets slow or goes down? In Lab 10A you logged structured JSON. In this lab you'll **turn those logs into metrics** and build a dashboard that answers: "How healthy is my bot's external dependency?"

By the end of this lab you will have:

1. **Metric filters** that extract API latency and error counts from your structured logs
2. **Custom CloudWatch metrics** tracking external API performance
3. **A CloudWatch Dashboard** showing your bot's health at a glance
4. **An alarm** that fires when the external API is too slow
5. **Proof it works** — you'll deploy a degraded version, watch latency spike, and see the alarm fire

**The feedback loop:** Healthy bot → deploy a slow dependency → dashboard shows the spike → alarm fires → fix it → dashboard recovers. You'll see the difference between "my code works" and "my dependency is healthy."

---

## Prerequisites

- ✅ Completed **Lab 10A** — your `workshop-chatbot-lab10` Lambda function must be deployed and working
- ✅ AWS CLI authenticated (`aws sts get-caller-identity`)

> **📌 If you cleaned up Lab 10A:** You need the function and role back. Follow Lab 10A Steps 2–4 to recreate `workshop-lab10-lambda-role` and deploy `workshop-chatbot-lab10`. Then invoke it once to verify it works before continuing.

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| AWS Lambda | Function execution | Always Free (1M requests/month) |
| CloudWatch Custom Metrics | Metrics you define from log data | Always Free (10 custom metrics) |
| CloudWatch Dashboard | Visual operational view | Always Free (3 dashboards) |
| CloudWatch Alarms | Threshold-based alerting | Always Free (10 alarms) |

**Estimated cost for this lab: $0.00**

---

## Concepts

**Metric Filter** — a rule that watches CloudWatch Logs and extracts numbers into CloudWatch Metrics. You define a pattern ("find log entries where `event` equals `api_call_success`") and a value to extract (`$.api_latency_ms`). Every time a matching log entry arrives, the number gets published as a metric data point. This turns text logs into graphable, alertable numbers.

**Custom Metric** — a metric you define (as opposed to built-in metrics like Lambda Invocations). Custom metrics live in a **namespace** you choose (we'll use `WorkshopChatbot`). They work exactly like built-in metrics — you can graph them, alarm on them, and put them in dashboards.

**Namespace** — a container for related metrics. AWS uses namespaces like `AWS/Lambda` and `AWS/S3`. You create your own (like `WorkshopChatbot`) to keep your custom metrics organized and separate from AWS's.

**Dependency Monitoring** — watching the health of services your application *depends on*, not just your own code. Your chatbot's code might be perfect, but if the trivia API is slow, your users still have a bad experience. Monitoring dependencies separately lets you answer "is the problem in MY code or in something I call?"

**Dashboard** — a single-page view combining multiple metrics. Instead of checking metrics one by one, a dashboard shows everything at a glance: invocations, errors, API latency, and failure count — all on one screen.

---

## ⚠️ Reminder: How to Read Commands

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account number | `123456789012` |

---

## Lab Steps

### Step 1: Set Up and Verify Your Chatbot Is Running

**Step 1a:** Set your AWS profile.

**Windows (PowerShell):**
```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**macOS / Linux:**
```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

> **⚠️ This setting is lost if you close your terminal.** If any command later gives an auth error, come back here.

**Step 1b:** Verify the chatbot function exists and works. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
cd ~\Desktop\workshop-lab-10a
aws lambda invoke --function-name workshop-chatbot-lab10 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json; Get-Content response.json
```

**macOS / Linux:**
```bash
cd ~/Desktop/workshop-lab-10a
aws lambda invoke --function-name workshop-chatbot-lab10 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json && cat response.json
```

**✅ You should see** `StatusCode: 200` and a trivia question in the response. If you get a "function not found" error, go back to Lab 10A Steps 2–4 to redeploy it.

> If you get an expired token error, run `aws sso login --profile <YOUR_PROFILE_NAME>`, approve in browser, and try again.

---

### Step 2: Create a Metric Filter for API Latency

This filter watches your chatbot's logs and extracts the `api_latency_ms` value every time the external API call succeeds. It publishes that number as a custom CloudWatch metric.

**Step 2a:** Create the metric transformation file. Open your text editor → new file. 📋 Copy and paste:

```json
[
    {
        "metricName": "ExternalAPILatency",
        "metricNamespace": "WorkshopChatbot",
        "metricValue": "$.api_latency_ms",
        "defaultValue": 0
    }
]
```

**Step 2b:** Save it as `latency-transform.json` in your `workshop-lab-10a` folder.

> **What does this do?** It tells CloudWatch: "When you find a matching log entry, extract the value of the `api_latency_ms` field and publish it as a metric called `ExternalAPILatency` in the `WorkshopChatbot` namespace."

**Step 2c:** Create the metric filter. 📋 Copy and paste (from your `workshop-lab-10a` folder):

**Windows (PowerShell):**
```powershell
aws logs put-metric-filter --log-group-name "/aws/lambda/workshop-chatbot-lab10" --filter-name api-latency --filter-pattern "{ $.event = ""api_call_success"" }" --metric-transformations file://latency-transform.json --region us-east-1
```

**macOS / Linux:**
```bash
aws logs put-metric-filter --log-group-name "/aws/lambda/workshop-chatbot-lab10" --filter-name api-latency --filter-pattern '{ $.event = "api_call_success" }' --metric-transformations file://latency-transform.json --region us-east-1
```

**✅ No output means success.**

> **⚠️ Windows note:** The filter pattern uses `""` (double-double-quotes) around `api_call_success` because PowerShell interprets single quotes differently. On Mac/Linux, single quotes around the whole pattern work fine.

> **What does the filter pattern do?** `{ $.event = "api_call_success" }` means "match any log entry that is JSON AND has a field called `event` with the value `api_call_success`." Only those entries will have their `api_latency_ms` extracted.

---

### Step 3: Create a Metric Filter for API Errors

Now create a second filter that counts every time the external API fails.

**Step 3a:** Create the error transformation file. Open your text editor → new file. 📋 Copy and paste:

```json
[
    {
        "metricName": "ExternalAPIErrors",
        "metricNamespace": "WorkshopChatbot",
        "metricValue": "1",
        "defaultValue": 0
    }
]
```

**Step 3b:** Save it as `error-transform.json` in your `workshop-lab-10a` folder.

> **What's different?** The `metricValue` is `"1"` (a fixed number), not a field extraction. Every time a log entry matches the error pattern, it counts as 1. This gives you an error *count* rather than an extracted number.

**Step 3c:** Create the error metric filter. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws logs put-metric-filter --log-group-name "/aws/lambda/workshop-chatbot-lab10" --filter-name api-errors --filter-pattern "{ $.event = ""api_call_failed"" }" --metric-transformations file://error-transform.json --region us-east-1
```

**macOS / Linux:**
```bash
aws logs put-metric-filter --log-group-name "/aws/lambda/workshop-chatbot-lab10" --filter-name api-errors --filter-pattern '{ $.event = "api_call_failed" }' --metric-transformations file://error-transform.json --region us-east-1
```

**✅ No output means success.**

---

### Step 4: Generate Metric Data

Metric filters only process NEW log entries (they don't backfill). Invoke the chatbot several times to push data through the filters.

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
1..5 | ForEach-Object { aws lambda invoke --function-name workshop-chatbot-lab10 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json | Out-Null; Write-Host "Invocation $_ complete" }
```

**macOS / Linux:**
```bash
for i in 1 2 3 4 5; do aws lambda invoke --function-name workshop-chatbot-lab10 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json > /dev/null 2>&1; echo "Invocation $i complete"; done
```

**✅ You should see** "Invocation 1 complete" through "Invocation 5 complete."

> **💡 Wait 1–2 minutes** for the metric filters to process the logs and publish the data points. Custom metrics have a short delay before they appear.

---

### Step 5: Verify Custom Metrics Are Publishing

**Step 5a:** Check the API latency metric. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
$endTime = (Get-Date).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
$startTime = (Get-Date).AddMinutes(-10).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
aws cloudwatch get-metric-statistics --namespace "WorkshopChatbot" --metric-name ExternalAPILatency --start-time $startTime --end-time $endTime --period 60 --statistics Average Maximum --region us-east-1
```

**macOS / Linux:**
```bash
aws cloudwatch get-metric-statistics --namespace "WorkshopChatbot" --metric-name ExternalAPILatency --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%SZ) --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) --period 60 --statistics Average Maximum --region us-east-1
```

**✅ You should see** datapoints with `Average` and `Maximum` values (typically 300–600ms for the trivia API).

> **💡 If Datapoints is empty:** The metrics need 1–2 minutes to appear. Wait and retry. If still empty after 3 minutes, verify your metric filter exists: `aws logs describe-metric-filters --log-group-name "/aws/lambda/workshop-chatbot-lab10" --region us-east-1`

**Step 5b: What do these numbers mean?**
- `Average` — the typical API response time across all invocations in that minute
- `Maximum` — the slowest single call (cold starts are often higher)

> **🎯 This is your healthy baseline.** Remember these numbers. In Step 9, you'll watch them spike when the API gets slow.

---

### Step 6: Build a CloudWatch Dashboard

Now build a single-pane view of your chatbot's health.

**Step 6a:** Create the dashboard definition file. Open your text editor → new file. 📋 Copy and paste:

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
                "title": "Chatbot Invocations",
                "metrics": [
                    ["AWS/Lambda", "Invocations", "FunctionName", "workshop-chatbot-lab10"]
                ],
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
                "title": "Lambda Errors",
                "metrics": [
                    ["AWS/Lambda", "Errors", "FunctionName", "workshop-chatbot-lab10"]
                ],
                "period": 60,
                "stat": "Sum",
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
                "title": "External API Latency (ms)",
                "metrics": [
                    ["WorkshopChatbot", "ExternalAPILatency"]
                ],
                "period": 60,
                "stat": "Average",
                "region": "us-east-1",
                "annotations": {
                    "horizontal": [
                        {
                            "label": "Alarm threshold (2000ms)",
                            "value": 2000,
                            "color": "#d13212"
                        }
                    ]
                }
            }
        },
        {
            "type": "metric",
            "x": 12,
            "y": 6,
            "width": 12,
            "height": 6,
            "properties": {
                "title": "External API Errors",
                "metrics": [
                    ["WorkshopChatbot", "ExternalAPIErrors"]
                ],
                "period": 60,
                "stat": "Sum",
                "region": "us-east-1"
            }
        }
    ]
}
```

**Step 6b:** Save it as `dashboard.json` in your `workshop-lab-10a` folder.

**Step 6c:** Deploy the dashboard. 📋 Copy and paste (from the `workshop-lab-10a` folder, where `dashboard.json` is):

```
aws cloudwatch put-dashboard --dashboard-name workshop-chatbot-dashboard --dashboard-body file://dashboard.json --region us-east-1
```

**✅ You should see** `"DashboardValidationMessages": []` — empty means no errors.

---

### Step 7: ✅ Console Checkpoint — View Your Dashboard

**Step 7a:** Open the AWS Console → **CloudWatch** → **Dashboards** (left menu).

**Step 7b:** Click **workshop-chatbot-dashboard**.

**✅ You should see** four panels:
- **Chatbot Invocations** — shows your 5+ invocations
- **Lambda Errors** — should be 0 (your code isn't crashing)
- **External API Latency (ms)** — shows the typical 300–600ms response time with a red threshold line at 2000ms
- **External API Errors** — should be 1 or 2 (the trivia API is responding normally and has failsafe for too many requests in short time)

> **💡 This is what "healthy" looks like.** All four panels are calm. Latency is well below the red line. A few errors. Remember this view — you're about to break things.

---

### Step 8: Create a Latency Alarm

Create an alarm that fires when the external API is too slow (average latency > 2000ms).

📋 Copy and paste:

```
aws cloudwatch put-metric-alarm --alarm-name chatbot-api-slow --metric-name ExternalAPILatency --namespace "WorkshopChatbot" --statistic Maximum --period 60 --threshold 2000 --comparison-operator GreaterThanThreshold --evaluation-periods 1 --alarm-description "External API latency exceeds 2 seconds" --region us-east-1
```

**✅ No output means success.**

> **What does this alarm do?** If the maximum `ExternalAPILatency` exceeds 2000ms (2 seconds) in any 1-minute window, the alarm transitions to ALARM state. We use `Maximum` (not Average) because even a single slow request indicates a dependency problem — averaging would hide it if most requests are fast.

**Step 8a:** Verify the alarm was created. 📋 Copy and paste:

```
aws cloudwatch describe-alarms --alarm-names chatbot-api-slow --region us-east-1 --query "MetricAlarms[0].[AlarmName,StateValue]" --output text
```

**✅ You should see:** `chatbot-api-slow    INSUFFICIENT_DATA` (hasn't evaluated yet) or `chatbot-api-slow    OK` (latency is below threshold).

---

### Step 9: 🚨 Simulate a Slow Dependency

Now you'll deploy a version of the chatbot that simulates the external API being slow — a 3-second delay before every API call. This is what it looks like when a dependency degrades in production.

**Step 9a:** Create the slow version. Open your text editor → new file. 📋 Copy and paste:

```python
import json
import logging
import time
import urllib.request
import urllib.error

logger = logging.getLogger()
logger.setLevel(logging.INFO)

TRIVIA_API_URL = "https://opentdb.com/api.php?amount=1&type=multiple"

def lambda_handler(event, context):
    """Chatbot - SLOW VERSION simulating degraded external API."""
    start_time = time.time()
    request_id = context.aws_request_id

    body = {}
    if event.get("body"):
        try:
            body = json.loads(event["body"])
        except (json.JSONDecodeError, TypeError):
            body = {}

    user_message = body.get("message", "Give me a trivia question")

    logger.info(json.dumps({
        "level": "INFO",
        "event": "request_received",
        "request_id": request_id,
        "user_message": user_message
    }))

    # Call external trivia API
    api_start = time.time()
    try:
        # SIMULATED DEGRADATION: 3-second delay
        time.sleep(3)

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
        "api_latency_ms": api_latency_ms
    }))

    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({
            "response": bot_response,
            "metadata": {
                "api_latency_ms": api_latency_ms,
                "total_duration_ms": total_duration_ms
            }
        })
    }
```

**Step 9b:** Save it as `handler.py` in your `workshop-lab-10a` folder (**overwriting** the healthy version).

> **What's different?** Line 36: `time.sleep(3)` adds a 3-second delay before every API call. The logged `api_latency_ms` will now be ~3300ms+ instead of ~400ms. The chatbot still *works* — it's just very slow. This simulates a real-world scenario where a third-party API degrades.

**Step 9c:** Re-package and deploy the slow version.

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

### Step 10: Invoke the Slow Version and Watch Latency Spike

**Step 10a:** Invoke the slow chatbot several times (each call will take ~3–4 seconds). 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
1..3 | ForEach-Object { aws lambda invoke --function-name workshop-chatbot-lab10 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json; Write-Host "Slow invocation $_ complete"; Get-Content response.json | Write-Host }
```

**macOS / Linux:**
```bash
for i in 1 2 3; do aws lambda invoke --function-name workshop-chatbot-lab10 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json; echo "Slow invocation $i complete"; cat response.json; echo ""; done
```

**✅ You should see** each invocation taking noticeably longer (3–4 seconds vs the previous <1 second). In the response metadata, `api_latency_ms` should be ~3300+ instead of the previous ~400.

> **💡 Notice the difference?** The chatbot still *works* — it returns a trivia question. But it's 8x slower. In production, users would notice this as the app feeling sluggish, even though technically "no errors are happening."

---

### Step 11: Watch the Alarm Fire

**Step 11a:** Wait 1–2 minutes for CloudWatch to evaluate, then check the alarm. 📋 Copy and paste:

```
aws cloudwatch describe-alarms --alarm-names chatbot-api-slow --region us-east-1 --query "MetricAlarms[0].[AlarmName,StateValue,StateReason]" --output text
```

**✅ You should see:** `chatbot-api-slow    ALARM    Threshold Crossed: 1 datapoint [3XXX.X] was greater than the threshold (2000.0).`

> **🎯 The alarm caught the degraded dependency.** Your code didn't crash. Lambda didn't error. But the external API response time crossed your threshold, and the alarm told you. Without this monitoring, users would just experience slowness and you'd have no idea why.

✅ Console Checkpoint**
Step 11b: **Add an Average/Maximum toggle** — By default the latency panel plots the **Average**, which smooths out spikes. Add a dropdown so you can switch to **Maximum** (what the alarm evaluates) without editing the widget:

On the dashboard, click **Actions → Variables → Create a variable**
Choose **Pattern variable (advanced)**
In the pattern box, type `Average` and select the match
Set **Input type** to **Select menu (dropdown)** and add two values: `Average` and `Maximum`
Under secondary settings — **Name**: `stat`, **Label**: `Statistic`, **Default**: `Average`
Click **Create variable**, then **Save dashboard**
Flip the new **Statistic** dropdown to `Maximum`.

1. The **External API Latency** panel should now show a dramatic spike above the red 2000ms threshold line
2. The **Chatbot Invocations** panel shows the requests still succeeded/failed respectively
3. The **Lambda Errors** panel is still 0 — your code is fine, it's the dependency that's slow

> **💡 This is the key insight of dependency monitoring:** "no errors" doesn't mean "healthy." Latency degradation is invisible without specific measurement. Your dashboard now shows the full picture.

---

### Step 12: Restore the Healthy Version

**Step 12a:** Rewrite `handler.py` with the original healthy code from Lab 10A (remove the `time.sleep(3)` line). Open the file and **delete line 36** (`time.sleep(3)`), or replace the whole file. 📋 The healthy version should NOT have the `time.sleep(3)` line.

**Step 12b:** Re-deploy. 📋 Copy and paste:

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

**Step 12c:** Invoke to verify it's fast again. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-chatbot-lab10 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json; Get-Content response.json
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-chatbot-lab10 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json && cat response.json
```

**✅ You should see** `api_latency_ms` back to ~300–600ms (no more 3000+). The chatbot is fast again.

> After 1–2 minutes, the alarm should return to `OK` and the dashboard latency panel should drop back below the red line.

---

## What You Just Did

| What You Built | Why It Matters |
|---------------|----------------|
| Metric filter extracting API latency from logs | Turns text logs into graphable, alertable numbers — automatically |
| Metric filter counting API errors | Tracks dependency failures without custom code |
| Custom CloudWatch metrics in your own namespace | Your application's health data, separate from AWS's built-in metrics |
| A 4-panel operational dashboard | Single-pane view of chatbot health: traffic, errors, dependency speed, dependency failures |
| Latency alarm | Proactive notification when a dependency degrades — even when your code doesn't crash |
| Proved the alarm catches slow dependencies | "No errors" ≠ "healthy." Latency degradation needs its own monitoring |

**Key takeaways:**
- **Metric filters** let you extract custom metrics from structured logs without changing your code
- **Dependencies are your #1 risk.** Most outages aren't your code crashing — they're a third-party API going slow or down
- **"Working" is not the same as "healthy."** The slow chatbot still returned answers, but users would hate the 3-second wait
- **Dashboards tell the story** — all four panels together show traffic + errors + speed + dependency health
- In Lab 10C, you'll make the chatbot **self-healing** — automatically switching to a fallback when the dependency degrades

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

The SAA exam tests:
- CloudWatch metric filters and custom metrics
- CloudWatch Dashboards for operational visibility
- Monitoring external dependencies as an operational best practice
- Alarm configuration (namespace, metric, threshold, period, evaluation periods)
- Operational Excellence pillar: measure and monitor everything

**Sample question type:** "An application calls a third-party API. The team wants to alert when that API's response time exceeds 2 seconds. The application logs response times in JSON format to CloudWatch Logs. What should they do?"  
**Answer:** Create a CloudWatch Logs metric filter with a pattern matching the API call log entry, extracting the latency field as a custom metric. Create a CloudWatch Alarm on that custom metric with a threshold of 2000ms.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| Custom metrics show empty Datapoints | Metric filters only process NEW logs (not historical) | Invoke the function a few more times and wait 2 minutes |
| Filter pattern error on Windows | PowerShell mangled the quotes in the pattern | Use `""` (double-double-quotes) inside the pattern string, not `'` |
| `ResourceNotFoundException` on log group | The function hasn't been invoked yet | Invoke the function once first to create the log group |
| Dashboard shows "No data available" | Metrics haven't published yet or time range is wrong | Wait 2 minutes; check the dashboard time range is set to "Last 1 hour" |
| Alarm stays in INSUFFICIENT_DATA | Not enough new invocations since the alarm was created | Invoke the function 3+ times and wait 2 minutes |
| Slow version still responds under 1s | The zip packaged the old file | Delete `function.zip`, verify `handler.py` has `time.sleep(3)` on line 36, re-zip, re-deploy |
| `chatbot-api-slow` alarm doesn't fire | Latency metric hasn't crossed threshold yet | The slow invocations need to flow through the metric filter first — wait 2 minutes after the slow invocations |

---

## Cleanup

> **📌 If continuing to Lab 10C:** Keep everything. Lab 10C uses the same function, dashboard, and alarm.

**If stopping here, clean up everything:**

```
aws cloudwatch delete-alarms --alarm-names chatbot-api-slow --region us-east-1
aws cloudwatch delete-dashboards --dashboard-names workshop-chatbot-dashboard --region us-east-1
aws logs delete-metric-filter --log-group-name "/aws/lambda/workshop-chatbot-lab10" --filter-name api-latency --region us-east-1
aws logs delete-metric-filter --log-group-name "/aws/lambda/workshop-chatbot-lab10" --filter-name api-errors --region us-east-1
aws lambda delete-function --function-name workshop-chatbot-lab10 --region us-east-1
aws logs delete-log-group --log-group-name /aws/lambda/workshop-chatbot-lab10 --region us-east-1
aws iam detach-role-policy --role-name workshop-lab10-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name workshop-lab10-lambda-role
```

Delete the project folder:

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
