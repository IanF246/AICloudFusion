# Lab 9C: Prevent It Next Time — Add a Smoke Test to CI

**Session:** 9 — Monitoring & Observability  
**Track:** Solutions Architecture + CI/CD  
**Difficulty:** Advanced  
**Estimated Time:** 45–55 minutes  
**Target Cert:** AWS Solutions Architect – Associate (SAA)

---

## Overview

In Labs 9A and 9B, a broken code change reached production because **the CI pipeline didn't test for it**. The pipeline ran, the code deployed, and the function started failing. You only found out because of the CloudWatch alarm — reactive, not preventive.

This lab closes the loop: you'll add a **smoke test** to your CI/CD pipeline that invokes the Lambda function after every deploy and checks that it actually works. If the response is broken, the pipeline **fails and blocks the merge** — the bad code never reaches production.

By the end of this lab, you will have:

1. **A GitHub Actions workflow** that deploys your Lambda function
2. **A post-deploy smoke test** that invokes the function and checks the response
3. **Proved it works** by pushing a good change (pipeline passes) and a bad change (pipeline fails and blocks)
4. **A CloudWatch Dashboard** that gives you a single-pane view of your function's health

**The lesson:** Monitoring tells you *when* things break. CI/CD tests prevent them from *breaking in the first place*. Defence in depth = both.

---

## Prerequisites

- ✅ Completed **Lab 8A** — you need a GitHub repo (`workshop-iac`) with OIDC trust and the `github-actions-infra` pipeline role
- ✅ Completed **Lab 9A** (understand CloudWatch alarms and Lambda deployment)
- ✅ The **`workshop-lab9-lambda-role`** IAM role must still exist (if you cleaned up Lab 9A, you need to recreate it — see Step 1d below)
- ✅ AWS CLI authenticated
- ✅ GitHub CLI authenticated (`gh auth status`)
- ✅ Git installed and configured

> **⚠️ This lab requires Session 8's GitHub + OIDC setup.** If you haven't done that, you'll need either:
> - Complete Lab 8A first (recommended), OR
> - Follow the simplified alternative in the "No OIDC? Manual Alternative" box at each relevant step

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| AWS Lambda | Function execution | Always Free (1M requests/month) |
| Amazon CloudWatch | Metrics + 1 dashboard | Always Free (3 dashboards free) |
| GitHub Actions | CI/CD pipeline execution | Free (2,000 minutes/month for private repos) |

**Estimated cost for this lab: $0.00**

---

## Concepts

**Smoke Test** — a quick, automated check that runs after deployment to verify the application actually works. It doesn't test every feature — it tests the most basic thing: "does this function run without crashing?" Like turning on a machine and checking it doesn't immediately catch fire (hence "smoke").

**Post-Deploy Verification** — the practice of automatically testing your deployment immediately after it completes. If the test fails, you know the deploy broke something and can roll back immediately — before users are affected.

**Shift Left** — the idea of catching problems earlier in the process. Instead of finding bugs in production (expensive, damages users), catch them in the pipeline (cheap, damages nobody). A smoke test shifts detection *left* — from "production alarm fires" to "pipeline blocks the merge."

**Defence in Depth** — security and reliability principle: don't rely on one safeguard. Have multiple layers. In this context: the smoke test catches bugs before deploy (prevention), AND the CloudWatch alarm catches anything that slips through (detection). Both together are stronger than either alone.

**CloudWatch Dashboard** — a customizable page showing multiple metrics in one view. Instead of checking metrics one by one, a dashboard gives you a single-pane operational view: invocations, errors, duration, all at a glance.

---

## ⚠️ Reminder: How to Read Commands

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account number | `123456789012` |
| `<YOUR_GITHUB_USERNAME>` | Your GitHub username | `janedoe` |

---

## Lab Steps

### Step 1: Set Up Your Project

**Step 1a:** Set your AWS profile:

**Windows (PowerShell):**
```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
aws sts get-caller-identity
```

**macOS / Linux:**
```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
aws sts get-caller-identity
```

**✅ You should see** your account ID. If you get an expired token error, run `aws sso login --profile <YOUR_PROFILE_NAME>`, approve in the browser, and try again.

**Step 1b:** Navigate to your `workshop-iac` project from Session 8:

**Windows (PowerShell):**
```powershell
cd ~\Desktop\workshop-iac
```

**macOS / Linux:**
```bash
cd ~/Desktop/workshop-iac
```

> **💡 If you don't have this folder:** Create a new repo for this lab. Run `mkdir ~/Desktop/workshop-lab-9c && cd ~/Desktop/workshop-lab-9c && git init`. You'll need to create a GitHub repo and push to it manually (see Lab 8A Steps 2–3).

**Step 1c:** Create a folder for the Lambda code within your project:

```
mkdir -p app
```

On Windows (PowerShell):
```powershell
New-Item -ItemType Directory -Path app -Force
```

**Step 1d:** Verify the Lambda execution role exists. 📋 Copy and paste:

```
aws iam get-role --role-name workshop-lab9-lambda-role --query "Role.RoleName" --output text
```

**✅ You should see:** `workshop-lab9-lambda-role`

> **If you get `NoSuchEntity`:** You cleaned up from Lab 9A. Recreate the role now:
>
> 1. Create a file called `lambda-trust.json` with this content:
> ```json
> {
>     "Version": "2012-10-17",
>     "Statement": [
>         {
>             "Effect": "Allow",
>             "Principal": {"Service": "lambda.amazonaws.com"},
>             "Action": "sts:AssumeRole"
>         }
>     ]
> }
> ```
> 2. Run:
> ```
> aws iam create-role --role-name workshop-lab9-lambda-role --assume-role-policy-document file://lambda-trust.json
> aws iam attach-role-policy --role-name workshop-lab9-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
> ```
> 3. Wait 10 seconds, then continue.

---

### Step 2: Create the Lambda Function Code (In Your Repo)

**Step 2a:** Create the handler file. In VS Code, right-click the `app` folder → **New File** → name it `handler.py`. 📋 Copy and paste:

```python
import json
import logging
import time

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """Workshop API endpoint."""
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

Save the file.

---

### Step 3: Create the CI/CD Workflow with Smoke Test

This is the pipeline that deploys your function AND tests it automatically.

**Step 3a:** Create the workflows directory:

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Path ".github\workflows" -Force
```

**macOS / Linux:**
```bash
mkdir -p .github/workflows
```

**Step 3b:** Create the workflow file. In VS Code, right-click `.github/workflows` → **New File** → name it `deploy-and-test.yml`. 📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>` in 2 places**:

```yaml
name: Deploy and Smoke Test

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  deploy-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<YOUR_ACCOUNT_ID>:role/github-actions-infra
          aws-region: us-east-1

      - name: Package Lambda
        run: |
          cd app
          zip -r ../function.zip handler.py

      - name: Deploy Lambda
        run: |
          # Update the function code (create it if it doesn't exist)
          aws lambda update-function-code \
            --function-name workshop-api-lab9 \
            --zip-file fileb://function.zip \
            --region us-east-1 \
            --query "LastUpdateStatus" \
            --output text || \
          aws lambda create-function \
            --function-name workshop-api-lab9 \
            --runtime python3.12 \
            --role arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-lab9-lambda-role \
            --handler handler.lambda_handler \
            --zip-file fileb://function.zip \
            --timeout 10 \
            --memory-size 128 \
            --region us-east-1

      - name: Wait for function to be active
        run: |
          echo "Waiting for function to be ready..."
          sleep 5
          echo "Function is active."

      - name: Smoke Test - Invoke and verify response
        run: |
          echo "🧪 Running smoke test..."
          
          # Invoke the function
          aws lambda invoke \
            --function-name workshop-api-lab9 \
            --region us-east-1 \
            response.json
          
          echo "Response:"
          cat response.json
          echo ""
          
          # Check for function errors
          if grep -q "errorMessage" response.json; then
            echo ""
            echo "❌ SMOKE TEST FAILED!"
            echo "The function returned an error:"
            cat response.json | python3 -m json.tool
            echo ""
            echo "This deploy would break the application."
            echo "Fix the code before merging."
            exit 1
          fi
          
          # Check for expected response
          if grep -q 'healthy' response.json; then
            echo ""
            echo "✅ SMOKE TEST PASSED!"
            echo "Function responds correctly."
          else
            echo ""
            echo "❌ SMOKE TEST FAILED!"
            echo "Unexpected response format:"
            cat response.json
            exit 1
          fi
```

**Step 3c:** Save the file.

> **What does this workflow do?**
> 1. Triggers on every push to `main` and every pull request
> 2. Authenticates to AWS using OIDC (no stored keys — from Session 8)
> 3. Packages and deploys the Lambda function
> 4. **Smoke test:** invokes the function and checks the response
>    - If the response contains `"errorMessage"` → the test **fails** (pipeline goes red)
>    - If the response contains `"status": "healthy"` → the test **passes** (pipeline goes green)
>    - Any other response → also fails (unexpected behaviour)

> **🔑 The key insight:** The smoke test catches the *exact bug* from Lab 9B. If someone pushes code that references an undefined variable, the Lambda will error when invoked, the smoke test will see `"errorMessage"` in the response, and the pipeline will fail — blocking the merge.

---

### Step 4: Give the Pipeline Role Lambda Permissions

Your `github-actions-infra` role (from Lab 8A) needs permission to deploy and invoke the Lambda function.

**Step 4a:** Create the permissions file. In VS Code, right-click the project root → **New File** → `lambda-pipeline-permissions.json`. 📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>` in 2 places**:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "LambdaDeploy",
            "Effect": "Allow",
            "Action": [
                "lambda:UpdateFunctionCode",
                "lambda:CreateFunction",
                "lambda:GetFunction",
                "lambda:InvokeFunction",
                "lambda:GetFunctionConfiguration"
            ],
            "Resource": "arn:aws:lambda:us-east-1:<YOUR_ACCOUNT_ID>:function:workshop-api-lab9"
        },
        {
            "Sid": "PassRoleToLambda",
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-lab9-lambda-role"
        }
    ]
}
```

**Replace `<YOUR_ACCOUNT_ID>` in 2 places.** Save the file.

**Step 4b:** Attach the policy to the pipeline role. 📋 Copy and paste (from the `workshop-iac` folder, where `lambda-pipeline-permissions.json` is):

```
aws iam put-role-policy --role-name github-actions-infra --policy-name lambda-deploy-permissions --policy-document file://lambda-pipeline-permissions.json
```

**✅ No output means success.**

---

### Step 5: Commit and Push — Watch the Pipeline Pass

**Step 5a:** Commit everything and push. 📋 Copy and paste:

```
git add .
git commit -m "Add Lambda deploy pipeline with smoke test"
git push
```

**Step 5b:** Watch the pipeline run. 📋 Copy and paste:

```
gh run watch
```

This shows the pipeline running in real time. You should see each step complete: checkout → configure AWS → package → deploy → smoke test.

**✅ You should see** the run complete with a green checkmark. The smoke test should show:
```
✅ SMOKE TEST PASSED!
Function responds correctly.
```

> **💡 If `gh run watch` doesn't work:** open your GitHub repo in a browser and click the **Actions** tab. You'll see the run in progress.

---

### Step 6: 🚨 Push the Bad Code — Watch the Pipeline Catch It

Now the critical test: push a broken change and prove the pipeline **blocks it**.

**Step 6a:** Create a branch for the bad change. 📋 Copy and paste:

```
git checkout -b bad-deploy
```

**Step 6b:** Edit `app/handler.py` and introduce the bug. Open the file in VS Code and **add this line** after the logging block (around line 21, before the `result = ...` line):

```python
    # Someone added a database connection but forgot to define the variable
    connection_string = database_url
```

Your file should look like this around line 20-24:
```python
    ...
    }))

    # Someone added a database connection but forgot to define the variable
    connection_string = database_url

    # Application logic
    result = {"status": "healthy", "message": "Hello from the workshop API!"}
    ...
```

Save the file.

**Step 6c:** Commit and push the bad branch:

```
git add .
git commit -m "Add database connection"
git push -u origin bad-deploy
```

**Step 6d:** Open a pull request. 📋 Copy and paste:

```
gh pr create --title "Add database connection" --body "Adding database connectivity to the API"
```

**✅ You should see** a PR URL. The pipeline will automatically run against the PR.

> **⚠️ If the pipeline doesn't trigger:** The workflow file must exist on `main` for PR triggers to work. If you skipped Step 5 or the first push failed, the workflow doesn't exist on main yet and GitHub won't run it on your PR. Go back to Step 5 and ensure the push to main succeeded.

**Step 6e:** Watch the pipeline. 📋 Copy and paste:

```
gh run watch
```

**✅ You should see** the pipeline **fail** at the smoke test step. Review in GitHub the exact error which should look like:
```
❌ SMOKE TEST FAILED!
The function returned an error:
{
    "errorMessage": "name 'database_url' is not defined",
    "errorType": "NameError",
    ...
}
This deploy would break the application.
Fix the code before merging.
```

**🎯 The pipeline caught the bug.** The bad code never reaches production. No alarm fires. No users are affected. The PR is blocked until the code is fixed.

**Step 6f: ✅ Console Checkpoint** — Open your GitHub repo → **Pull requests** tab:
1. Click on your "Add database connection" PR
2. You should see a red ❌ next to the workflow check
3. Select the PR and click "Checks" tab to see the full smoke test failure output

> **💡 What just happened:** Without the smoke test, this exact bug reached production in Lab 9B and caused a full outage. With the smoke test, it was caught before merge. That's the shift-left principle in action.

---

### Step 7: Clean Up the Bad Branch

**Step 7a:** Close the PR and go back to main:

```
gh pr close bad-deploy --delete-branch
git checkout main
```

> **💡 If `gh pr close` gives an error:** you can close it manually on GitHub (PR page → "Close pull request"), then locally run `git checkout main` and `git branch -D bad-deploy`.

---

### Step 8: Build a CloudWatch Dashboard (Operational Visibility)

Now that your function is deployed and monitored, build a dashboard that shows everything at a glance.

**Step 8a:** Create the dashboard definition. In VS Code, create a file called `dashboard.json` in the project root. 📋 Copy and paste:

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
                "title": "Invocations",
                "metrics": [
                    ["AWS/Lambda", "Invocations", "FunctionName", "workshop-api-lab9"]
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
                "title": "Errors",
                "metrics": [
                    ["AWS/Lambda", "Errors", "FunctionName", "workshop-api-lab9"]
                ],
                "period": 60,
                "stat": "Sum",
                "region": "us-east-1",
                "annotations": {
                    "horizontal": [
                        {
                            "label": "Alarm threshold",
                            "value": 1,
                            "color": "#d13212"
                        }
                    ]
                }
            }
        },
        {
            "type": "metric",
            "x": 0,
            "y": 6,
            "width": 12,
            "height": 6,
            "properties": {
                "title": "Duration (ms)",
                "metrics": [
                    ["AWS/Lambda", "Duration", "FunctionName", "workshop-api-lab9"]
                ],
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
                "title": "Error Rate (%)",
                "metrics": [
                    [{"expression": "(errors / invocations) * 100", "label": "Error Rate", "id": "rate"}],
                    ["AWS/Lambda", "Errors", "FunctionName", "workshop-api-lab9", {"id": "errors", "visible": false}],
                    ["AWS/Lambda", "Invocations", "FunctionName", "workshop-api-lab9", {"id": "invocations", "visible": false}]
                ],
                "period": 60,
                "stat": "Sum",
                "region": "us-east-1"
            }
        }
    ]
}
```

**Step 8b:** Create the dashboard. 📋 Copy and paste (from the project root, where `dashboard.json` is):

```
aws cloudwatch put-dashboard --dashboard-name workshop-lab9-overview --dashboard-body file://dashboard.json --region us-east-1
```

**✅ You should see** JSON with `"DashboardValidationMessages": []` (empty = no errors).

**Step 8c: ✅ Console Checkpoint** — In the CloudWatch console:
1. In the left menu, click **Dashboards**
2. Click **workshop-lab9-overview**
3. You should see four panels: Invocations, Errors (with a red threshold line at 1), Duration, and Error Rate
4. The Errors panel should show the spike from Lab 9B (if that data is still in range) and zeros after

> **💡 What you've built:** A single screen that tells you everything about your function's health. In a real team, this dashboard would be on a monitor in the office or linked in Slack. Anyone can glance at it and know if the service is healthy.

---

## What You Just Did

| What You Built | Why It Matters |
|---------------|----------------|
| CI/CD pipeline with deploy | Automation — no manual deploys |
| Post-deploy smoke test | Catches broken code *before* it affects users |
| Proved the pipeline blocks a bad deploy | The exact bug from Lab 9B is now prevented |
| CloudWatch Dashboard | Single-pane operational visibility |

**The defence-in-depth story across Session 9:**

| Layer | What It Does | When It Catches Problems |
|-------|-------------|-------------------------|
| **Smoke test in CI** (this lab) | Tests after deploy, blocks bad merges | *Before* users are affected |
| **CloudWatch Alarm** (Lab 9A) | Alerts when errors occur in production | *Seconds* after problems start |
| **CloudWatch Logs + Insights** (Lab 9B) | Provides diagnostic detail | *During* investigation |
| **Dashboard** (this lab) | Ongoing operational visibility | *Always* — shows trends and current state |

No single layer is enough. Together, they give you prevention, detection, diagnosis, and visibility.

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

The SAA exam tests:
- CI/CD pipeline design and best practices
- Post-deployment verification as an operational practice
- CloudWatch Dashboards for operational visibility
- Defence in depth as an architectural principle
- Operational Excellence pillar: automated testing, monitoring, incident response

**Sample question type:** "A development team deploys code changes that occasionally break their Lambda function. How should they prevent broken deployments from reaching production?"  
**Answer:** Add a post-deployment smoke test to the CI/CD pipeline that invokes the function and verifies a healthy response. If the test fails, the pipeline blocks the code from being merged, preventing the broken code from reaching production.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| Pipeline fails at "Configure AWS credentials" | OIDC trust policy doesn't match your repo | Check the `sub` claim in your pipeline trust policy (Lab 8A Step 5–6). Run `gh api "repos/<username>/workshop-iac/actions/oidc/customization/sub"` to find the correct prefix |
| "Not authorized to perform lambda:UpdateFunctionCode" | Pipeline role is missing the Lambda permissions | Re-run Step 4b to attach the `lambda-deploy-permissions` policy |
| "is not authorized to perform: iam:PassRole" | Pipeline role can't pass the Lambda execution role | Ensure `lambda-pipeline-permissions.json` includes the `PassRoleToLambda` statement and the ARN is correct |
| "No such file or directory: function.zip" | The `cd app` or zip step failed | Check that `app/handler.py` exists and is committed (not just saved locally — you need `git add` + `git commit` + `git push`) |
| "Function not found" on create-function | The `workshop-lab9-lambda-role` doesn't exist | You cleaned up from Lab 9A. Follow Step 1d to recreate the Lambda execution role |
| Smoke test passes even with broken code | The function wasn't updated (cached version) | The wait step should handle this; if not, increase the sleep duration in the workflow |
| `gh pr create` fails | Not on a branch, or repo not connected | Ensure you're on the `bad-deploy` branch and pushed it (`git push -u origin bad-deploy`) |
| Pipeline doesn't trigger on PR | Workflow file not on `main` yet | The workflow file must exist on `main` for PR triggers to work. Push the workflow to main first (Step 5), then create the PR |
| Dashboard shows "No data available" | Function hasn't been invoked recently | Invoke the function a few times to generate fresh data points |

---

## Cleanup

**Step 1:** Delete the CloudWatch Dashboard:
```
aws cloudwatch delete-dashboards --dashboard-names workshop-lab9-overview --region us-east-1
```

**Step 2:** Remove the Lambda deploy permissions from the pipeline role:
```
aws iam delete-role-policy --role-name github-actions-infra --policy-name lambda-deploy-permissions
```

**Step 3:** Delete the Lambda function:
```
aws lambda delete-function --function-name workshop-api-lab9 --region us-east-1
```

**Step 4:** Delete the CloudWatch Log Group:
```
aws logs delete-log-group --log-group-name /aws/lambda/workshop-api-lab9 --region us-east-1
```

**Step 5:** Delete the Lambda execution role:
```
aws iam detach-role-policy --role-name workshop-lab9-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name workshop-lab9-lambda-role
```

**Step 6:** Delete the alarm and SNS topic (if created in Lab 9A):
```
aws cloudwatch delete-alarms --alarm-names workshop-lab9-errors --region us-east-1
aws sns delete-topic --topic-arn arn:aws:sns:us-east-1:<YOUR_ACCOUNT_ID>:workshop-lab9-alerts --region us-east-1
```

**Step 7:** Remove the local files added to your repo:
```
git checkout main
git branch -D bad-deploy
```

> **💡 Keep the workflow file** if you want to continue using the pipeline for other work. It's a portfolio piece — a working CI/CD pipeline with deployment and automated testing.

---

## Help

If you get stuck, post in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** you received
4. Your **operating system** (Windows / Mac / Linux)
