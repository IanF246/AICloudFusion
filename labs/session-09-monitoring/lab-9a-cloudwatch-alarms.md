# Lab 9A: Deploy, Monitor, and Alert — Your First CloudWatch Alarm

**Session:** 9 — Monitoring & Observability  
**Track:** Solutions Architecture  
**Difficulty:** Beginner  
**Estimated Time:** 35–40 minutes  
**Target Cert:** AWS Solutions Architect – Associate (SAA)

---

## Overview

Your infrastructure is deployed. Your pipeline ships code. But **how do you know it's working?** Right now, if your Lambda function started crashing at 2am, nobody would know until users complained. That's not acceptable in the real world.

In this lab you'll set up the monitoring foundation that professional teams rely on:

1. **Deploy a Lambda function** (a simple API endpoint)
2. **Invoke it** and observe its healthy metrics in CloudWatch
3. **Create an SNS notification topic** so you can receive email alerts
4. **Create a CloudWatch Alarm** that fires when errors exceed a threshold
5. **Trigger the alarm** by invoking a broken function, and receive the alert email

**The feedback loop:** You'll start with a function that works — and no alerting. Then you'll see that when errors happen, nobody notices. After adding the alarm, the *same errors* now trigger immediate notification. Monitoring turns silent failures into visible, actionable alerts.

---

## Prerequisites

- ✅ AWS CLI authenticated (`aws sts get-caller-identity` shows your account)
- ✅ An **email address** you can check during the lab (for SNS confirmation)
- ✅ Completed **Lab 1A** (AWS account + CLI setup)

> **💡 This lab is standalone.** You do NOT need Sessions 7–8 (IaC/CI-CD) to complete it. If you did complete those sessions, great — Session 9B will build on that pipeline.

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| AWS Lambda | Serverless function execution | Always Free (1M requests/month) |
| Amazon CloudWatch | Metrics, alarms, logs | Always Free (10 alarms, 5GB logs) |
| Amazon SNS | Email notifications | Always Free (1,000 emails/month) |

**Estimated cost for this lab: $0.00**

All services used are within the AWS Always Free tier. Complete the Cleanup section at the end to remove resources.

---

## Concepts

**Amazon CloudWatch** — AWS's monitoring service. It automatically collects metrics (numbers over time) from every AWS service. Think of it as a dashboard for your cloud — CPU usage, error counts, response times, all tracked and graphable.

**Metric** — A single measurement tracked over time. For Lambda, AWS automatically tracks: how many times it ran (Invocations), how many times it crashed (Errors), how long each run took (Duration), and whether it was too slow (Throttles). You don't have to set these up — they appear automatically.

**Alarm** — A rule you create that watches a metric and takes action when it crosses a threshold. Example: "If Errors is greater than or equal to 1 in any 1-minute period, send me an email." The alarm has three states:
- **OK** — the metric is below the threshold (everything is fine)
- **ALARM** — the metric crossed the threshold (something is wrong)
- **INSUFFICIENT_DATA** — not enough data yet to evaluate (just created, or no invocations)

**Amazon SNS (Simple Notification Service)** — delivers messages to subscribers. You create a "topic" (a channel), subscribe your email to it, and then CloudWatch sends alerts to the topic when an alarm fires. The email arrives within seconds.

**Why monitoring matters** — without it, you're flying blind. A function could be failing 100% of requests and you'd only find out when someone manually checks or a customer complains. Alarms give you **proactive visibility**: the system tells *you* there's a problem, instead of you discovering it by accident.

---

## ⚠️ Reminder: How to Read Commands

Commands go in your terminal (PowerShell on Windows, Terminal on Mac/Linux). Files are created in a text editor (VS Code recommended — right-click folder → New File).

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name | `AdministratorAccess-123456789012` |
| `<YOUR_EMAIL>` | An email address you can check right now | `jane@example.com` |

---

## Lab Steps

### Step 1: Set Your AWS Profile and Create a Project Folder

**Step 1a:** Set your AWS profile.

**Windows (PowerShell):**
```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**macOS / Linux:**
```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

> **⚠️ This setting is lost if you close your terminal.** If any command later in this lab gives an authentication error, come back here and re-set it.

**Step 1b:** Verify you're authenticated. 📋 Copy and paste:

```
aws sts get-caller-identity
```

**✅ You should see** your account ID and role. If you get an error about expired tokens, run `aws sso login --profile <YOUR_PROFILE_NAME>` first, approve in the browser, and try again.

**📝 Write down your 12-digit account ID** — you'll need it several times in this lab.

**Step 1c:** Create a project folder for this lab.

**Windows (PowerShell):**
```powershell
mkdir ~\Desktop\workshop-lab-9a
cd ~\Desktop\workshop-lab-9a
pwd
```

**macOS / Linux:**
```bash
mkdir ~/Desktop/workshop-lab-9a
cd ~/Desktop/workshop-lab-9a
pwd
```

**✅ You should see** the path to your new folder.

---

### Step 2: Create the Lambda Function Code

You'll deploy a simple API function that returns a JSON response. This represents a real application endpoint — something users hit when they visit your website or app.

**Step 2a:** Open your text editor and create a new file.

**Step 2b:** 📋 Copy and paste this code:

```python
import json
import logging
import time

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """Workshop API - returns a health check response."""
    start_time = time.time()

    logger.info(json.dumps({
        "level": "INFO",
        "message": "Request received",
        "request_id": context.aws_request_id,
        "path": event.get("path", "/"),
        "method": event.get("httpMethod", "GET")
    }))

    # Application logic
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

**Step 2c:** Save the file as `handler.py` in your `workshop-lab-9a` folder.

> ⚠️ **Windows users:** in Notepad, change "Save as type" to "All Files" so it saves as `handler.py` and not `handler.py.txt`.

**Step 2d: What does this code do?**
- It's a Lambda function that acts as an API endpoint
- When called, it logs structured information (who called, what path, how long it took)
- It returns a `200` status code with a JSON body: `{"status": "healthy", "message": "Hello from the workshop API!"}`
- The structured logging (`json.dumps({...})`) is important — it makes logs searchable later (Lab 9B)

---

### Step 3: Create the Lambda Execution Role

Before Lambda can run your code, it needs an **IAM role** — permission to exist and write logs. This role says "Lambda is allowed to run and send its output to CloudWatch Logs."

**Step 3a:** Create the trust policy file. Open your text editor → new file. 📋 Copy and paste:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "lambda.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

**Step 3b:** Save it as `lambda-trust.json` in your `workshop-lab-9a` folder.

> **What does this do?** It tells AWS "the Lambda service is allowed to assume this role." Without this, Lambda can't use the role.

**Step 3c:** Create the role. 📋 Copy and paste (from your `workshop-lab-9a` folder, where `lambda-trust.json` is):

```
aws iam create-role --role-name workshop-lab9-lambda-role --assume-role-policy-document file://lambda-trust.json
```

**✅ You should see** JSON output with the role ARN. No output means an error — check that you're in the right folder.

**Step 3d:** Give the role permission to write logs. 📋 Copy and paste:

```
aws iam attach-role-policy --role-name workshop-lab9-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

**✅ No output means success.** This attaches AWS's built-in policy that lets Lambda write to CloudWatch Logs.

> **💡 Wait 10 seconds** before the next step. AWS needs a moment to make the role available (IAM role propagation).

---

### Step 4: Package and Deploy the Lambda Function

> **⚠️ Verify you're in the right folder first.** Run `pwd` — it should show your `workshop-lab-9a` folder. If not, `cd` back to it. All commands in this step use `file://` which means "look in the current folder."

**Step 4a:** Zip your handler file.

**Windows (PowerShell):**
```powershell
Compress-Archive -Path handler.py -DestinationPath function.zip -Force
```

**macOS / Linux:**
```bash
zip function.zip handler.py
```

> **⚠️ Important:** Make sure you're in the `workshop-lab-9a` folder when you run this (run `pwd` to check). The zip must contain `handler.py` at the root level — not inside a subfolder. If you accidentally zip from a parent directory (e.g., `Compress-Archive -Path workshop-lab-9a\handler.py`), the file will be nested and Lambda won't find it, giving you "Unable to import module 'handler'."

**Step 4b:** Create the Lambda function. 📋 Copy and paste:

```
aws lambda create-function --function-name workshop-api-lab9 --runtime python3.12 --role arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-lab9-lambda-role --handler handler.lambda_handler --zip-file fileb://function.zip --timeout 10 --memory-size 128 --region us-east-1
```

> **⚠️ Replace `<YOUR_ACCOUNT_ID>`** with your 12-digit AWS account ID (the one you saw in Step 1b).

**✅ You should see** JSON output showing your function details, including `"State": "Pending"` or `"State": "Active"`.

> **What do the flags mean?**
> - `--runtime python3.12` — run this code using Python 3.12
> - `--handler handler.lambda_handler` — the file is `handler.py`, the function inside it is `lambda_handler`
> - `--timeout 10` — kill the function if it runs longer than 10 seconds
> - `--memory-size 128` — allocate 128 MB of memory (the minimum; this also determines CPU)

---

### Step 5: Invoke the Function and See the Healthy Baseline

Now you'll call your function a few times to generate metrics. This establishes what "normal" looks like.

**Step 5a:** Invoke the function. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-api-lab9 --region us-east-1 response.json; Get-Content response.json
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-api-lab9 --region us-east-1 response.json && cat response.json
```

**✅ You should see** two things:
1. `"StatusCode": 200` (the invocation succeeded)
2. The file contents: `{"statusCode": 200, "body": "{\"status\": \"healthy\", \"message\": \"Hello from the workshop API!\"}"}`

**Step 5b:** Invoke it a few more times to generate data points. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
1..5 | ForEach-Object { aws lambda invoke --function-name workshop-api-lab9 --region us-east-1 response.json | Out-Null; Write-Host "Invocation $_ complete" }
```

**macOS / Linux:**
```bash
for i in 1 2 3 4 5; do aws lambda invoke --function-name workshop-api-lab9 --region us-east-1 response.json > /dev/null 2>&1; echo "Invocation $i complete"; done
```

**✅ You should see** "Invocation 1 complete" through "Invocation 5 complete."

---

### Step 6: See Your Function's Metrics in CloudWatch

CloudWatch automatically collects metrics for every Lambda function. Let's look at them.

**Step 6a:** Check the Invocations metric. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
$endTime = (Get-Date).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
$startTime = (Get-Date).AddMinutes(-10).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
aws cloudwatch get-metric-statistics --namespace "AWS/Lambda" --metric-name Invocations --dimensions "Name=FunctionName,Value=workshop-api-lab9" --start-time $startTime --end-time $endTime --period 60 --statistics Sum --region us-east-1
```

**macOS / Linux:**
```bash
aws cloudwatch get-metric-statistics --namespace "AWS/Lambda" --metric-name Invocations --dimensions "Name=FunctionName,Value=workshop-api-lab9" --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%SZ) --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) --period 60 --statistics Sum --region us-east-1
```

**✅ You should see** JSON with `"Datapoints"` showing your invocations. The `"Sum"` values should add up to 6 (your 1 + 5 invocations).

> **💡 If Datapoints is empty:** CloudWatch metrics can take 1–2 minutes to appear. Wait a minute and try again.

**Step 6b:** Check the Errors metric. 📋 Copy and paste (same command, different metric name):

**Windows (PowerShell):**
```powershell
aws cloudwatch get-metric-statistics --namespace "AWS/Lambda" --metric-name Errors --dimensions "Name=FunctionName,Value=workshop-api-lab9" --start-time $startTime --end-time $endTime --period 60 --statistics Sum --region us-east-1
```

**macOS / Linux:**
```bash
aws cloudwatch get-metric-statistics --namespace "AWS/Lambda" --metric-name Errors --dimensions "Name=FunctionName,Value=workshop-api-lab9" --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%SZ) --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) --period 60 --statistics Sum --region us-east-1
```

**✅ You should see** `"Sum": 0.0` — no errors. This is your **healthy baseline**. Remember this: zero errors means your app is running correctly.

**Step 6c: ✅ Console Checkpoint** — Open the AWS Console in your browser:
1. Go to **CloudWatch** (search "CloudWatch" in the top bar)
2. In the left menu, click **All metrics**
3. Under "Browse", click **Lambda**
4. Click **By Function Name**
5. Find `workshop-api-lab9` and tick the checkbox next to **Invocations**

You should see a graph with your 6 invocations as a data point. **This is what "healthy" looks like** — invocations happening, errors at zero.

> **💡 What you're seeing:** CloudWatch automatically collects these metrics for every Lambda function. You didn't have to configure anything — AWS does it for you. The power comes from what you *do* with them (alarms, dashboards, queries).

---

### Step 7: Create an SNS Topic for Alert Emails

Before creating an alarm, you need somewhere to send the alert. SNS (Simple Notification Service) delivers messages — in this case, an email to you.

**Step 7a:** Create the topic. 📋 Copy and paste:

```
aws sns create-topic --name workshop-lab9-alerts --region us-east-1
```

**✅ You should see** JSON with a `"TopicArn"`. **📝 Copy the TopicArn** — you'll use it in the next steps. It looks like: `arn:aws:sns:us-east-1:123456789012:workshop-lab9-alerts`

**Step 7b:** Subscribe your email to the topic. 📋 Copy and paste, **replacing `<YOUR_EMAIL>`**:

```
aws sns subscribe --topic-arn arn:aws:sns:us-east-1:<YOUR_ACCOUNT_ID>:workshop-lab9-alerts --protocol email --notification-endpoint <YOUR_EMAIL> --region us-east-1
```

> **⚠️ Replace both `<YOUR_ACCOUNT_ID>` and `<YOUR_EMAIL>`.**

**✅ You should see** `"SubscriptionArn": "pending confirmation"`.

**Step 7c: ⚠️ IMPORTANT — Confirm the subscription.** Check your email inbox (and spam folder). You'll receive an email from AWS with a "Confirm subscription" link. **Click it.** The alarm won't send emails until you confirm.

**✅ You should see** a web page saying "Subscription confirmed!"

> **What just happened?** You created a notification channel and proved you own the email. AWS requires this confirmation so nobody can subscribe *someone else's* email and spam them with alerts.

---

### Step 8: Create a CloudWatch Alarm

Now the key step: an alarm that watches the Lambda Errors metric and fires if **any errors occur** in a 1-minute window.

📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>`**:

```
aws cloudwatch put-metric-alarm --alarm-name workshop-lab9-errors --metric-name Errors --namespace "AWS/Lambda" --statistic Sum --period 60 --threshold 1 --comparison-operator GreaterThanOrEqualToThreshold --evaluation-periods 1 --dimensions "Name=FunctionName,Value=workshop-api-lab9" --alarm-actions arn:aws:sns:us-east-1:<YOUR_ACCOUNT_ID>:workshop-lab9-alerts --region us-east-1
```

**✅ No output means success.**

> **What do the flags mean?**
> - `--metric-name Errors` — watch the Errors metric
> - `--namespace "AWS/Lambda"` — in the Lambda metrics namespace
> - `--statistic Sum` — add up all errors in the period
> - `--period 60` — evaluate every 60 seconds
> - `--threshold 1` — fire if the sum is ≥ 1
> - `--evaluation-periods 1` — only need 1 bad period to trigger (not "3 in a row")
> - `--alarm-actions` — when it fires, send a message to your SNS topic

**Step 8a:** Check the alarm state. 📋 Copy and paste:

```
aws cloudwatch describe-alarms --alarm-names workshop-lab9-errors --region us-east-1 --query "MetricAlarms[0].[AlarmName,StateValue]" --output text
```

**✅ You should see:** `workshop-lab9-errors    INSUFFICIENT_DATA`

That's correct — the alarm just started and hasn't evaluated a data point yet. It will move to `OK` within 1–2 minutes (since there are currently zero errors).

**Step 8b: ✅ Console Checkpoint** — In the CloudWatch console:
1. In the left menu, click **Alarms** → **All alarms**
2. You should see `workshop-lab9-errors` with state **Insufficient data** (grey) or **OK** (green)

> **💡 What you've built so far:** A monitored application. If errors start happening, you'll get an email within 60 seconds. This is the foundation of operational awareness — you no longer need to manually check if things are working.

---

### Step 9: 🚨 Break the Function — See the Alarm Fire

Now you'll simulate what happens when a bad code change reaches production. You'll deploy a broken version of the function and watch the alarm do its job.

**Step 9a:** Create the broken version. Open your text editor → new file. 📋 Copy and paste:

```python
import json
import logging
import time

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """Workshop API - THIS VERSION HAS A BUG."""
    start_time = time.time()

    logger.info(json.dumps({
        "level": "INFO",
        "message": "Request received",
        "request_id": context.aws_request_id,
        "path": event.get("path", "/"),
        "method": event.get("httpMethod", "GET")
    }))

    # BUG: someone removed the database_url variable but left this reference
    connection_string = database_url

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

**Step 9b:** Save it as `handler.py` in your `workshop-lab-9a` folder (overwriting the good version).

> **What's the bug?** Line 21 references a variable called `database_url` that doesn't exist anywhere in the code. This is a realistic mistake — someone refactored, removed the config where `database_url` was defined, but forgot to remove the line that uses it. Python won't catch this until the code actually runs.

**Step 9c:** Re-package and deploy the broken code.

> **⚠️ Verify you're still in the `workshop-lab-9a` folder** (`pwd`).

**Windows (PowerShell):**
```powershell
Compress-Archive -Path handler.py -DestinationPath function.zip -Force
aws lambda update-function-code --function-name workshop-api-lab9 --zip-file fileb://function.zip --region us-east-1 --query "LastUpdateStatus" --output text
```

**macOS / Linux:**
```bash
zip function.zip handler.py
aws lambda update-function-code --function-name workshop-api-lab9 --zip-file fileb://function.zip --region us-east-1 --query "LastUpdateStatus" --output text
```

> **⚠️ Verify you're in the right folder** (`pwd` should show `workshop-lab-9a`). If the update returns `Successful` but invoking the function still shows the OLD healthy response, the zip didn't package correctly. Delete `function.zip`, verify `handler.py` has the `database_url` bug on line 21, and re-zip.

**✅ You should see:** `InProgress` or `Successful`

**Step 9d:** Invoke the broken function. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-api-lab9 --region us-east-1 response.json; Get-Content response.json
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-api-lab9 --region us-east-1 response.json && cat response.json
```

**✅ You should see:**
1. In the invoke output: `"FunctionError": "Unhandled"` — this tells you the function crashed
2. In `response.json`: `{"errorMessage": "name 'database_url' is not defined", "errorType": "NameError", ...}`

**That's the bug in action.** Every request to this function now fails.

**Step 9e:** Invoke it a few more times to make the error spike obvious:

**Windows (PowerShell):**
```powershell
1..5 | ForEach-Object { aws lambda invoke --function-name workshop-api-lab9 --region us-east-1 response.json | Out-Null; Write-Host "Error invocation $_ done" }
```

**macOS / Linux:**
```bash
for i in 1 2 3 4 5; do aws lambda invoke --function-name workshop-api-lab9 --region us-east-1 response.json > /dev/null 2>&1; echo "Error invocation $i done"; done
```

---

### Step 10: Watch the Alarm Trigger

**Step 10a:** Wait 1–2 minutes for CloudWatch to evaluate the new data, then check the alarm state. 📋 Copy and paste:

```
aws cloudwatch describe-alarms --alarm-names workshop-lab9-errors --region us-east-1 --query "MetricAlarms[0].[AlarmName,StateValue,StateReason]" --output text
```

**✅ You should see:** `workshop-lab9-errors    ALARM    Threshold Crossed: 1 datapoint [X.0 (date)] was greater than or equal to the threshold (1.0).`

> **📝 If you still see `OK` or `INSUFFICIENT_DATA`:** wait another minute and try again. CloudWatch evaluates on 60-second windows and it may not have processed the latest invocations yet.

**Step 10b: 📧 Check your email.** You should have received an alert from AWS with subject "ALARM: workshop-lab9-errors". The email body tells you which metric crossed which threshold and when.

**Step 10c: ✅ Console Checkpoint** — In the CloudWatch console:
1. Go to **Alarms** → **All alarms**
2. `workshop-lab9-errors` should now show state **In alarm** (red)
3. Click on the alarm name to see the graph — you'll see the error count spike from 0 to 6

> **🎯 What you just proved:** Without the alarm, those 6 failures were completely silent. The function was broken and nobody knew. With the alarm, you got an email within 60 seconds. That's the difference between "users complain" and "the team fixes it before users notice."

---

## What You Just Did

| What You Built | Why It Matters |
|---------------|----------------|
| Deployed a Lambda function | A real API endpoint serving requests |
| Observed healthy metrics in CloudWatch | You know what "normal" looks like (zero errors, consistent invocations) |
| Created an SNS topic + email subscription | You have a notification channel ready to receive alerts |
| Created a CloudWatch Alarm on Errors | The system now proactively tells you when something breaks |
| Broke the function and watched the alarm fire | Proved the monitoring works — silent failures become visible alerts |

**Key takeaways:**
- CloudWatch **automatically** collects metrics for Lambda (no setup needed)
- An alarm turns a passive metric into an **active notification**
- The alarm fired because errors went from 0 → 6 and crossed the threshold of 1
- In production, this is how teams achieve **"mean time to detect"** measured in seconds, not hours

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

The SAA exam tests:
- CloudWatch metrics, alarms, and actions (what they are and when to use them)
- SNS as a notification target for alarms
- The difference between `OK`, `ALARM`, and `INSUFFICIENT_DATA` states
- Lambda monitoring (built-in metrics: Invocations, Errors, Duration, Throttles)
- Operational Excellence pillar: monitoring and alerting as a core practice

**Sample question type:** "A team wants to be notified immediately when a Lambda function's error rate increases. Which AWS services should they use?"  
**Answer:** Create a CloudWatch Alarm on the Lambda Errors metric with an SNS topic as the alarm action. Subscribe the team's email or pager to the topic.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `No such file or directory: lambda-trust.json` | You're not in the project folder | Run `cd ~/Desktop/workshop-lab-9a` (or the Windows equivalent) and verify with `pwd` |
| `The role cannot be assumed` when creating the function | IAM role hasn't propagated yet | Wait 10 seconds and try again |
| `Function already exists` | You (or a previous attempt) already created it | Skip this step, or delete it first: `aws lambda delete-function --function-name workshop-api-lab9 --region us-east-1` |
| `Unable to import module 'handler'` | The zip structure is wrong — `handler.py` is nested in a subfolder | Delete `function.zip`, make sure you're IN the `workshop-lab-9a` folder (`pwd`), then re-run `Compress-Archive -Path handler.py` (not a path like `.\subfolder\handler.py`) |
| Function invocation returns the OLD response after updating code | Zip packaged the wrong file or a stale version | Delete `function.zip`, confirm `handler.py` has your latest changes (open it and check), re-zip, and re-deploy with `update-function-code` |
| CloudWatch metrics show empty Datapoints | Metrics take 1–2 minutes to appear | Wait a minute and re-run the get-metric-statistics command |
| Alarm stays in INSUFFICIENT_DATA | No error data in the evaluation window yet | Invoke the broken function again and wait 1–2 minutes |
| No email received | You didn't confirm the SNS subscription | Check spam folder; look for "AWS Notification - Subscription Confirmation" email and click "Confirm subscription" |
| `InvalidParameterValue` on put-metric-alarm | Account ID wrong in the SNS ARN | Double-check the `--alarm-actions` ARN matches your account ID and topic name |

---

## Cleanup

Complete these steps to remove all resources created in this lab.

**Step 1:** Delete the CloudWatch alarm:
```
aws cloudwatch delete-alarms --alarm-names workshop-lab9-errors --region us-east-1
```

**Step 2:** Delete the SNS topic (replace `<YOUR_ACCOUNT_ID>`):
```
aws sns delete-topic --topic-arn arn:aws:sns:us-east-1:<YOUR_ACCOUNT_ID>:workshop-lab9-alerts --region us-east-1
```

**Step 3:** Delete the Lambda function:
```
aws lambda delete-function --function-name workshop-api-lab9 --region us-east-1
```

**Step 4:** Delete the CloudWatch Log Group (Lambda created this automatically):
```
aws logs delete-log-group --log-group-name /aws/lambda/workshop-api-lab9 --region us-east-1
```

**Step 5:** Remove the Lambda role:
```
aws iam detach-role-policy --role-name workshop-lab9-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name workshop-lab9-lambda-role
```

**Step 6:** Delete the project folder.

**Windows (PowerShell):**
```powershell
cd ~\Desktop
Remove-Item -Recurse -Force workshop-lab-9a
```

**macOS / Linux:**
```bash
cd ~/Desktop
rm -rf workshop-lab-9a
```

**✅ Verify cleanup:** Run `aws lambda list-functions --region us-east-1 --query "Functions[?FunctionName=='workshop-api-lab9'].FunctionName"` — it should return `[]`.

---

## Help

If you get stuck, post in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** you received
4. Your **operating system** (Windows / Mac / Linux)
