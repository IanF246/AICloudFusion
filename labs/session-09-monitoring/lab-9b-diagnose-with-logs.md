# Lab 9B: Diagnose the Problem — CloudWatch Logs & Log Insights

**Session:** 9 — Monitoring & Observability  
**Track:** Solutions Architecture  
**Difficulty:** Intermediate  
**Estimated Time:** 40–50 minutes  
**Target Cert:** AWS Solutions Architect – Associate (SAA)

---

## Overview

In Lab 9A, your alarm told you **something is wrong**. But it didn't tell you **what** is wrong or **why**. In the real world, "your Lambda has errors" is just the beginning — you need to investigate, find the root cause, and fix it.

This lab teaches the diagnostic workflow that engineers use every day:

1. **Start from the alarm** — something broke, but what?
2. **Check the metrics** — see *when* the errors started (correlate with a deploy)
3. **Dive into CloudWatch Logs** — find the actual error messages and stack traces
4. **Use Log Insights** to query across thousands of log entries and surface the smoking gun
5. **Fix the code and verify** the metrics return to healthy

**The scenario:** You're an engineer who just got paged. The alarm fired. Your job is to figure out what went wrong, how to fix it, and prove it's fixed. You'll do this by reading the evidence in the logs.

---

## Prerequisites

- ✅ Completed **Lab 9A** — OR have a working Lambda function called `workshop-api-lab9` deployed (follow Lab 9A Steps 1–4 if starting fresh)
- ✅ AWS CLI authenticated (`aws sts get-caller-identity`)

> **📌 If you completed Lab 9A and cleaned up:** You'll need to re-deploy the function. Follow Lab 9A Steps 1–4 to recreate the role and function, then deploy the **broken** version (Lab 9A Step 9a–9c) so you have errors to investigate.

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| AWS Lambda | Serverless function execution | Always Free (1M requests/month) |
| Amazon CloudWatch Logs | Stores function output and errors | Always Free (5GB ingestion, 5GB storage/month) |
| CloudWatch Logs Insights | Query engine for searching logs | Always Free (5GB scanned/month) |

**Estimated cost for this lab: $0.00**

---

## Concepts

**CloudWatch Log Group** — a container for logs from one source. When you create a Lambda function, AWS automatically creates a log group named `/aws/lambda/<function-name>`. Everything your function prints or logs goes here.

**Log Stream** — within a log group, each Lambda execution environment gets its own stream. Think of the log group as a folder and streams as individual files inside it.

**Structured Logging** — writing logs as JSON (`{"level": "ERROR", "message": "..."}`) instead of plain text. This makes them machine-searchable. You can query by field (find all ERROR-level logs, or all logs from a specific request ID).

**CloudWatch Logs Insights** — a query language for searching logs at scale. Instead of scrolling through thousands of lines, you write a query like `filter @message like /ERROR/` and it returns exactly the entries you need, sorted by time. This is how engineers find needles in haystacks.

**Mean Time to Detect (MTTD)** — how long between "something broke" and "we know it broke." Alarms reduce this to seconds.

**Mean Time to Resolve (MTTR)** — how long between "we know it's broken" and "it's fixed." Good logs and queries reduce *this* — because you find the root cause faster.

---

## ⚠️ Reminder: How to Read Commands

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account number | `123456789012` |

---

## Lab Steps

### Step 1: Set Up and Ensure the Broken Function Is Deployed

**Step 1a:** Set your AWS profile.

**Windows (PowerShell):**
```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**macOS / Linux:**
```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**Step 1b:** Verify the function exists and the broken version is deployed. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-api-lab9 --region us-east-1 response.json; Get-Content response.json
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-api-lab9 --region us-east-1 response.json && cat response.json
```

**✅ You should see** `"FunctionError": "Unhandled"` and the response should contain `"errorType": "NameError"`. This confirms the broken version is deployed.

> **If you get an expired token error:** run `aws sso login --profile <YOUR_PROFILE_NAME>`, approve in browser, re-set your profile variable, and try again.

> **If the function doesn't exist** or returns a healthy response: go back to Lab 9A Steps 1–4 to create the role and function, then Steps 9a–9c to deploy the broken version.

**Step 1c:** Navigate to your project folder (or create one if starting fresh):

**Windows (PowerShell):**
```powershell
cd ~\Desktop\workshop-lab-9a
```

**macOS / Linux:**
```bash
cd ~/Desktop/workshop-lab-9a
```

---

### Step 2: 🚨 The Page — Your Alarm Is Firing

Imagine you just received this alert email:

> **Subject:** ALARM: "workshop-lab9-errors" in US East (N. Virginia)  
> **Body:** Threshold Crossed: 1 datapoint [6.0] was greater than or equal to the threshold (1.0).

You know something is wrong. **Your job: figure out what, when, and why.**

---

### Step 3: Investigation Step 1 — Check the Metrics (When Did It Start?)

The first question is always: **when did this start happening?**

**Step 3a:** Pull the error metrics for the last 30 minutes. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
$endTime = (Get-Date).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
$startTime = (Get-Date).AddMinutes(-30).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
aws cloudwatch get-metric-statistics --namespace "AWS/Lambda" --metric-name Errors --dimensions "Name=FunctionName,Value=workshop-api-lab9" --start-time $startTime --end-time $endTime --period 60 --statistics Sum --region us-east-1
```

**macOS / Linux:**
```bash
aws cloudwatch get-metric-statistics --namespace "AWS/Lambda" --metric-name Errors --dimensions "Name=FunctionName,Value=workshop-api-lab9" --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%SZ) --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) --period 60 --statistics Sum --region us-east-1
```

**✅ You should see** a series of datapoints. Look at the pattern:
- Earlier timestamps: `"Sum": 0.0` — no errors (this was the healthy period)
- Later timestamps: `"Sum": 1.0` or higher — errors spiking

> **📝 Note the timestamp** where errors first appeared. This is your first clue: "the problem started at [this time]."

**Step 3b:** Compare with invocation counts for the same period:

**Windows (PowerShell):**
```powershell
aws cloudwatch get-metric-statistics --namespace "AWS/Lambda" --metric-name Invocations --dimensions "Name=FunctionName,Value=workshop-api-lab9" --start-time $startTime --end-time $endTime --period 60 --statistics Sum --region us-east-1
```

**macOS / Linux:**
```bash
aws cloudwatch get-metric-statistics --namespace "AWS/Lambda" --metric-name Invocations --dimensions "Name=FunctionName,Value=workshop-api-lab9" --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%SZ) --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) --period 60 --statistics Sum --region us-east-1
```

**✅ What to observe:** The invocation count tells you how many requests came in. If errors equal invocations, that means **100% of requests are failing** — a total outage, not a partial degradation. That's critical information for your incident response.

> **💡 What you've learned so far:**
> - Errors started at a specific timestamp
> - The error rate is 100% (every request fails)
> - This isn't a gradual degradation — it's a sudden break
>
> **Hypothesis:** Something changed at that timestamp. A deploy? A config change? Let's look at the logs to find out.

---

### Step 4: Investigation Step 2 — Find the Log Group

CloudWatch Logs stores everything your Lambda function prints. Let's find it.

**Step 4a:** List the log group. 📋 Copy and paste:

```
aws logs describe-log-groups --log-group-name-prefix "/aws/lambda/workshop-api-lab9" --region us-east-1 --query "logGroups[].logGroupName" --output text
```

**✅ You should see:** `/aws/lambda/workshop-api-lab9`

> **What does this mean?** AWS automatically created this log group when your Lambda first ran. Every `print()` statement, every `logger.info()`, and every unhandled exception gets written here.

**Step 4b:** List the recent log streams (each stream = one execution environment). 📋 Copy and paste:

```
aws logs describe-log-streams --log-group-name "/aws/lambda/workshop-api-lab9" --region us-east-1 --order-by LastEventTime --descending --limit 3 --query "logStreams[].[logStreamName,lastEventTimestamp]" --output table
```

**✅ You should see** a table with 1–3 stream names and timestamps. The most recent stream contains the errors you're looking for.

---

### Step 5: Investigation Step 3 — Read the Raw Logs (Find the Smoking Gun)

Now you'll read the actual log entries to find the error message and stack trace.

**Step 5a:** Get the most recent log stream name. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
$stream = aws logs describe-log-streams --log-group-name "/aws/lambda/workshop-api-lab9" --region us-east-1 --order-by LastEventTime --descending --limit 1 --query "logStreams[0].logStreamName" --output text
Write-Host "Most recent stream: $stream"
```

**macOS / Linux:**
```bash
STREAM=$(aws logs describe-log-streams --log-group-name "/aws/lambda/workshop-api-lab9" --region us-east-1 --order-by LastEventTime --descending --limit 1 --query "logStreams[0].logStreamName" --output text)
echo "Most recent stream: $STREAM"
```

**Step 5b:** Read the log events from that stream. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws logs get-log-events --log-group-name "/aws/lambda/workshop-api-lab9" --log-stream-name "$stream" --region us-east-1 --limit 20 --query "events[].[timestamp,message]" --output text
```

> **⚠️ PowerShell gotcha:** Lambda log stream names contain `$LATEST` (e.g., `2026/07/17/[$LATEST]abc123`). Because `$` has special meaning in PowerShell, you MUST use the variable (`"$stream"`) — do NOT try to type or paste the stream name directly, or PowerShell will interpret `$LATEST` as an empty variable and the command will fail. If you get an error about a missing stream, this is almost certainly why.

**macOS / Linux:**
```bash
aws logs get-log-events --log-group-name "/aws/lambda/workshop-api-lab9" --log-stream-name "$STREAM" --region us-east-1 --limit 20 --query "events[].[timestamp,message]" --output text
```

**✅ You should see** log entries including lines like:

```
[INFO]  ... {"level": "INFO", "message": "Request received", ...}
[ERROR] NameError: name 'database_url' is not defined
Traceback (most recent call last):
  File "/var/task/handler.py", line 21, in lambda_handler
    connection_string = database_url
```

**🎯 There's your smoking gun:**
- **Error type:** `NameError`
- **Error message:** `name 'database_url' is not defined`
- **Exact location:** `handler.py`, line 21
- **The offending code:** `connection_string = database_url`

> **💡 What this tells you:** The code references a variable called `database_url` that doesn't exist. Someone either removed its definition or introduced this line without defining it. This is the root cause.

---

### Step 6: Investigation Step 4 — Use Log Insights to Query at Scale

In a real application, you'd have thousands of log entries. Scrolling through them manually doesn't scale. CloudWatch Logs Insights lets you search across all entries with a query.

**Step 6a:** Run a query to find all ERROR-level entries in the last hour. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
$endTime = [int64](([datetime]::UtcNow) - ([datetime]'1970-01-01')).TotalSeconds
$startTime = $endTime - 3600
$queryId = aws logs start-query --log-group-name "/aws/lambda/workshop-api-lab9" --start-time $startTime --end-time $endTime --query-string "fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 10" --region us-east-1 --query "queryId" --output text
Write-Host "Query started. ID: $queryId"
```

**macOS / Linux:**
```bash
QUERY_ID=$(aws logs start-query --log-group-name "/aws/lambda/workshop-api-lab9" --start-time $(date -u -d '1 hour ago' +%s) --end-time $(date -u +%s) --query-string "fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 10" --region us-east-1 --query "queryId" --output text)
echo "Query started. ID: $QUERY_ID"
```

> **What does this query do?**
> - `fields @timestamp, @message` — show the time and message for each result
> - `filter @message like /ERROR/` — only return entries containing "ERROR"
> - `sort @timestamp desc` — newest first
> - `limit 10` — show the 10 most recent errors

**Step 6b:** Wait 3–5 seconds, then get the results. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
Start-Sleep 5
aws logs get-query-results --query-id $queryId --region us-east-1
```

**macOS / Linux:**
```bash
sleep 5
aws logs get-query-results --query-id "$QUERY_ID" --region us-east-1
```

**✅ You should see** JSON results with `"status": "Complete"` and a `"results"` array. Each entry shows:
- `@timestamp` — exactly when the error occurred
- `@message` — the full error: `[ERROR] NameError: name 'database_url' is not defined\nTraceback...`

Also note the `"statistics"` section at the bottom:
- `"recordsMatched"` — how many log entries matched your filter
- `"recordsScanned"` — how many total entries were searched

> **💡 Why Log Insights matters:** You just searched through ALL your logs and found every error, sorted by time, in under 5 seconds. In a real application with millions of log entries, this is how you find the needle in the haystack. Without it, you'd be scrolling through logs for hours.

**Step 6c:** Run a second query — count errors per minute to see the spike pattern:

**Windows (PowerShell):**
```powershell
$queryId2 = aws logs start-query --log-group-name "/aws/lambda/workshop-api-lab9" --start-time $startTime --end-time $endTime --query-string "filter @message like /ERROR/ | stats count(*) as error_count by bin(1m)" --region us-east-1 --query "queryId" --output text
Start-Sleep 5
aws logs get-query-results --query-id $queryId2 --region us-east-1
```

**macOS / Linux:**
```bash
QUERY_ID2=$(aws logs start-query --log-group-name "/aws/lambda/workshop-api-lab9" --start-time $(date -u -d '1 hour ago' +%s) --end-time $(date -u +%s) --query-string "filter @message like /ERROR/ | stats count(*) as error_count by bin(1m)" --region us-east-1 --query "queryId" --output text)
sleep 5
aws logs get-query-results --query-id "$QUERY_ID2" --region us-east-1
```

**✅ You should see** results grouped by minute, with an `error_count` field. This shows you:
- Minutes with 0 errors (before the bad deploy)
- Minutes where errors spiked (after the bad deploy)

> **🎯 Investigation summary so far:**
> 1. ✅ **When:** errors started at [specific timestamp]
> 2. ✅ **Severity:** 100% failure rate (every invocation fails)
> 3. ✅ **What:** `NameError: name 'database_url' is not defined`
> 4. ✅ **Where:** `handler.py`, line 21
> 5. ✅ **Why:** Code references a variable that was never defined
>
> You now have everything you need to fix it.

---

### Step 7: Fix the Code and Verify Recovery

Now that you've identified the root cause, fix it.

**Step 7a:** Open your text editor and create the fixed version. 📋 Copy and paste:

```python
import json
import logging
import time

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """Workshop API - FIXED version (database_url reference removed)."""
    start_time = time.time()

    logger.info(json.dumps({
        "level": "INFO",
        "message": "Request received",
        "request_id": context.aws_request_id,
        "path": event.get("path", "/"),
        "method": event.get("httpMethod", "GET")
    }))

    # FIXED: removed the undefined database_url reference
    result = {"status": "healthy", "message": "Hello from the workshop API!"}

    duration = (time.time() - start_time) * 1000
    logger.info(json.dumps({
        "level": "INFO",
        "message": "Request completed successfully",
        "request_id": context.aws_request_id,
        "duration_ms": round(duration, 2)
    }))

    return {
        "statusCode": 200,
        "body": json.dumps(result)
    }
```

**Step 7b:** Save as `handler.py` (overwriting the broken version).

**Step 7c:** Re-deploy the fix.

> **⚠️ Verify you're in the `workshop-lab-9a` folder** (`pwd`) where `handler.py` and `function.zip` are.

**Windows (PowerShell):**
```powershell
Compress-Archive -Path handler.py -DestinationPath function.zip -Force
aws lambda update-function-code --function-name workshop-api-lab9 --zip-file fileb://function.zip --region us-east-1 --query "LastUpdateStatus" --output text
```

**macOS / Linux:**
```bash
zip -f function.zip handler.py
aws lambda update-function-code --function-name workshop-api-lab9 --zip-file fileb://function.zip --region us-east-1 --query "LastUpdateStatus" --output text
```

**✅ You should see:** `InProgress` or `Successful`

**Step 7d:** Verify the fix works. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-api-lab9 --region us-east-1 response.json; Get-Content response.json
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-api-lab9 --region us-east-1 response.json && cat response.json
```

**✅ You should see:**
1. `"StatusCode": 200` — **no** `FunctionError` this time!
2. `{"statusCode": 200, "body": "{\"status\": \"healthy\", \"message\": \"Hello from the workshop API!\"}"}`

**The function is healthy again.**

**Step 7e:** Invoke several times to generate good metrics:

**Windows (PowerShell):**
```powershell
1..5 | ForEach-Object { aws lambda invoke --function-name workshop-api-lab9 --region us-east-1 response.json | Out-Null; Write-Host "Healthy invocation $_ done" }
```

**macOS / Linux:**
```bash
for i in 1 2 3 4 5; do aws lambda invoke --function-name workshop-api-lab9 --region us-east-1 response.json > /dev/null 2>&1; echo "Healthy invocation $i done"; done
```

---

### Step 8: Verify Recovery in Metrics

**Step 8a:** Wait 1–2 minutes, then check the error metric again:

**Windows (PowerShell):**
```powershell
$endTime = (Get-Date).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
$startTime = (Get-Date).AddMinutes(-10).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
aws cloudwatch get-metric-statistics --namespace "AWS/Lambda" --metric-name Errors --dimensions "Name=FunctionName,Value=workshop-api-lab9" --start-time $startTime --end-time $endTime --period 60 --statistics Sum --region us-east-1
```

**macOS / Linux:**
```bash
aws cloudwatch get-metric-statistics --namespace "AWS/Lambda" --metric-name Errors --dimensions "Name=FunctionName,Value=workshop-api-lab9" --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%SZ) --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) --period 60 --statistics Sum --region us-east-1
```

**✅ You should see** the most recent datapoints showing `"Sum": 0.0` — errors have stopped.

**Step 8b:** If you set up the alarm in Lab 9A, check its state:

```
aws cloudwatch describe-alarms --alarm-names workshop-lab9-errors --region us-east-1 --query "MetricAlarms[0].[AlarmName,StateValue]" --output text
```

**✅ You should see** `workshop-lab9-errors    OK` — the alarm has recovered because errors are back to zero.

> **🎯 The full incident lifecycle, completed:**
> 1. 🚨 **Detection** — alarm fired, email arrived (seconds)
> 2. 🔍 **Investigation** — metrics showed when; logs showed what and where (minutes)
> 3. 🔧 **Fix** — removed the bad line, re-deployed (minutes)
> 4. ✅ **Verification** — metrics confirm recovery, alarm returns to OK
>
> In a real job, you'd also write a post-incident review explaining what happened and what to do to prevent it from happening again. That's exactly what Lab 9C addresses — improving the CI pipeline to catch bugs *before* they ship.

---

### Step 9: ✅ Console Checkpoint — See the Full Story Visually

**Step 9a:** Open the CloudWatch console → **Logs Insights** (left menu):
1. Select log group: `/aws/lambda/workshop-api-lab9`
2. In the query box, paste: `filter @message like /ERROR/ | stats count(*) by bin(5m)`
3. Click **Run query**
4. You should see a bar chart showing errors concentrated in one time window (the broken period) and zero everywhere else

**Step 9b:** Go to **Metrics** → **All metrics** → **Lambda** → **By Function Name**:
1. Tick **Errors** for `workshop-api-lab9`
2. Set the period to the last 1 hour
3. You should see the spike (when the bug was deployed) and the drop back to zero (when you fixed it)

> **💡 This visual is the story of your incident:** healthy → broke → diagnosed → fixed. In a real team, you'd screenshot this graph for the post-incident report.

---

## What You Just Did

| Investigation Step | What You Learned |
|-------------------|-----------------|
| Checked error metrics | **When** the problem started — correlating timestamp with a deploy |
| Compared errors to invocations | **Severity** — 100% failure rate (total outage) |
| Read raw log entries | **What** — the exact error message and stack trace |
| Used Log Insights query | **At scale** — found the smoking gun in seconds, not hours |
| Queried errors-per-minute | **Pattern** — sudden spike (code change), not gradual (load issue) |
| Fixed and re-deployed | **Verification** — metrics confirm recovery |

**Key takeaways:**
- **Metrics tell you *when* and *how bad*.** They're your first stop after an alert.
- **Logs tell you *what* and *where*.** The stack trace is the smoking gun.
- **Log Insights lets you query at scale.** Real applications have millions of log entries — you can't scroll through them.
- **The diagnostic workflow is always:** alert → metrics (when/severity) → logs (what/where) → fix → verify
- **The alarm returning to OK proves the fix worked** — you don't just hope, you measure.

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

The SAA exam tests:
- CloudWatch Logs and Log Groups (what they store, how they're organized)
- Log Insights for querying logs at scale
- Correlating metrics with events (identifying when a change caused a problem)
- Operational Excellence: monitoring, alerting, and incident response
- The relationship between Lambda and CloudWatch (automatic log group creation)

**Sample question type:** "An application team notices errors in their Lambda function. How can they quickly identify the root cause across thousands of log entries?"  
**Answer:** Use CloudWatch Logs Insights to query the function's log group. Filter for ERROR entries, sort by timestamp, and examine the stack traces to identify the failing code and the timestamp when failures began.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `ResourceNotFoundException` on log group | The function hasn't been invoked yet (no logs exist) | Invoke the function first — logs are only created on first invocation |
| Log Insights query returns empty results | Wrong time range or log group not yet populated | Increase the time range (use 3600 seconds = 1 hour); make sure the function has been invoked recently |
| `$stream` variable is empty (Windows) | The describe-log-streams command failed | Verify the log group exists with `aws logs describe-log-groups`; confirm you're in the right region |
| `get-log-events` says stream not found (Windows) | The stream name contains `$LATEST` which PowerShell interprets as a variable | Make sure you stored the stream name in `$stream` using the command in Step 5a (don't type the stream name manually); use `"$stream"` in double quotes |
| `MalformedQueryException` on `start-query` | The start/end time is outside the log group's existence | Make sure `$startTime` and `$endTime` are reasonable Unix timestamps (10-digit numbers). Try using `3600` (1 hour) instead of a larger range |
| Can't see the error spike in console | Time range too narrow or wrong | Change the CloudWatch Metrics graph to "Last 1 hour" and set period to 1 minute |
| Query status shows "Running" not "Complete" | Query hasn't finished yet | Wait another 5 seconds and run `get-query-results` again |

---

## Cleanup

> **📌 If continuing to Lab 9C:** Keep the Lambda function and role. Lab 9C uses them.

**If stopping here, clean up everything:**

```
aws lambda delete-function --function-name workshop-api-lab9 --region us-east-1
aws logs delete-log-group --log-group-name /aws/lambda/workshop-api-lab9 --region us-east-1
aws cloudwatch delete-alarms --alarm-names workshop-lab9-errors --region us-east-1
aws sns delete-topic --topic-arn arn:aws:sns:us-east-1:<YOUR_ACCOUNT_ID>:workshop-lab9-alerts --region us-east-1
aws iam detach-role-policy --role-name workshop-lab9-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name workshop-lab9-lambda-role
```

Delete the project folder:

**Windows (PowerShell):**
```powershell
cd ~\Desktop; Remove-Item -Recurse -Force workshop-lab-9a
```

**macOS / Linux:**
```bash
cd ~/Desktop && rm -rf workshop-lab-9a
```

---

## Help

If you get stuck, post in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** you received
4. Your **operating system** (Windows / Mac / Linux)
