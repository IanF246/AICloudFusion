# Lab 6C: Architecting for Cost Optimization & Sustainability

**Session:** 6 — AWS Well-Architected Framework  
**Track:** Solutions Architecture  
**Difficulty:** Advanced  
**Estimated Time:** 40–45 minutes  
**Target Cert:** AWS Solutions Architect – Associate (SAA)

---

## Overview

In this lab, you will complete the **AWS Well-Architected Framework** by applying the final two pillars:

- **Cost Optimization pillar** — eliminate waste and pay only for what you need
- **Sustainability pillar** — reduce the environmental impact of your cloud workloads

You will build a **custom Cost Projection Dashboard** that calculates and displays the projected monthly cost of your infrastructure based on its current configuration. Then you will make two architectural changes and **watch the projected cost drop on your own dashboard in real time.**

**The scenario:** Your team deployed a Lambda function six months ago. It works fine, but nobody has reviewed the configuration since launch. Logs are growing forever, the function runs on the default architecture, and there is no visibility into what it costs. Your job: build a cost dashboard, identify the waste, fix it, and prove the savings.

**The feedback loop:** For each problem, you will:
1. **SEE** the projected cost on your dashboard (high)
2. **FIX** the configuration
3. **VERIFY** the projected cost drops on the same dashboard (low)

---

## Prerequisites

- ✅ Completed **Lab 1A** (AWS CLI installed and configured)
- ✅ AWS CLI authenticated — run `aws sts get-caller-identity` and confirm it returns your account info
- ✅ **VS Code** installed (from Lab 1B) — any text editor works, but these labs assume VS Code
- ✅ A web browser (to view the CloudWatch Dashboard)

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| AWS Lambda | Serverless compute | Always Free: 1M requests + 400,000 GB-seconds/month |
| Amazon CloudWatch | Dashboards, metrics, logs | Always Free: 3 dashboards, 10 custom metrics, 5 GB log storage/month |
| AWS IAM | Identity and access management | Always Free |

**Estimated cost for this lab: $0.00**

---

## Concepts

Before we start, here is what each concept means:

**Cost Optimization pillar** — "Deliver business value at the lowest price point." This means: don't pay for resources you aren't using, choose the cheapest option that meets your requirements, and set limits so costs don't grow unbounded.

**Sustainability pillar** — "Minimize the environmental impact of running cloud workloads." This means: use efficient hardware (fewer watts per computation), don't store data you don't need (less energy for storage), and right-size resources (less idle capacity).

**AWS Graviton (arm64)** — AWS designed its own processors called Graviton. Lambda functions running on Graviton (`arm64` architecture) are approximately **20% cheaper per millisecond** AND use **less energy per computation** compared to the default `x86_64` (Intel/AMD). Same code, lower cost, smaller carbon footprint.

**CloudWatch Logs Retention** — By default, CloudWatch Logs stores your logs **forever** (never expire). You pay $0.03 per GB per month for stored logs. Setting a retention policy (e.g., 7 days) automatically deletes old logs, capping your storage costs.

**Custom CloudWatch Metrics** — You can publish your own numbers to CloudWatch (not just the ones AWS generates). In this lab, you will publish a "projected monthly cost" metric that your dashboard displays as a graph.

**CloudWatch Dashboard** — A customizable screen in the AWS Console that displays metrics as graphs, numbers, or tables. You will build one that shows your projected costs and Lambda performance side by side.

---

## ⚠️ Reminder: How to Read Commands in This Lab

Commands are inside gray code boxes. **📋 Copy and paste** them into your terminal.

In this lab, you will save your account ID as a variable so commands are easier to run.

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name from Lab 1A | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account number | `123456789012` |
| `<DURATION_MS>` | The computation time from your workload test | `363` |

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
mkdir ~\Desktop\workshop-lab-6c
cd ~\Desktop\workshop-lab-6c
```

**macOS / Linux:**

📋 Copy and paste:

```bash
mkdir ~/Desktop/workshop-lab-6c
cd ~/Desktop/workshop-lab-6c
```

**Verify:** `pwd` should show a path ending in `workshop-lab-6c`.

> **💡 From now on, save ALL files you create in this lab to this folder.**

**Open the folder in VS Code.** 📋 Copy and paste:

```
code .
```

> This opens VS Code with `workshop-lab-6c` as its **file tree**, so the JSON files and the Lambda code you create land in the right place. (You set up the `code` command in Lab 1B — if you see `'code' is not recognized`, close and reopen your terminal, or revisit Lab 1B, Step 6.)

---

### Step 3: Create the IAM Role

Both Lambda functions in this lab will use the same role. It needs permission to run Lambda, write logs, read Lambda configurations, and publish CloudWatch metrics.

**Step 3a:** In the VS Code file tree, create a **New File** named `trust-policy.json`. 📋 Copy and paste this entire block into it:

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

**Save** the file (**Ctrl+S** / **Cmd+S**).

> **What does this file do?** It tells AWS "Lambda functions are allowed to use this role." This is the same trust policy from Lab 6B.

**Step 3b: Create the role**

📋 Copy and paste:

```
aws iam create-role --role-name workshop-waf-cost-role --assume-role-policy-document file://trust-policy.json
```

**✅ You should see** JSON output with the role details.

**Step 3c: Attach the required policies**

📋 Copy and paste these three commands one at a time:

```
aws iam attach-role-policy --role-name workshop-waf-cost-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

```
aws iam attach-role-policy --role-name workshop-waf-cost-role --policy-arn arn:aws:iam::aws:policy/CloudWatchFullAccess
```

```
aws iam attach-role-policy --role-name workshop-waf-cost-role --policy-arn arn:aws:iam::aws:policy/AWSLambda_ReadOnlyAccess
```

**✅ No output means success** for each command.

> **What do these policies do?**
> - `AWSLambdaBasicExecutionRole` — lets the function write its own logs
> - `CloudWatchFullAccess` — lets the function publish custom metrics and read log groups
> - `AWSLambda_ReadOnlyAccess` — lets the cost calculator read the workload function's configuration (memory, architecture)

**⏳ Wait 10 seconds** before continuing. IAM roles take a moment to become usable.

---

### Step 4: Deploy the Workload Lambda (x86_64 — the "wasteful" config)

This is the same prime-number workload from Lab 6B, deployed with the default x86_64 architecture. You will measure its performance and cost, then improve it later.

**Step 4a:** In the VS Code file tree, create a **New File** named `workload_function.py`. 📋 Copy and paste this entire code block into it:

```python
import json
import time
import math

def lambda_handler(event, context):
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

    return {
        'statusCode': 200,
        'body': json.dumps({
            'memory_mb': int(memory_mb),
            'duration_ms': duration_ms,
            'primes_found': len(primes)
        })
    }
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

> **⚠️ Common mistakes:** Make sure the file is named `workload_function.py` (not `workload_function.py.txt`). Make sure there are no extra spaces at the beginning.

> **What does this function do?** It calculates prime numbers — CPU-intensive work that takes measurable time. It reports back the memory allocated and how long the computation took. You will use these numbers for cost projections.

**Step 4b: Zip the code**

**Windows (PowerShell):**
```powershell
Compress-Archive -Path workload_function.py -DestinationPath workload.zip -Force
```

**macOS / Linux:**
```bash
zip workload.zip workload_function.py
```

**Step 4c: Deploy with x86_64 architecture**

📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>`** with your 12-digit account number:

**Windows (PowerShell):**
```powershell
aws lambda create-function --function-name workshop-waf-cost-workload --runtime python3.12 --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-waf-cost-role" --handler workload_function.lambda_handler --zip-file fileb://workload.zip --memory-size 256 --timeout 30 --architectures x86_64 --region us-east-1
```

**macOS / Linux:**
```bash
aws lambda create-function \
    --function-name workshop-waf-cost-workload \
    --runtime python3.12 \
    --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-waf-cost-role" \
    --handler workload_function.lambda_handler \
    --zip-file fileb://workload.zip \
    --memory-size 256 \
    --timeout 30 \
    --architectures x86_64 \
    --region us-east-1
```

**✅ You should see** JSON output with the function details. Look for `"Architectures": ["x86_64"]` in the output.

> **💡 If you get an error** about the role not being found: wait 10 more seconds and try again.

---

### Step 5: Test the Workload and Measure Performance

**Step 5a:** In the VS Code file tree, create a **New File** named `workload-payload.json`. 📋 Copy and paste this into it:

```json
{"workload_size": 50000}
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

**Step 5b: Invoke the function**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-waf-cost-workload --payload file://workload-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 response.json; Write-Output "=== WORKLOAD RESULT ===" (Get-Content response.json | ConvertFrom-Json).body
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-waf-cost-workload --payload file://workload-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 response.json; echo "=== WORKLOAD RESULT ==="; cat response.json
```

**✅ You should see something like:**

```json
{"memory_mb": 256, "duration_ms": 363.4, "primes_found": 5133}
```

> **📝 Write down the `duration_ms` value:** ______ ms
>
> This is your baseline performance on x86_64. You will need this number in the next step.
> Your number will likely be around **300–400 ms**.

> **💡 Run it 2–3 times.** Lambda runs on shared hardware, so results vary slightly. Use a typical value (not the highest or lowest).

---

### Step 6: Deploy the Cost Calculator Lambda

This is the key piece of the lab. The cost calculator reads the **real current configuration** of your workload Lambda and your log group, then calculates a **projected monthly cost** based on published AWS prices. It publishes that projection as a custom CloudWatch metric so your dashboard can graph it.

**Step 6a:** In the VS Code file tree, create a **New File** named `cost_calculator.py`. 📋 Copy and paste this entire code block into it:

```python
import json
import boto3

def lambda_handler(event, context):
    region = 'us-east-1'
    lambda_client = boto3.client('lambda', region_name=region)
    logs_client = boto3.client('logs', region_name=region)
    cw_client = boto3.client('cloudwatch', region_name=region)

    # --- Read inputs ---
    workload_fn_name = 'workshop-waf-cost-workload'
    log_group_name = '/aws/lambda/workshop-waf-cost-workload'
    assumed_monthly_invocations = 1000000  # 1 million
    avg_duration_ms = event.get('avg_duration_ms', 400)

    # --- Read Lambda configuration ---
    fn_config = lambda_client.get_function_configuration(
        FunctionName=workload_fn_name
    )
    memory_mb = fn_config['MemorySize']
    arch = fn_config.get('Architectures', ['x86_64'])[0]

    # --- Lambda pricing (us-east-1, published AWS prices) ---
    if arch == 'arm64':
        price_per_gb_ms = 0.0000000133  # Graviton: ~20% cheaper
    else:
        price_per_gb_ms = 0.0000000167  # x86_64 (Intel/AMD)

    price_per_request = 0.0000002  # $0.20 per 1M requests

    # --- Calculate compute cost ---
    memory_gb = memory_mb / 1024
    cost_per_invocation = (memory_gb * avg_duration_ms * price_per_gb_ms) + price_per_request
    monthly_compute_cost = cost_per_invocation * assumed_monthly_invocations

    # --- Read Log Group configuration ---
    try:
        log_groups = logs_client.describe_log_groups(
            logGroupNamePrefix=log_group_name
        )
        lg = log_groups['logGroups'][0]
        retention_days = lg.get('retentionInDays', 0)
        stored_bytes = lg.get('storedBytes', 0)
    except Exception:
        retention_days = 0
        stored_bytes = 0

    # --- Calculate logs storage cost ---
    # Logs pricing: $0.03 per GB stored per month
    logs_price_per_gb = 0.03

    if retention_days == 0:
        # Never expire: logs grow ~500 MB/month, project 12-month accumulation
        projected_logs_gb = (stored_bytes / (1024 ** 3)) + (0.5 * 12)
        monthly_logs_cost = projected_logs_gb * logs_price_per_gb
        logs_status = "NEVER EXPIRE - grows forever"
    else:
        # With retention: storage bounded to (daily_ingest * retention_days)
        daily_ingest_gb = 0.5 / 30
        projected_logs_gb = daily_ingest_gb * retention_days
        monthly_logs_cost = projected_logs_gb * logs_price_per_gb
        logs_status = f"{retention_days}-day retention - bounded"

    total_monthly_cost = monthly_compute_cost + monthly_logs_cost

    # --- Publish custom metrics to CloudWatch ---
    cw_client.put_metric_data(
        Namespace='Workshop/CostProjection',
        MetricData=[
            {
                'MetricName': 'ProjectedComputeCostUSD',
                'Value': round(monthly_compute_cost, 4),
                'Unit': 'None'
            },
            {
                'MetricName': 'ProjectedLogsCostUSD',
                'Value': round(monthly_logs_cost, 4),
                'Unit': 'None'
            },
            {
                'MetricName': 'ProjectedTotalCostUSD',
                'Value': round(total_monthly_cost, 4),
                'Unit': 'None'
            }
        ]
    )

    # --- Build human-readable report ---
    report = (
        f"COST PROJECTION REPORT\n"
        f"======================\n"
        f"COMPUTE:\n"
        f"  Function: {workload_fn_name}\n"
        f"  Architecture: {arch}\n"
        f"  Memory: {memory_mb} MB\n"
        f"  Avg duration: {avg_duration_ms} ms\n"
        f"  Monthly invocations: {assumed_monthly_invocations:,}\n"
        f"  Projected compute cost: ${monthly_compute_cost:.4f}/month\n"
        f"\n"
        f"LOGS:\n"
        f"  Log group: {log_group_name}\n"
        f"  Retention: {logs_status}\n"
        f"  Projected logs cost: ${monthly_logs_cost:.4f}/month\n"
        f"\n"
        f"TOTAL PROJECTED COST: ${total_monthly_cost:.4f}/month\n"
        f"======================\n"
        f"Metrics published to CloudWatch dashboard."
    )

    return {
        'statusCode': 200,
        'body': report
    }
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

> **⚠️ Common mistakes:** Make sure the file is named `cost_calculator.py` (not `cost_calculator.py.txt`). Make sure there are no extra spaces at the beginning of lines.

> **What does this function do?**
> - Reads the real configuration of your workload Lambda (memory size, architecture)
> - Reads the real configuration of your log group (retention policy, stored bytes)
> - Calculates projected monthly costs using published AWS prices
> - Publishes those projections as custom CloudWatch metrics (so your dashboard can graph them)
> - Returns a human-readable cost report

**Step 6b: Zip the calculator**

**Windows (PowerShell):**
```powershell
Compress-Archive -Path cost_calculator.py -DestinationPath calculator.zip -Force
```

**macOS / Linux:**
```bash
zip calculator.zip cost_calculator.py
```

**Step 6c: Deploy the calculator**

📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>`**:

**Windows (PowerShell):**
```powershell
aws lambda create-function --function-name workshop-waf-cost-calculator --runtime python3.12 --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-waf-cost-role" --handler cost_calculator.lambda_handler --zip-file fileb://calculator.zip --memory-size 256 --timeout 30 --region us-east-1
```

**macOS / Linux:**
```bash
aws lambda create-function \
    --function-name workshop-waf-cost-calculator \
    --runtime python3.12 \
    --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-waf-cost-role" \
    --handler cost_calculator.lambda_handler \
    --zip-file fileb://calculator.zip \
    --memory-size 256 \
    --timeout 30 \
    --region us-east-1
```

**✅ You should see** JSON output with the function details.

---

### Step 7: Create the Cost Projection Dashboard

Before running the calculator, let's create the dashboard so it's ready to display data as soon as you publish your first metric.

**Step 7a:** In the VS Code file tree, create a **New File** named `dashboard.json`. 📋 Copy and paste this entire JSON block into it:

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
                "title": "Projected Monthly Cost (USD)",
                "metrics": [
                    ["Workshop/CostProjection", "ProjectedTotalCostUSD", {"label": "Total Cost", "color": "#d62728"}],
                    ["Workshop/CostProjection", "ProjectedComputeCostUSD", {"label": "Compute Cost", "color": "#1f77b4"}],
                    ["Workshop/CostProjection", "ProjectedLogsCostUSD", {"label": "Logs Cost", "color": "#ff7f0e"}]
                ],
                "region": "us-east-1",
                "period": 60,
                "stat": "Average",
                "view": "timeSeries"
            }
        },
        {
            "type": "metric",
            "x": 12,
            "y": 0,
            "width": 12,
            "height": 6,
            "properties": {
                "title": "Lambda Performance (Duration ms)",
                "metrics": [
                    ["AWS/Lambda", "Duration", "FunctionName", "workshop-waf-cost-workload", {"label": "Avg Duration", "stat": "Average"}]
                ],
                "region": "us-east-1",
                "period": 60,
                "view": "timeSeries"
            }
        },
        {
            "type": "metric",
            "x": 0,
            "y": 6,
            "width": 12,
            "height": 6,
            "properties": {
                "title": "Lambda Invocations",
                "metrics": [
                    ["AWS/Lambda", "Invocations", "FunctionName", "workshop-waf-cost-workload", {"label": "Invocations", "stat": "Sum"}]
                ],
                "region": "us-east-1",
                "period": 60,
                "view": "timeSeries"
            }
        }
    ]
}
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

> **What does this file do?** It defines a CloudWatch dashboard with three widgets:
> - **Projected Monthly Cost** — your custom cost projections as a time-series graph (this is the one you'll watch drop)
> - **Lambda Performance** — real average duration from CloudWatch
> - **Lambda Invocations** — how many times the function ran

**Step 7b: Create the dashboard**

📋 Copy and paste:

```
aws cloudwatch put-dashboard --dashboard-name Workshop-WAF-CostOptimization --dashboard-body file://dashboard.json --region us-east-1
```

**✅ You should see:**
```json
{
    "DashboardValidationMessages": []
}
```

An empty list means the dashboard was created successfully with no errors.

**Step 7c: ✅ Console Checkpoint — View Your Dashboard**

1. Go to [https://console.aws.amazon.com/](https://console.aws.amazon.com/)
2. Search for **CloudWatch** and click it
3. Click **Dashboards** in the left sidebar
4. Click **Workshop-WAF-CostOptimization**

**✅ You should see** the three widgets. They will show "No data available" right now — that's expected. You haven't published any metrics yet. The cost graph will populate in the next step.

---

## PART 1 — Cost Optimization: Compute (Graviton)

### Step 8: 🚨 SEE THE PROBLEM — Run the Cost Calculator (Before)

Now run the calculator to measure the projected cost of your current (wasteful) configuration.

**Step 8a:** In the VS Code file tree, create a **New File** named `calculator-payload.json`. 📋 Copy and paste this into it, **replacing `<DURATION_MS>`** with the typical duration you recorded in Step 5 (just the number, no "ms"):

```json
{"avg_duration_ms": <DURATION_MS>}
```

For example, if your duration was 363 ms, the file would contain:
```json
{"avg_duration_ms": 363}
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

**Step 8b: Invoke the cost calculator**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-waf-cost-calculator --payload file://calculator-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 calc-response.json; Write-Output "=== COST PROJECTION (BEFORE) ===" (Get-Content calc-response.json | ConvertFrom-Json).body
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-waf-cost-calculator --payload file://calculator-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 calc-response.json; echo "=== COST PROJECTION (BEFORE) ==="; cat calc-response.json
```

**✅ You should see something like:**

```
=== COST PROJECTION (BEFORE) ===
COST PROJECTION REPORT
======================
COMPUTE:
  Function: workshop-waf-cost-workload
  Architecture: x86_64
  Memory: 256 MB
  Avg duration: 363 ms
  Monthly invocations: 1,000,000
  Projected compute cost: $1.7155/month

LOGS:
  Log group: /aws/lambda/workshop-waf-cost-workload
  Retention: NEVER EXPIRE - grows forever
  Projected logs cost: $0.1800/month

TOTAL PROJECTED COST: $1.8955/month
======================
Metrics published to CloudWatch dashboard.
```

> **📝 Write down the Total Projected Cost:** $______/month
>
> This is your "before" number. Your dashboard now has its first data point.

**Step 8c: ✅ Console Checkpoint — View the "Before" Cost on Your Dashboard**

1. Go back to your **Workshop-WAF-CostOptimization** dashboard in CloudWatch
2. **Refresh the page** (F5 or click the refresh icon)
3. In the top-right time range selector, choose **Last 1 hour** so you can see recent data points

**✅ You should see** the "Projected Monthly Cost" graph now showing a data point. The red line (Total Cost) should be around $1.90.

> **💡 It may take 1–2 minutes** for the data point to appear. If you don't see it yet, wait a minute and refresh again.

> **🚨 What you're looking at:** This is your infrastructure's projected monthly cost if nothing changes. The function runs on x86_64 (expensive architecture), and logs never expire (storage grows forever). You are going to fix both problems and watch this number drop.

---

### Step 9: ✅ FIX THE PROBLEM — Switch to Graviton (arm64)

AWS Graviton processors are ~20% cheaper per millisecond AND more energy-efficient (better performance per watt). Switching is a single command — same code, same behavior, lower cost, smaller carbon footprint.

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda update-function-code --function-name workshop-waf-cost-workload --zip-file fileb://workload.zip --architectures arm64 --region us-east-1 --query "Architectures" --output text
```

**macOS / Linux:**
```bash
aws lambda update-function-code \
    --function-name workshop-waf-cost-workload \
    --zip-file fileb://workload.zip \
    --architectures arm64 \
    --region us-east-1 \
    --query "Architectures" \
    --output text
```

**✅ You should see:** `arm64`

> **What just happened?** You redeployed the exact same code but told Lambda to run it on Graviton (arm64) instead of Intel/AMD (x86_64). The function still does the same thing — but cheaper and greener.

> **💡 Why `update-function-code` instead of `update-function-configuration`?** The processor architecture is tied to the compiled code package, so AWS requires you to upload the code again when changing architectures. Python doesn't need recompilation, so the same zip works for both.

**⏳ Wait 5 seconds** for the update to complete.

---

### Step 10: Measure the New Performance on Graviton

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-waf-cost-workload --payload file://workload-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 response-arm64.json; Write-Output "=== WORKLOAD RESULT (arm64 Graviton) ===" (Get-Content response-arm64.json | ConvertFrom-Json).body
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-waf-cost-workload --payload file://workload-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 response-arm64.json; echo "=== WORKLOAD RESULT (arm64 Graviton) ==="; cat response-arm64.json
```

**✅ You should see something like:**

```json
{"memory_mb": 256, "duration_ms": 322.5, "primes_found": 5133}
```

> **📝 Write down the new `duration_ms` value:** ______ ms
>
> It should be slightly faster than before (Graviton tends to be ~10% faster on compute-bound work). But the bigger saving is the **price per millisecond** — 20% cheaper regardless of speed.

> **💡 Run it 2–3 times** and use a typical value.

---

### Step 11: ✅ VERIFY THE FIX — Re-Run the Calculator and Watch the Dashboard Drop

Now update your calculator payload with the new duration and re-run it.

**Step 11a:** Open `calculator-payload.json` in VS Code (click it in the file tree). **Replace** the duration with your new Graviton duration. For example:

```json
{"avg_duration_ms": 322}
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

**Step 11b: Run the calculator again**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-waf-cost-calculator --payload file://calculator-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 calc-response2.json; Write-Output "=== COST PROJECTION (AFTER GRAVITON) ===" (Get-Content calc-response2.json | ConvertFrom-Json).body
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-waf-cost-calculator --payload file://calculator-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 calc-response2.json; echo "=== COST PROJECTION (AFTER GRAVITON) ==="; cat calc-response2.json
```

**✅ You should see the compute cost drop significantly:**

```
COMPUTE:
  Architecture: arm64
  Projected compute cost: $1.2707/month
```

> **📝 Compare:**
> - Before (x86_64): ~$1.72/month compute
> - After (arm64): ~$1.27/month compute
> - **Savings: ~26%** (20% from cheaper price + ~10% from faster execution)

**Step 11c: ✅ Console Checkpoint — Watch the Cost Drop on Your Dashboard**

1. Go back to your **Workshop-WAF-CostOptimization** dashboard in CloudWatch
2. **Refresh the page**

**✅ You should see** the "Projected Monthly Cost" graph now has a **second data point lower than the first.** The red line (Total Cost) has dropped.

> **🎉 Cost Optimization + Sustainability achieved (compute)!** By switching one configuration flag (x86_64 → arm64), you cut compute costs by ~26%. The same code runs faster on more energy-efficient hardware. This is the Cost Optimization pillar (pay less) AND the Sustainability pillar (less energy per computation) in one change.

---

## PART 2 — Cost Optimization: Log Retention

### Step 12: 🚨 SEE THE PROBLEM — Logs That Never Expire

Let's look at your workload Lambda's log group. When a Lambda function runs, AWS automatically creates a log group for it. By default, those logs are stored **forever**.

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws logs describe-log-groups --log-group-name-prefix "/aws/lambda/workshop-waf-cost-workload" --query "logGroups[0].{LogGroup:logGroupName,Retention:retentionInDays}" --output table --region us-east-1
```

**macOS / Linux:**
```bash
aws logs describe-log-groups --log-group-name-prefix "/aws/lambda/workshop-waf-cost-workload" --query "logGroups[0].{LogGroup:logGroupName,Retention:retentionInDays}" --output table --region us-east-1
```

**✅ You should see:**

```
-------------------------------------------------------------
|                     DescribeLogGroups                      |
+------------------------------------------+----------------+
|                 LogGroup                 |   Retention    |
+------------------------------------------+----------------+
|  /aws/lambda/workshop-waf-cost-workload  |  None          |
+------------------------------------------+----------------+
```

> **🚨 "None" means "never expire."** These logs will accumulate forever, and you pay $0.03/GB/month for every GB stored. For a busy function (1M invocations/month), that's ~500 MB of new logs per month. After a year: ~6 GB stored = $0.18/month growing every month — forever.
>
> This violates both the **Cost Optimization pillar** (paying for data you'll never look at) and the **Sustainability pillar** (storing data consumes energy even when nobody reads it).

---

### Step 13: ✅ FIX THE PROBLEM — Set 7-Day Retention

For a development/testing function, 7 days of logs is more than enough. Old logs are automatically deleted.

📋 Copy and paste:

```
aws logs put-retention-policy --log-group-name /aws/lambda/workshop-waf-cost-workload --retention-in-days 7 --region us-east-1
```

**✅ No output means success.**

**Verify the change:**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws logs describe-log-groups --log-group-name-prefix "/aws/lambda/workshop-waf-cost-workload" --query "logGroups[0].retentionInDays" --output text --region us-east-1
```

**macOS / Linux:**
```bash
aws logs describe-log-groups --log-group-name-prefix "/aws/lambda/workshop-waf-cost-workload" --query "logGroups[0].retentionInDays" --output text --region us-east-1
```

**✅ You should see:** `7`

> **What just happened?** AWS will now automatically delete any log entries older than 7 days. Your log storage is now **bounded** — it can never grow beyond ~7 days × daily ingest. For a function producing ~17 MB/day, that caps storage at ~120 MB instead of growing to 6+ GB.

**✅ Console Checkpoint:**

1. Go to [https://console.aws.amazon.com/](https://console.aws.amazon.com/)
2. Search for **CloudWatch** and click it
3. Click **Log groups** in the left sidebar
4. Find `/aws/lambda/workshop-waf-cost-workload`
5. Look at the **Expire events after** column — it should now say **7 days** (not "Never expire")

---

### Step 14: ✅ VERIFY THE FIX — Watch the Total Cost Drop Again

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-waf-cost-calculator --payload file://calculator-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 calc-response3.json; Write-Output "=== COST PROJECTION (AFTER RETENTION FIX) ===" (Get-Content calc-response3.json | ConvertFrom-Json).body
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-waf-cost-calculator --payload file://calculator-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 calc-response3.json; echo "=== COST PROJECTION (AFTER RETENTION FIX) ==="; cat calc-response3.json
```

**✅ You should see the logs cost drop dramatically:**

```
LOGS:
  Retention: 7-day retention - bounded
  Projected logs cost: $0.0035/month

TOTAL PROJECTED COST: $1.2742/month
```

> **📝 Final comparison:**
>
> | Stage | Compute | Logs | Total | Savings |
> |-------|---------|------|-------|---------|
> | Before (x86_64, no retention) | $1.72 | $0.18 | **$1.90** | — |
> | After Graviton switch | $1.27 | $0.18 | **$1.45** | -24% |
> | After Graviton + 7-day retention | $1.27 | $0.004 | **$1.27** | **-33%** |

**Step 14b: ✅ Console Checkpoint — Final Dashboard View**

1. Go back to your **Workshop-WAF-CostOptimization** dashboard
2. **Refresh the page**

**✅ You should see** three data points on the "Projected Monthly Cost" graph, with a clear **downward trend:**
- First point: ~$1.90 (wasteful config)
- Second point: ~$1.45 (after Graviton)
- Third point: ~$1.27 (after Graviton + retention)

The red line drops like a staircase. **This is your feedback loop** — each architectural change produces a measurable cost reduction that you can see on your own dashboard in real time.

> **🎉 Cost Optimization + Sustainability achieved!**
>
> With two configuration changes (no code changes!), you reduced projected costs by 33%:
> - **Graviton:** 20% cheaper per ms + faster execution = ~26% compute savings. Also uses less energy per computation (Sustainability).
> - **Log retention:** Bounded storage prevents indefinite growth. Costs drop from $0.18 to $0.004 (98% reduction). Less stored data = less energy for storage (Sustainability).

---

## What You Just Did

You completed the Well-Architected Framework by applying the final two pillars:

| Pillar | Problem You Saw | Fix You Applied | How You Verified |
|--------|----------------|-----------------|------------------|
| **Cost Optimization** | x86_64 architecture (expensive compute) + logs stored forever | Switched to arm64 (Graviton) + set 7-day log retention | Projected cost dropped from $1.90 to $1.27/month on your dashboard |
| **Sustainability** | More energy per computation (x86) + indefinite data storage | Graviton uses less energy per watt + bounded log storage | Same dashboard — less compute waste, less stored data |

**Key takeaways:**
- **Graviton (arm64) is a free win** — same code, ~20% cheaper, more energy-efficient. There is almost never a reason NOT to use it for Lambda (unless you have compiled native x86 dependencies).
- **Log retention is table stakes** — never leave logs at "never expire" in production. Pick a retention period that matches your debugging needs (7 days for dev, 30–90 days for production).
- **You can't optimize what you can't measure.** The custom dashboard gave you visibility into projected costs. Without it, you'd never know the architecture switch mattered.
- **Cost Optimization and Sustainability often align** — actions that save money (less compute, less storage) also reduce environmental impact.

---

## Session 6 Complete — All Six Pillars Covered

Across Labs 6A, 6B, and 6C, you have now applied every pillar of the Well-Architected Framework:

| Pillar | Lab | What You Did |
|--------|-----|--------------|
| Security | 6A | Blocked public access to S3 (HTTP 200 → 403) |
| Reliability | 6A | Enabled versioning to recover deleted files |
| Performance Efficiency | 6B | Doubled Lambda memory → 2x faster |
| Operational Excellence | 6B | CloudWatch Alarm → email alert on crash |
| Cost Optimization | 6C | Graviton + log retention → 33% projected savings |
| Sustainability | 6C | Same changes → less energy per computation + less stored data |

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

The SAA exam tests:
- Lambda architecture choices (x86_64 vs arm64/Graviton) and their cost implications
- CloudWatch Logs retention policies and cost management
- Custom CloudWatch metrics and dashboards
- The Cost Optimization and Sustainability pillars of the Well-Architected Framework
- Right-sizing and eliminating waste

**Sample question type:** "A company wants to reduce Lambda costs without changing application code. Which configuration change provides the most immediate cost savings?"  
**Answer:** Switch the function architecture from x86_64 to arm64 (Graviton) — approximately 20% cheaper with no code changes required.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `create-function` fails with "role cannot be assumed" | IAM role hasn't propagated yet | Wait 10 seconds and try again |
| `invoke` returns an error about the payload | PowerShell is mangling the JSON | Make sure you are using `--payload file://` (from a file), not inline JSON |
| Calculator returns "An error occurred" about GetFunctionConfiguration | The `AWSLambda_ReadOnlyAccess` policy isn't attached | Re-run the `attach-role-policy` command from Step 3c |
| Dashboard shows "No data available" after running calculator | Custom metrics take 1–2 minutes to appear | Wait 2 minutes and refresh the dashboard page |
| `update-function-code` with `--architectures` fails | Possible CLI version issue | Run `aws --version` — you need AWS CLI v2.x. Update if needed. |
| `(Get-Content ... \| ConvertFrom-Json).body` shows error | The response file doesn't exist or is in wrong folder | Make sure you are in the `workshop-lab-6c` folder. Run `pwd` to confirm. |
| Dashboard cost line doesn't drop | The data points may be too close together in time | Set the dashboard time range to "Last 15 minutes" and zoom in. Each calculator run publishes a new point. |
| Retention still shows "None" after put-retention-policy | May be looking at the wrong log group | Make sure the log group name is exactly `/aws/lambda/workshop-waf-cost-workload` |

---

## Cleanup

**⚠️ Important:** Clean up all resources to avoid any charges.

### Step 1: Delete the CloudWatch Dashboard

📋 Copy and paste:

Check to see what dashboards are active:
```
aws cloudwatch list-dashboards --region us-east-1
```
Delete the relevant dashboard:

```
aws cloudwatch delete-dashboards --dashboard-names Workshop-WAF-CostOptimization --region us-east-1
```

**✅ No output means success, but you can always verify using the list dashboards command**

### Step 2: Delete the Lambda Functions

📋 Copy and paste these commands:

List which Lambdas are active:
```
aws lambda list-functions --region us-east-1
```
Delete the relevant Lambdas:

```
aws lambda delete-function --function-name workshop-waf-cost-calculator --region us-east-1
```

```
aws lambda delete-function --function-name workshop-waf-cost-workload --region us-east-1
```

**✅ Output with StatusCode: 204 means success for each.**

### Step 3: Delete the Log Groups

📋 Copy and paste these commands:

Check to see what Log Groups are active:
```
aws logs describe-log-groups --region us-east-1
```
Delete the relevant Log Groups:

```
aws logs delete-log-group --log-group-name /aws/lambda/workshop-waf-cost-workload --region us-east-1
```

```
aws logs delete-log-group --log-group-name /aws/lambda/workshop-waf-cost-calculator --region us-east-1
```

**✅ No output means success for each, but you can verify by using the list log groups command.**

### Step 4: Delete the IAM Role

📋 Copy and paste these four commands one at a time:

```
aws iam detach-role-policy --role-name workshop-waf-cost-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

```
aws iam detach-role-policy --role-name workshop-waf-cost-role --policy-arn arn:aws:iam::aws:policy/CloudWatchFullAccess
```

```
aws iam detach-role-policy --role-name workshop-waf-cost-role --policy-arn arn:aws:iam::aws:policy/AWSLambda_ReadOnlyAccess
```

```
aws iam delete-role --role-name workshop-waf-cost-role
```

**✅ No output means success for each.**

### Step 5: Delete Local Files

> **⚠️ Close VS Code first.** If VS Code still has the `workshop-lab-6c` folder open, the delete will fail — especially on Windows. Choose **File → Close Folder** or quit VS Code before running the commands below.

**Windows (PowerShell):**
```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-6c
```

**macOS / Linux:**
```bash
cd ~/Desktop
rm -rf ~/Desktop/workshop-lab-6c
```

### Step 6: Verify Cleanup

📋 Copy and paste — confirm the Lambda functions are gone:

```
aws lambda list-functions --query "Functions[?starts_with(FunctionName,'workshop-waf-cost')].FunctionName" --output text --region us-east-1
```

**✅ You should see** no output (empty) — both functions are deleted.

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** you received (copy and paste it)
4. Your **operating system** (Windows, Mac, or Linux)
