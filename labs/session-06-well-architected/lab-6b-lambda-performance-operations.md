# Lab 6B: Architecting Lambda for Performance & Operational Excellence

**Session:** 6 — AWS Well-Architected Framework  
**Track:** Solutions Architecture  
**Difficulty:** Intermediate  
**Estimated Time:** 30–35 minutes  
**Target Cert:** AWS Solutions Architect – Associate (SAA)

---

## Overview

In this lab, you will deploy a Lambda function and apply two more pillars of the **AWS Well-Architected Framework**:

- **Performance Efficiency pillar** — right-size your compute so it runs fast without waste
- **Operational Excellence pillar** — know when things break without manually watching

The same **feedback loop** pattern as Lab 6A: you will first see the bad state, then apply the fix, then verify the improvement with measurable data.

**The scenario:** You deployed a Lambda function that processes data. It is running slowly and when it crashes, nobody knows until a customer complains. Your job: make it faster (Performance Efficiency) and make failures visible (Operational Excellence).

---

## Prerequisites

- ✅ Completed **Lab 1A** (AWS CLI installed and configured)
- ✅ AWS CLI authenticated — run `aws sts get-caller-identity` and confirm it returns your account info
- ✅ A text editor for creating files
- ✅ An email address you can check during this lab (for the alert notification)

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| AWS Lambda | Serverless compute | Always Free: 1M requests + 400,000 GB-seconds/month |
| Amazon CloudWatch | Monitoring and alarms | Always Free: 10 alarms, 5 GB log ingestion/month |
| Amazon SNS | Notification service | Always Free: 1,000 email notifications/month |

**Estimated cost for this lab: $0.00**

---

## Concepts

Before we start, here is what each concept means:

**Performance Efficiency pillar** — "Use computing resources efficiently to meet requirements, and maintain that efficiency as demand changes." In Lambda, this means choosing the right memory allocation — too little and the function is slow, too much and you pay for unused capacity.

**Lambda Memory and CPU** — Lambda does not let you choose CPU directly. Instead, when you increase memory, AWS proportionally increases CPU. A function with 256 MB gets 2x the CPU of a function with 128 MB. This is the primary "tuning knob" for Lambda performance.

**Right-sizing** — choosing the configuration that balances performance and cost. If doubling memory cuts execution time in half, the cost is the same (2x memory × 0.5x time = same total). But the user experience is better because it finishes faster.

**Operational Excellence pillar** — "Run and monitor systems to deliver business value, and continually improve supporting processes and procedures." This means: when something breaks, you should know immediately — not hours later when a customer complains.

**CloudWatch Alarm** — a rule that monitors a metric and takes action when a threshold is crossed. For example: "If Lambda errors > 0 in the last minute, send an alert." This is automated monitoring — you do not need a human watching a dashboard.

**CloudWatch Metrics** — numbers that AWS services publish automatically. Lambda publishes: Invocations (how many times it ran), Duration (how long each run took), Errors (how many crashed), and Throttles (how many were rejected due to limits).

---

## ⚠️ Reminder: How to Read Commands in This Lab

Commands are inside gray code boxes. **📋 Copy and paste** them into your terminal.

In this lab, you will save several values as variables so commands are easier to run.

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name from Lab 1A | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account number | `123456789012` |
| `<YOUR_EMAIL>` | An email address you can check during this lab | `jane@example.com` |
| `<TOPIC_ARN>` | The SNS topic ARN (from Step 9) | `arn:aws:sns:us-east-1:123456789012:workshop-waf-alerts` |

---

## Lab Steps

### Step 1: Set Your AWS Profile

**Windows (PowerShell):**

📋 Copy and paste this command, **replacing `<YOUR_PROFILE_NAME>`**:

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**macOS / Linux:**

```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**Verify it works.** 📋 Copy and paste:

```
aws sts get-caller-identity
```

**✅ You should see** your account ID and role.

---

### Step 2: Create Your Project Folder

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
mkdir ~\Desktop\workshop-lab-6b
cd ~\Desktop\workshop-lab-6b
```

**macOS / Linux:**

📋 Copy and paste:

```bash
mkdir ~/Desktop/workshop-lab-6b
cd ~/Desktop/workshop-lab-6b
```

**Verify:** `pwd` should show a path ending in `workshop-lab-6b`.

> **💡 From now on, save ALL files you create in this lab to this folder.**

---

### Step 3: Create the Lambda IAM Role

**Step 3a:** Open your text editor and create a **new, empty file**. 📋 Copy and paste this into the file:

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

**Save the file as `trust-policy.json`** in your `workshop-lab-6b` folder.

**Step 3b:** Create the role and attach the logging policy.

📋 Copy and paste these two commands:

```
aws iam create-role --role-name workshop-waf-lambda-role --assume-role-policy-document file://trust-policy.json
```

```
aws iam attach-role-policy --role-name workshop-waf-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

**✅ No output on the second command means success.**

**⏳ Wait 10 seconds** for the role to propagate.

---

### Step 4: Write the Lambda Function

This function does two things:
- Calculates prime numbers (CPU-intensive work that shows performance differences)
- Crashes on purpose when it receives `{"action": "crash"}` (for the Operational Excellence section)

**Step 4a:** Open your text editor and create a **new, empty file**. 📋 Copy and paste this entire code block:

```python
import json
import time
import math

def lambda_handler(event, context):
    action = event.get('action', '')
    
    if action == 'crash':
        raise Exception("ERROR: Invalid input received. Application crashed!")
    
    start = time.time()
    
    n = event.get('workload_size', 50000)
    primes = []
    for num in range(2, n):
        is_prime = True
        for i in range(2, int(math.sqrt(num)) + 1):
            if num % i == 0:
                is_prime = False
                break
        if is_prime:
            primes.append(num)
    
    duration_ms = round((time.time() - start) * 1000, 2)
    memory_mb = context.memory_limit_in_mb
    
    result = (
        f"PERFORMANCE REPORT\n"
        f"==================\n"
        f"Memory allocated: {memory_mb} MB\n"
        f"Computation time: {duration_ms} ms\n"
        f"Primes found: {len(primes)} (up to {n})\n"
        f"=================="
    )
    
    return {
        'statusCode': 200,
        'body': result
    }
```

**Save the file as `workload_function.py`** in your `workshop-lab-6b` folder.

> **⚠️ Common mistakes:** Make sure the file extension is `.py` (not `.py.txt`). Make sure there are no extra spaces at the beginning.

> **What does this function do?**
> - It calculates prime numbers up to 50,000 — this is CPU-bound work that takes measurable time
> - It reports back: how much memory was allocated and how long the computation took
> - If you send `{"action": "crash"}`, it intentionally throws an error (for testing alerts)

---

### Step 5: Deploy the Function with 128 MB Memory

**Step 5a: Zip the code**

**Windows (PowerShell):**
```powershell
Compress-Archive -Path workload_function.py -DestinationPath workload.zip -Force
```

**macOS / Linux:**
```bash
zip workload.zip workload_function.py
```

**Step 5b: Deploy with minimum memory (128 MB)**

📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>`**:

**Windows (PowerShell):**
```powershell
aws lambda create-function --function-name workshop-waf-workload --runtime python3.12 --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-waf-lambda-role" --handler workload_function.lambda_handler --zip-file fileb://workload.zip --memory-size 128 --timeout 30 --region us-east-1
```

**macOS / Linux:**
```bash
aws lambda create-function \
    --function-name workshop-waf-workload \
    --runtime python3.12 \
    --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-waf-lambda-role" \
    --handler workload_function.lambda_handler \
    --zip-file fileb://workload.zip \
    --memory-size 128 \
    --timeout 30 \
    --region us-east-1
```

**✅ You should see** JSON output with the function details.

> **💡 If you get an error** about the role not being found: wait 10 more seconds and try again.

---

## PART 1 — Performance Efficiency Pillar

### Step 6: 🚨 SEE THE PROBLEM — Slow Performance at 128 MB

First, create the test payload file. **Step 6a:** Open your text editor and create a **new file**. 📋 Copy and paste:

```json
{"workload_size": 50000}
```

**Save as `workload-payload.json`** in your `workshop-lab-6b` folder.

**Step 6b: Invoke the function and measure performance**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-waf-workload --payload file://workload-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 response.json
```
```
Write-Output "=== RESULT (128 MB Memory) ===" (Get-Content response.json | ConvertFrom-Json).body
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-waf-workload --payload file://workload-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 response.json
```
```
echo "=== RESULT (128 MB Memory) ==="cat response.json
```

**✅ You should see something like:**

```
=== RESULT (128 MB Memory) ===
PERFORMANCE REPORT
==================
Memory allocated: 128 MB
Computation time: 710.7 ms
Primes found: 5133 (up to 50000)
==================
```

> **📝 Write down the `Computation time` value:** ______ ms
>
> Your number will be different, but it should be somewhere around **600–800 ms**. This is slow for a function that should complete quickly.

> **💡 Run it 2–3 times.** Lambda runs on shared hardware, so any single run can vary. Invoke the command above a few times and use a typical value (ignore any one-off spikes). This gives you a reliable baseline to compare against.

> **💡 On macOS/Linux:** The raw JSON output will have `\n` characters. Look for the number after `Computation time:` — that is the key metric.

---

### Step 7: ✅ FIX THE PROBLEM — Increase Memory to 256 MB

Doubling the memory also doubles the CPU allocated to your function.

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda update-function-configuration --function-name workshop-waf-workload --memory-size 256 --region us-east-1 --query "MemorySize" --output text
```

**macOS / Linux:**
```bash
aws lambda update-function-configuration --function-name workshop-waf-workload --memory-size 256 --region us-east-1 --query "MemorySize" --output text
```

**✅ You should see:** `256`

**⏳ Wait 3 seconds** for the configuration to take effect.

**Now invoke again with the same workload:**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-waf-workload --payload file://workload-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 response.json

Write-Output "=== RESULT (256 MB Memory) ===" (Get-Content response.json | ConvertFrom-Json).body
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-waf-workload --payload file://workload-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 response.json

echo "=== RESULT (256 MB Memory) ===" cat response.json
```

**✅ You should see something like:**

```
=== RESULT (256 MB Memory) ===
PERFORMANCE REPORT
==================
Memory allocated: 256 MB
Computation time: 350.2 ms
Primes found: 5133 (up to 50000)
==================
```

> **📝 Write down the new `Computation time` value:** ______ ms
>
> Your number should be roughly **half** of what it was at 128 MB (around **300–400 ms**).

> **💡 Run it 2–3 times here too** and use a typical value, for the same shared-hardware reason. Compare the typical 256 MB time against the typical 128 MB time from Step 6.

---

### Step 8: Compare the Results

| Configuration | Computation Time | Improvement |
|---------------|-----------------|-------------|
| 128 MB (before) | ~700 ms | — |
| 256 MB (after) | ~350 ms | **~2x faster** |

> **🎉 Performance Efficiency pillar achieved!** By doubling the memory (and thus CPU), the function runs twice as fast. Same code, same result, better performance.
>
> **💡 The cost insight:** In Lambda, you pay for (memory × time). If you double memory but cut time in half, the cost is approximately the same — but the user experience is much better. Right-sizing means finding the sweet spot where increasing memory no longer provides significant speed improvement.
>
> **💡 Real-world:** AWS Lambda Power Tuning is an open-source tool that automatically tests your function at different memory sizes and finds the optimal configuration. In production, you would use it instead of testing manually.

---

## PART 2 — Operational Excellence Pillar

### Step 9: 🚨 SEE THE PROBLEM — Silent Failures

When your function crashes, what happens? Let's find out.

**Step 9a: Create the crash payload**

Open your text editor and create a **new file**. 📋 Copy and paste:

```json
{"action": "crash"}
```

**Save as `crash-payload.json`** in your `workshop-lab-6b` folder.

**Step 9b: Invoke with the crash payload**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-waf-workload --payload file://crash-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 crash-response.json

Write-Output "=== CRASH RESULT ===" (Get-Content crash-response.json)
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-waf-workload --payload file://crash-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 crash-response.json

echo "=== CRASH RESULT ===" cat crash-response.json
```

**✅ You should see:**

```json
{
    "StatusCode": 200,
    "FunctionError": "Unhandled",
    "ExecutedVersion": "$LATEST"
}
=== CRASH RESULT ===
{"errorMessage": "ERROR: Invalid input received. Application crashed!", ...}
```

> **🚨 The problem:** The function crashed — but who knows? The CLI shows the error because you were watching, but in production, nobody is sitting at a terminal watching invocations. This error would go unnoticed until a customer reports something is broken.
>
> **This breaks the Operational Excellence pillar** — failures are invisible. You need automated alerting.

---

### Step 10: ✅ FIX THE PROBLEM — Set Up Automated Alerting

You will create a CloudWatch Alarm that watches the Lambda error count and sends you an email when it fires.

**Step 10a: Create an SNS Topic for alerts**

📋 Copy and paste:

```
aws sns create-topic --name workshop-waf-alerts --region us-east-1
```

**✅ You should see** JSON output with a `TopicArn`.

> **📝 Write down the TopicArn:** ______________________________

**Step 10b: Subscribe your email to the topic**

📋 Copy and paste, **replacing `<TOPIC_ARN>` and `<YOUR_EMAIL>`**:

```
aws sns subscribe --topic-arn <TOPIC_ARN> --protocol email --notification-endpoint <YOUR_EMAIL> --region us-east-1
```

**✅ You should see** `"SubscriptionArn": "pending confirmation"`.

**Step 10c: Confirm your email subscription**

1. Check your email inbox for a message from **AWS Notifications**
2. Click the **Confirm subscription** link
3. You should see a confirmation page

> **💡 Check your spam/junk folder** if you do not see it within 2 minutes.

**Step 10d: Create the CloudWatch Alarm**

📋 Copy and paste, **replacing `<TOPIC_ARN>`**:

**Windows (PowerShell):**
```powershell
aws cloudwatch put-metric-alarm --alarm-name workshop-lambda-errors --metric-name Errors --namespace AWS/Lambda --dimensions "Name=FunctionName,Value=workshop-waf-workload" --statistic Sum --period 60 --evaluation-periods 1 --threshold 1 --comparison-operator GreaterThanOrEqualToThreshold --alarm-actions <TOPIC_ARN> --region us-east-1
```

**macOS / Linux:**
```bash
aws cloudwatch put-metric-alarm \
    --alarm-name workshop-lambda-errors \
    --metric-name Errors \
    --namespace AWS/Lambda \
    --dimensions "Name=FunctionName,Value=workshop-waf-workload" \
    --statistic Sum \
    --period 60 \
    --evaluation-periods 1 \
    --threshold 1 \
    --comparison-operator GreaterThanOrEqualToThreshold \
    --alarm-actions <TOPIC_ARN> \
    --region us-east-1
```

**✅ No output means success.**

> **What does this alarm do?** It monitors the `Errors` metric for your Lambda function. If the total errors in any 60-second period is >= 1, the alarm fires and sends a message to your SNS topic (which forwards it to your email).
>
> **In plain English:** "If my function crashes even once in a minute, email me immediately."

---

### Step 11: ✅ VERIFY THE FIX — Trigger the Alarm

**Step 11a: Invoke with the crash payload again (to trigger the error metric)**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-waf-workload --payload file://crash-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 crash-response2.json
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-waf-workload --payload file://crash-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 crash-response2.json
```

**✅ You should see** `FunctionError: "Unhandled"` again.

**Step 11b: Wait for the alarm to fire**

The alarm evaluates every 60 seconds. After 1–3 minutes, it should transition to the `ALARM` state.

📋 Copy and paste this command to check the alarm state:

**Windows (PowerShell):**
```powershell
aws cloudwatch describe-alarms --alarm-names workshop-lambda-errors --query "MetricAlarms[0].{Name:AlarmName,State:StateValue}" --output table --region us-east-1
```

**macOS / Linux:**
```bash
aws cloudwatch describe-alarms --alarm-names workshop-lambda-errors --query "MetricAlarms[0].{Name:AlarmName,State:StateValue}" --output table --region us-east-1
```

**✅ You should see one of these states:**

| State | What It Means |
|-------|---------------|
| `INSUFFICIENT_DATA` | The alarm just started and hasn't evaluated yet — **wait 1–2 minutes and run again** |
| `ALARM` | 🚨 The alarm fired! An error was detected and your email was sent. |
| `OK` | No errors in the evaluation period |

> **⏳ Keep running the describe command every 30–60 seconds** until you see `ALARM`.

**Step 11c: Check your email**

Once the alarm enters the `ALARM` state, check your email inbox. You should have received a notification from **AWS Notifications** with the subject "ALARM: workshop-lambda-errors".

> **💡 If you do not receive the email:** Check spam. Also confirm your subscription was confirmed in Step 10c. The alarm may take up to 3 minutes to evaluate and send.

**Step 11d: Console Checkpoint**

1. Go to [https://console.aws.amazon.com/](https://console.aws.amazon.com/)
2. Search for **CloudWatch** and click it
3. Click **Alarms** in the left sidebar
4. You should see **workshop-lambda-errors** with state **In alarm** (red)
5. Click on it to see the metric graph — you should see the error spike

**✅ Checkpoint:** The alarm fired, you received an email, and the console shows the alarm in red. Before this alarm, failures were invisible. Now they are detected and reported automatically within 1 minute.

> **🎉 Operational Excellence pillar achieved!** You went from "failures are invisible" to "I get an email within 1 minute of any crash." In production, this alert would go to PagerDuty, Slack, or a ticketing system — ensuring someone always responds.

---

## What You Just Did

You applied two more Well-Architected Framework pillars to a Lambda function:

| Pillar | Problem You Saw | Fix You Applied | How You Verified |
|--------|----------------|-----------------|------------------|
| **Performance Efficiency** | Function took ~700 ms at 128 MB | Increased memory to 256 MB (doubles CPU) | Same workload completed in ~350 ms (2x faster) |
| **Operational Excellence** | Function crashed silently — nobody knew | CloudWatch Alarm on Errors + SNS email notification | Crash triggered alarm → email arrived automatically |

**Key takeaways:**
- **Right-sizing** is about finding the optimal memory/CPU for your workload. More memory = more CPU = faster execution.
- **Automated alerting** means failures are never invisible. You do not rely on humans watching dashboards.
- Both fixes cost $0.00 to implement (free tier) but dramatically improve the quality of your system.

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

The SAA exam tests:
- Lambda memory configuration and its relationship to CPU allocation
- CloudWatch Alarms: metrics, thresholds, evaluation periods, actions
- SNS integration with CloudWatch for notifications
- The Performance Efficiency and Operational Excellence pillars

**Sample question type:** "A Lambda function is running slowly. What is the FIRST configuration change that could improve performance?"  
**Answer:** Increase the memory allocation — Lambda allocates CPU proportionally to memory.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `create-function` fails with "role cannot be assumed" | IAM role hasn't propagated yet | Wait 10 seconds and try again |
| `invoke` returns an error about the payload | PowerShell is mangling the JSON | Make sure you are using `--payload file://` (from a file), not inline JSON |
| Computation time is similar at 128 MB and 256 MB | The workload may be too small to show a difference | Try with `"workload_size": 100000` instead of 50000 |
| Alarm stays in `INSUFFICIENT_DATA` for more than 5 minutes | The error metric hasn't populated yet | Invoke the crash payload one more time, then wait 2 minutes |
| Email notification not received | Subscription may not be confirmed, or email is in spam | Check spam. Run `aws sns list-subscriptions-by-topic` to verify the subscription is confirmed. |
| `(Get-Content ... \| ConvertFrom-Json).body` shows error | The response file doesn't exist or is in wrong folder | Make sure you are in the `workshop-lab-6b` folder. Run `pwd` to confirm. |

---

## Cleanup

**⚠️ Important:** Clean up all resources to avoid any charges.

### Step 1: Delete the CloudWatch Alarm

📋 Copy and paste:

```
aws cloudwatch delete-alarms --alarm-names workshop-lambda-errors --region us-east-1
```

**✅ No output means success.**

### Step 2: Unsubscribe and Delete the SNS Topic

Check for active subscriptions:

```
aws sns list-subscriptions --region us-east-1
```

Unsubscribe from the topic linked to the current workshop:

```
aws sns unsubscribe --subscription-arn <SUBSCRIPTION_ARN> --region us-east-1
```

Command to list topics:

```
aws sns list-topics --region us-east-1
```

📋 Copy and paste, **replacing `<TOPIC_ARN>`**:

```
aws sns delete-topic --topic-arn <TOPIC_ARN> --region us-east-1
```

**✅ No output means success.**

### Step 3: Delete the Lambda Function

📋 Copy and paste:

```
aws lambda delete-function --function-name workshop-waf-workload --region us-east-1
```

**✅ JSON output with "StatusCode": 204 means success.**

### Step 4: Delete the IAM Role

📋 Copy and paste these two commands:

```
aws iam detach-role-policy --role-name workshop-waf-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

```
aws iam delete-role --role-name workshop-waf-lambda-role
```

**✅ No output means success for each.**

### Step 5: Delete Local Files

**Windows (PowerShell):**
```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-6b
```

**macOS / Linux:**
```bash
cd ~/Desktop
rm -rf ~/Desktop/workshop-lab-6b
```

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** you received (copy and paste it)
4. Your **operating system** (Windows, Mac, or Linux)
