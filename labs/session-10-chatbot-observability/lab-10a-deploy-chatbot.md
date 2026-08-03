# Lab 10A: Deploy a Trivia Chatbot — Your First Structured Logging App

**Session:** 10 — Chatbot Observability  
**Track:** Solutions Architecture  
**Difficulty:** Beginner  
**Estimated Time:** 35–40 minutes  
**Target Cert:** AWS Solutions Architect – Associate (SAA)

---

## Overview

In the real world, applications don't just run in isolation — they call external APIs, process user input, and respond in milliseconds. When something goes wrong (a slow API, a timeout, a bad response), you need to know **exactly what happened and when**. That's where structured logging comes in.

In this lab you'll deploy a "trivia chatbot" Lambda function that:

1. **Accepts a user message** via CLI invocation
2. **Calls an external API** (Open Trivia Database) to fetch a trivia question
3. **Returns a formatted response** with category, question, and answer
4. **Logs structured JSON** to CloudWatch at every stage (request received, API call result with latency, request completed)

**Why structured logging?** Plain text logs look like `"Got a request from user"` — fine for a developer reading one log, but impossible to search or filter across thousands. Structured JSON logs like `{"event": "api_call_success", "api_latency_ms": 342}` let you query, filter, and alert on specific fields. In Lab 10B, you'll use CloudWatch Logs Insights to query these structured logs and answer questions like "what's our average API latency?" and "how many requests failed?"

---

## Prerequisites

- ✅ AWS CLI authenticated (`aws sts get-caller-identity` shows your account)
- ✅ Completed **Lab 1A** (AWS account + CLI setup)

> **💡 This lab is standalone.** You do NOT need Sessions 7–9 to complete it. If you completed Lab 9A, great — this builds on the same concepts with a more realistic application.

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| AWS Lambda | Serverless function execution | Always Free (1M requests/month) |
| Amazon CloudWatch Logs | Log storage and search | Always Free (5GB ingestion/month) |

**Estimated cost for this lab: $0.00**

All services used are within the AWS Always Free tier. Complete the Cleanup section at the end to remove resources.

---

## Concepts

**AWS Lambda** — a serverless compute service. You upload code, and AWS runs it when triggered — no servers to manage. You only pay for the milliseconds your code actually runs. Lambda automatically scales: if 1,000 people call your chatbot at once, AWS spins up 1,000 copies of your function.

**External API Call** — when your code calls another service over the internet to get data. In this lab, your Lambda function calls the Open Trivia Database API to fetch a trivia question. This is how real applications work: a chatbot might call OpenAI, a weather app calls a weather API, a payment system calls Stripe. The challenge? External APIs can be slow, return errors, or go down entirely — and you need to know when that happens.

**Structured Logging** — writing log messages as JSON objects with consistent fields instead of free-text strings. Instead of `print("Got a question from the user")`, you write `logger.info(json.dumps({"event": "request_received", "user_message": "..."}))`. This makes logs:
- **Searchable** — find all entries where `event` equals `api_call_failed`
- **Filterable** — show only entries where `api_latency_ms` is greater than 500
- **Alertable** — trigger a notification when `level` equals `ERROR`

**JSON Logging vs Plain Text** — compare these two CloudWatch log entries:
- Plain text: `INFO: Called trivia API, took 342ms, got back a question about Science`
- Structured JSON: `{"event": "api_call_success", "api_latency_ms": 342, "api_status": 200}`

Both contain the same info, but only the JSON version lets you run queries like "average api_latency_ms over the last hour" or "count of entries where api_status != 200." Production teams always use structured logging because they operate at a scale where reading individual log lines is impossible.

**CloudWatch Logs** — AWS's log storage and query service. Every Lambda function automatically sends its `print()` and `logger.info()` output here. You can view logs in the console, search them, and (in Lab 10B) run SQL-like queries across them with CloudWatch Logs Insights.

---

## ⚠️ Reminder: How to Read Commands

Commands go in your terminal (PowerShell on Windows, Terminal on Mac/Linux). Files are created in a text editor (VS Code recommended — right-click folder → New File).

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account number | `123456789012` |

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

**📝 Write down your 12-digit account ID** — you'll need it when deploying the function.

**Step 1c:** Create a project folder for this lab.

**Windows (PowerShell):**
```powershell
mkdir ~\Desktop\workshop-lab-10a
cd ~\Desktop\workshop-lab-10a
pwd
```

**macOS / Linux:**
```bash
mkdir ~/Desktop/workshop-lab-10a
cd ~/Desktop/workshop-lab-10a
pwd
```

**✅ You should see** the path to your new folder (e.g., `C:\Users\YourName\Desktop\workshop-lab-10a`).

---

### Step 2: Create the Chatbot Lambda Function Code

This is the chatbot code. It receives a user message, calls the Open Trivia Database API to get a trivia question, and returns the result — logging structured JSON at every step.

**Step 2a:** Open your text editor and create a new file.

**Step 2b:** 📋 Copy and paste this code:

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
    """Chatbot that returns a trivia question from an external API."""
    start_time = time.time()
    request_id = context.aws_request_id

    # Parse user input
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

        # Format the trivia question
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

**Step 2c:** Save the file as `handler.py` in your `workshop-lab-10a` folder.

> ⚠️ **Windows users:** in Notepad, change "Save as type" to **"All Files"** so it saves as `handler.py` and not `handler.py.txt`.

**Step 2d: What does this code do?**

This function has three main phases, and it logs structured JSON at each one:

| Phase | What Happens | Log Event |
|-------|-------------|-----------|
| 1. Receive request | Parses the user's message from the input | `request_received` |
| 2. Call external API | Hits the Open Trivia Database, measures how long it takes | `api_call_success` or `api_call_failed` |
| 3. Return response | Formats the trivia question and sends it back | `request_completed` |

The key insight: every log entry is a JSON object with consistent fields (`event`, `request_id`, `api_latency_ms`). This means in Lab 10B you'll be able to query things like "average API latency across all requests" or "show me all failed API calls."

---

### Step 3: Create the Lambda Execution Role

Before Lambda can run your code, it needs an **IAM role** — permission to exist and write logs.

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

**Step 3b:** Save it as `lambda-trust.json` in your `workshop-lab-10a` folder.

> **What does this do?** It tells AWS "the Lambda service is allowed to assume this role." Without this, Lambda can't use the role to run your code or write logs.

**Step 3c:** Create the role. 📋 Copy and paste:

> **⚠️ Make sure you're in your `workshop-lab-10a` folder** where `lambda-trust.json` is saved. Run `pwd` to check.

```
aws iam create-role --role-name workshop-lab10-lambda-role --assume-role-policy-document file://lambda-trust.json
```

**✅ You should see** JSON output with the role ARN (something like `arn:aws:iam::123456789012:role/workshop-lab10-lambda-role`). If you get a "file not found" error, you're not in the right folder — `cd` back to `workshop-lab-10a`.

**Step 3d:** Give the role permission to write logs. 📋 Copy and paste:

```
aws iam attach-role-policy --role-name workshop-lab10-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

**✅ No output means success.** This attaches AWS's built-in policy that lets Lambda write to CloudWatch Logs.

> **💡 Wait 10 seconds** before the next step. AWS needs a moment to make the role available across all services (IAM role propagation). If you skip this wait, the next command may fail with "The role cannot be assumed."

---

### Step 4: Package and Deploy the Lambda Function

> **⚠️ Verify you're in the right folder first.** Run `pwd` — it should show your `workshop-lab-10a` folder. If not, `cd` back to it. All commands in this step use `file://` and `fileb://` which means "look in the current folder."

**Step 4a:** Zip your handler file.

**Windows (PowerShell):**
```powershell
Compress-Archive -Path handler.py -DestinationPath function.zip -Force
```

**macOS / Linux:**
```bash
zip function.zip handler.py
```

> **⚠️ Important:** Make sure you're in the `workshop-lab-10a` folder when you run this (run `pwd` to check). The zip must contain `handler.py` at the root level — not inside a subfolder. If you accidentally zip from a parent directory, the file will be nested and Lambda won't find it, giving you "Unable to import module 'handler'."

**Step 4b:** Create the Lambda function. 📋 Copy and paste:

```
aws lambda create-function --function-name workshop-chatbot-lab10 --runtime python3.12 --role arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-lab10-lambda-role --handler handler.lambda_handler --zip-file fileb://function.zip --timeout 10 --memory-size 128 --region us-east-1
```

> **⚠️ Replace `<YOUR_ACCOUNT_ID>`** with your 12-digit AWS account ID (the one you wrote down in Step 1b).

**✅ You should see** JSON output showing your function details, including `"State": "Pending"` or `"State": "Active"`.

> **What do the flags mean?**
> - `--function-name workshop-chatbot-lab10` — the name for your function in AWS
> - `--runtime python3.12` — run this code using Python 3.12
> - `--handler handler.lambda_handler` — the file is `handler.py`, the function inside it is `lambda_handler`
> - `--zip-file fileb://function.zip` — upload the code from this zip file (`fileb://` means binary file)
> - `--timeout 10` — kill the function if it runs longer than 10 seconds (important since we're calling an external API that could hang)
> - `--memory-size 128` — allocate 128 MB of memory (the minimum; this also determines CPU power)

---

### Step 5: Invoke the Chatbot and See the Trivia Response

Now you'll call your chatbot! You'll send it a message using a payload file, and it will call the trivia API and return a question.

**Step 5a:** Create the payload file. Open your text editor → new file. 📋 Copy and paste:

```json
{"body": "{\"message\": \"Give me a trivia question\"}"}
```

**Step 5b:** Save it as `payload.json` in your `workshop-lab-10a` folder.

> ⚠️ **Windows users:** in Notepad, change "Save as type" to **"All Files"** so it saves as `payload.json` and not `payload.json.txt`.

> **What does this do?** This simulates what a real user request would look like. The `body` field contains a JSON string with the user's message. Your Lambda function parses this to read the user's input.

**Step 5c:** Invoke the chatbot. 📋 Copy and paste:

> **⚠️ Make sure you're in your `workshop-lab-10a` folder** where `payload.json` is saved. Run `pwd` to check.

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-chatbot-lab10 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-chatbot-lab10 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json
```

**✅ You should see** output like:

```json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

**Step 5d:** Read the chatbot's response. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
Get-Content response.json
```

**macOS / Linux:**
```bash
cat response.json
```

**✅ You should see** something like:

```json
{"statusCode": 200, "headers": {"Content-Type": "application/json"}, "body": "{\"response\": \"Category: Science: Computers\\nQuestion: What does GHz stand for?\\nAnswer: Gigahertz\", \"metadata\": {\"api_latency_ms\": 342.15, \"total_duration_ms\": 345.67}}"}
```

> **🎉 Your chatbot is working!** It received your message, called the Open Trivia Database API, got a trivia question, and returned it to you. The exact question will be different each time (it's random).

> **What's in the response?**
> - `response` — the formatted trivia question with category, question text, and answer
> - `metadata.api_latency_ms` — how long the external API call took (typically 300–600ms)
> - `metadata.total_duration_ms` — total function execution time

---

### Step 6: Invoke Several Times and Check CloudWatch Logs

Now you'll invoke the chatbot a few more times to generate log data, then view the structured JSON logs in CloudWatch.

**Step 6a:** Invoke the chatbot several more times. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
1..5 | ForEach-Object { aws lambda invoke --function-name workshop-chatbot-lab10 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json | Out-Null; Write-Host "Invocation $_ complete" }
```

**macOS / Linux:**
```bash
for i in 1 2 3 4 5; do aws lambda invoke --function-name workshop-chatbot-lab10 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json > /dev/null 2>&1; echo "Invocation $i complete"; done
```

**✅ You should see** "Invocation 1 complete" through "Invocation 5 complete."

**Step 6b:** Wait about 30 seconds for the logs to arrive in CloudWatch, then view the most recent log entries. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
$logStream = (aws logs describe-log-streams --log-group-name /aws/lambda/workshop-chatbot-lab10 --order-by LastEventTime --descending --limit 1 --region us-east-1 | ConvertFrom-Json).logStreams[0].logStreamName
aws logs get-log-events --log-group-name /aws/lambda/workshop-chatbot-lab10 --log-stream-name $logStream --limit 20 --region us-east-1
```

**macOS / Linux:**
```bash
LOG_STREAM=$(aws logs describe-log-streams --log-group-name /aws/lambda/workshop-chatbot-lab10 --order-by LastEventTime --descending --limit 1 --region us-east-1 --query "logStreams[0].logStreamName" --output text)
aws logs get-log-events --log-group-name /aws/lambda/workshop-chatbot-lab10 --log-stream-name "$LOG_STREAM" --limit 20 --region us-east-1
```

**✅ You should see** JSON output with log events. Look for messages containing your structured log entries:

```
{"level": "INFO", "event": "request_received", "request_id": "...", "user_message": "Give me a trivia question"}
{"level": "INFO", "event": "api_call_success", "request_id": "...", "api_url": "...", "api_status": 200, "api_latency_ms": 342.15}
{"level": "INFO", "event": "request_completed", "request_id": "...", "total_duration_ms": 345.67, "api_latency_ms": 342.15}
```

> **💡 What you're seeing:** Three structured log entries per invocation:
> 1. `request_received` — the chatbot got a message from the user
> 2. `api_call_success` — the external trivia API responded (with latency measured!)
> 3. `request_completed` — the chatbot finished processing (with total time measured)
>
> Each entry shares the same `request_id`, so you can trace a single request through all its stages. This is called **request tracing** — essential for debugging in production.

---

### Step 7: Console Checkpoint — View Logs in the CloudWatch Console

Let's see these logs in the AWS Console for a visual view.

**Step 7a:** Open the AWS Console in your browser. Go to **CloudWatch** (search "CloudWatch" in the top search bar).

**Step 7b:** In the left menu, click **Logs** → **Log groups**.

**Step 7c:** Find and click on `/aws/lambda/workshop-chatbot-lab10`.

**Step 7d:** Click on the most recent **Log stream** (the one at the top with the latest timestamp).

**Step 7e:** Look through the log entries. You'll see:
- Lines starting with `START` and `END` — these are Lambda's built-in markers for each invocation
- Lines starting with `[INFO]` followed by your structured JSON — these are YOUR log entries
- Lines starting with `REPORT` — Lambda's summary of duration, memory used, and billed duration

**✅ You should see** your structured JSON entries with fields like `"event": "request_received"`, `"event": "api_call_success"`, and `"event": "request_completed"`.

> **🎯 Why this matters:** In Lab 10B, you'll use CloudWatch Logs Insights to run queries across ALL these log entries at once — for example, "what's the average API latency across all invocations?" or "show me the slowest requests." That's only possible because the logs are structured JSON with consistent field names.

---

## What You Just Did

| What You Built | Why It Matters |
|---------------|----------------|
| Deployed a Lambda chatbot that calls an external API | Real applications always integrate with other services — you built one |
| Measured external API latency in code | You know exactly how long each dependency takes |
| Logged structured JSON at every stage | Logs are searchable, filterable, and queryable (not just human-readable) |
| Viewed structured logs in CloudWatch | You can trace a single request through receive → API call → response |
| Observed latency metadata in responses | Your application reports its own performance (self-instrumentation) |

**Key takeaways:**
- External API calls are the #1 source of latency and errors in real applications
- Structured logging (JSON with consistent fields) makes debugging at scale possible
- Every log entry shares a `request_id` so you can trace one request through its entire lifecycle
- CloudWatch Logs captures everything automatically — you just need to `logger.info()` structured data
- In Lab 10B, you'll query these logs with CloudWatch Logs Insights to answer operational questions

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

The SAA exam tests:
- Lambda as a compute option for serverless architectures
- CloudWatch Logs as the default destination for Lambda function output
- Structured logging as a best practice for operational excellence (Well-Architected Framework)
- Lambda timeout and memory configuration tradeoffs
- External API integration patterns and failure handling

**Sample question type:** "A development team wants to troubleshoot intermittent latency issues in a Lambda function that calls a third-party API. What approach gives them the most actionable data?"  
**Answer:** Implement structured JSON logging that records the API latency for each request, then use CloudWatch Logs Insights to query and aggregate latency metrics across invocations.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `No such file or directory: lambda-trust.json` | You're not in the project folder | Run `cd ~/Desktop/workshop-lab-10a` (or `cd ~\Desktop\workshop-lab-10a` on Windows) and verify with `pwd` |
| `The role cannot be assumed` when creating the function | IAM role hasn't propagated yet | Wait 10 seconds and try again |
| `Function already exists` | You (or a previous attempt) already created it | Skip this step, or delete it first: `aws lambda delete-function --function-name workshop-chatbot-lab10 --region us-east-1` |
| `Unable to import module 'handler'` | The zip structure is wrong — `handler.py` is nested in a subfolder | Delete `function.zip`, make sure you're IN the `workshop-lab-10a` folder (`pwd`), then re-run `Compress-Archive -Path handler.py` (not a path like `.\subfolder\handler.py`) |
| `Invalid base64` error on invoke | Missing `--cli-binary-format raw-in-base64-out` flag | Re-run the invoke command and make sure the `--cli-binary-format raw-in-base64-out` flag is included |
| Response shows `"Sorry, I couldn't fetch a trivia question"` | The external trivia API is temporarily down or your Lambda has no internet | Wait a minute and try again; Lambda in the default config has internet access — if it persists, check you're in `us-east-1` |
| `No such file or directory: payload.json` | You're not in the folder where `payload.json` is saved | Run `pwd` to check, then `cd` to your `workshop-lab-10a` folder |
| CloudWatch log group not found | Logs take a few seconds to create the first time | Wait 30 seconds after the first invoke and try again |
| Log stream query returns empty or error | The log group name is case-sensitive | Make sure you use exactly `/aws/lambda/workshop-chatbot-lab10` (all lowercase) |
| `ExpiredTokenException` or auth errors | Your SSO session expired | Run `aws sso login --profile <YOUR_PROFILE_NAME>`, approve in browser, then re-set your profile variable |

---

## Cleanup

> **💡 If you're continuing to Lab 10B:** Skip cleanup for now — Lab 10B uses this same chatbot function and its logs. Come back here after completing Lab 10B.
>
> **If you're stopping here:** Complete these steps to remove all resources.

**Step 1:** Delete the Lambda function. 📋 Copy and paste:

```
aws lambda delete-function --function-name workshop-chatbot-lab10 --region us-east-1
```

**✅ No output means success.**

**Step 2:** Delete the CloudWatch Log Group. 📋 Copy and paste:

```
aws logs delete-log-group --log-group-name /aws/lambda/workshop-chatbot-lab10 --region us-east-1
```

**✅ No output means success.**

**Step 3:** Remove the Lambda role. 📋 Copy and paste (both commands):

```
aws iam detach-role-policy --role-name workshop-lab10-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

```
aws iam delete-role --role-name workshop-lab10-lambda-role
```

**✅ No output means success** for both commands.

**Step 4:** Delete the project folder.

**Windows (PowerShell):**
```powershell
cd ~\Desktop
Remove-Item -Recurse -Force workshop-lab-10a
```

**macOS / Linux:**
```bash
cd ~/Desktop
rm -rf workshop-lab-10a
```

**✅ Verify cleanup:** 📋 Copy and paste:

```
aws lambda list-functions --region us-east-1 --query "Functions[?FunctionName=='workshop-chatbot-lab10'].FunctionName"
```

**✅ You should see** `[]` — empty list, meaning the function is gone.

---

## Help

If you get stuck, post in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** you received
4. Your **operating system** (Windows / Mac / Linux)
