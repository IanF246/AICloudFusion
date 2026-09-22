# Lab 7B: Deploying a Lambda Function with Infrastructure as Code

**Session:** 7 — Infrastructure as Code with OpenTofu  
**Track:** Solutions Architecture  
**Difficulty:** Intermediate  
**Estimated Time:** 40–50 minutes  
**Target Cert:** AWS Solutions Architect – Associate (SAA)

---

## Overview

In Lab 7A you built the foundation: OpenTofu, a state backend, a deploy role, and a repo structure. Now you will use that foundation to **deploy a real, working Lambda function entirely in code** — its IAM role, its log group, and the function itself — and watch OpenTofu wire them together in the right order.

By the end of this lab, you will have:

1. **A repeatable workflow** — a helper script so you never have to remember which folder to run commands from
2. **A Lambda function deployed by code** — defined in `.tf` files, zipped automatically, deployed with one command
3. **An in-place update** — change one line of code, apply, and watch OpenTofu update the live function without recreating it
4. **A second commit** — your growing infrastructure saved in Git history

**The feedback loop for this lab:** In earlier sessions you created Lambda functions with a long sequence of CLI commands (create role, attach policy, wait, create function, create log group...). Here you will declare all of it in code and create it with a single command — then change it just as easily. You will *feel* the difference between imperative (step-by-step commands) and declarative (describe the end state) infrastructure.

---

## Prerequisites

- ✅ Completed **Lab 7A** — you must have:
  - The `workshop-iac` folder on your Desktop (your Git repo)
  - The state backend (S3 bucket + DynamoDB table)
  - The `workshop-tofu-deploy-role` IAM role
- ✅ AWS CLI authenticated — run `aws sts get-caller-identity` to confirm
- ✅ VS Code (recommended) and Git installed

> **⚠️ If you deleted your 7A resources:** You need to redo Lab 7A first. This lab builds directly on it.

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| AWS Lambda | Serverless compute | Always Free: 1M requests + 400,000 GB-seconds/month |
| Amazon CloudWatch Logs | Function logs | ~$0.00 ($0.50 per GB ingested — this lab generates kilobytes) |
| AWS IAM | Lambda execution role | Always Free |

**Estimated cost for this lab: $0.00**

---

## Concepts

**Resource** — in OpenTofu, a `resource` block declares one piece of infrastructure you want to exist (an IAM role, a Lambda function, a log group). You describe it; OpenTofu creates it.

**Data source** — a `data` block reads information rather than creating it. You will use a special data source called `archive_file` that **zips your code automatically** so you never have to zip it by hand.

**Resource dependencies** — OpenTofu reads your code, notices that the Lambda function needs the IAM role's ARN, and automatically creates the role *first*. You do not manually order things — OpenTofu builds a dependency graph and figures out the correct order.

**In-place update** — when you change a setting (like memory) and run apply, OpenTofu updates the existing resource rather than deleting and recreating it. The plan shows this as "1 to change" (not "1 to destroy, 1 to add").

**Provider** — a plugin that teaches OpenTofu how to talk to a system. You already use the `aws` provider. In this lab you add the `archive` provider (for zipping).

**Helper script** — a small script that wraps a command so it always runs correctly. You will write one that runs OpenTofu in the right environment folder for you.

---

## ⚠️ Reminder: How to Read Commands

Commands go in your terminal (📋 copy and paste). Files are created in **VS Code** by right-clicking the correct folder → **New File** → exact name (see Lab 7A's "How to Create Files" section if you need a refresher).

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account number | `123456789012` |

---

## Lab Steps

### Step 1: Set Your Profile and Reopen Your Project

**Step 1a: Set your AWS profile.**

**Windows (PowerShell):**
```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**macOS / Linux:**
```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**Verify.** 📋 Copy and paste:

```
aws sts get-caller-identity
```

**✅ You should see** your account ID. (If your token expired, run `aws sso login --profile <YOUR_PROFILE_NAME>`.)

**Step 1b: Open your project folder from Lab 7A in VS Code.**

**Windows (PowerShell):**
```powershell
cd ~\Desktop\workshop-iac
code .
```

**macOS / Linux:**
```bash
cd ~/Desktop/workshop-iac
code .
```

**✅ You should see** your `workshop-iac` project with the `infra/` folder and your files from Lab 7A. Keep your terminal in the `workshop-iac` folder.

---

### Step 2: Expand the Deploy Role's Permissions

In Lab 7A, your deploy role could only manage S3 and the lock table — that's all it needed. Now it needs to manage **Lambda functions, IAM roles, and log groups** too. You will update its permissions.

> **💡 Why not just give it full admin from the start?** Least privilege. You grant permissions as the need arises, so the role can never do more than the job requires. Expanding it now is exactly what you'd do on a real team when a project grows.

**Step 2a:** In VS Code, open the `tofu-permissions.json` file (in the top-level `workshop-iac` folder) you created in Lab 7A. Select everything (**Ctrl+A**) and delete it, then 📋 paste this expanded version, **replacing `<YOUR_ACCOUNT_ID>` in 4 places**:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "S3ManageWorkshopBuckets",
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::workshop-*",
                "arn:aws:s3:::workshop-*/*"
            ]
        },
        {
            "Sid": "DynamoDBStateLock",
            "Effect": "Allow",
            "Action": [
                "dynamodb:GetItem",
                "dynamodb:PutItem",
                "dynamodb:DeleteItem"
            ],
            "Resource": "arn:aws:dynamodb:us-east-1:<YOUR_ACCOUNT_ID>:table/terraform-locks"
        },
        {
            "Sid": "LambdaManage",
            "Effect": "Allow",
            "Action": "lambda:*",
            "Resource": "arn:aws:lambda:us-east-1:<YOUR_ACCOUNT_ID>:function:workshop-*"
        },
        {
            "Sid": "IAMManageLambdaRoles",
            "Effect": "Allow",
            "Action": [
                "iam:CreateRole",
                "iam:DeleteRole",
                "iam:GetRole",
                "iam:PassRole",
                "iam:TagRole",
                "iam:ListRolePolicies",
                "iam:ListAttachedRolePolicies",
                "iam:ListInstanceProfilesForRole",
                "iam:ListRoleTags",
                "iam:PutRolePolicy",
                "iam:GetRolePolicy",
                "iam:DeleteRolePolicy"
            ],
            "Resource": "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-*"
        },
        {
            "Sid": "LogsManage",
            "Effect": "Allow",
            "Action": "logs:*",
            "Resource": "arn:aws:logs:us-east-1:<YOUR_ACCOUNT_ID>:*"
        }
    ]
}
```

**Save the file (Ctrl+S).**

> **⚠️ Replace `<YOUR_ACCOUNT_ID>` in 4 places** (the DynamoDB, Lambda, IAM, and Logs ARNs). Your account ID is the 12-digit number from `aws sts get-caller-identity`.

> **What did you add?**
> - `LambdaManage` — create and manage Lambda functions named `workshop-*`
> - `IAMManageLambdaRoles` — create the execution role your Lambda runs as (the `ListInstanceProfilesForRole` and `ListRoleTags` actions are needed so OpenTofu can cleanly *delete* roles later)
> - `LogsManage` — create and manage CloudWatch log groups
> - `iam:PassRole` — lets OpenTofu hand the execution role to the Lambda service

**Step 2b:** Apply the updated permissions to the role. 📋 Copy and paste (run from the `workshop-iac` folder):

```
aws iam put-role-policy --role-name workshop-tofu-deploy-role --policy-name tofu-deploy-permissions --policy-document file://tofu-permissions.json
```

**✅ No output means success.**

**⏳ Wait 10 seconds** for the permissions to propagate.

---

## PART 1 — Create a Repeatable Workflow

In Lab 7A you had to `cd` into `infra/environments/dev` every time before running OpenTofu. That's easy to forget. Now you'll create a small **helper script** that runs OpenTofu in the right environment folder for you — so you can run commands from the project root every time.

### Step 3: Create the Helper Script

**Windows users:** create the PowerShell version. **Mac/Linux users:** create the bash version. (You only need the one for your OS.)

**Step 3a (Windows): Create `tofu.ps1`**

In VS Code, **right-click the `infra/scripts` folder** → **New File** → name it exactly `tofu.ps1` → press Enter. 📋 Paste this, then save:

```powershell
param(
    [Parameter(Mandatory = $true)][string]$Environment,
    [Parameter(Mandatory = $true)][string]$Action
)

$envPath = Join-Path $PSScriptRoot "..\environments\$Environment"

if (-not (Test-Path $envPath)) {
    Write-Host "Environment '$Environment' not found at $envPath"
    exit 1
}

Push-Location $envPath
tofu $Action
$exitCode = $LASTEXITCODE
Pop-Location
exit $exitCode
```

**Step 3b (Mac/Linux): Create `tofu.sh`**

Right-click the `infra/scripts` folder → **New File** → name it exactly `tofu.sh`. 📋 Paste this, then save:

```bash
#!/bin/bash
set -e

ENVIRONMENT="$1"
ACTION="$2"
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
ENV_PATH="$SCRIPT_DIR/../environments/$ENVIRONMENT"

if [ ! -d "$ENV_PATH" ]; then
  echo "Environment '$ENVIRONMENT' not found at $ENV_PATH"
  exit 1
fi

cd "$ENV_PATH"
tofu "$ACTION"
```

> **What does this script do?** It takes two inputs — the environment name (`dev`) and the action (`plan`, `apply`, etc.) — moves into that environment's folder, runs `tofu <action>`, then returns you to where you started. Now you run everything from the project root.

**Step 3c (Windows only): Allow scripts to run.**

By default, Windows blocks running `.ps1` scripts. Run this **once** to allow your own scripts. 📋 Copy and paste:

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser -Force
```

**✅ No output means success.**

> **What does this do?** It allows scripts you create locally to run, while still blocking unsigned scripts downloaded from the internet. This is a safe, standard developer setting.

---

## PART 2 — Define the Lambda Function in Code

### Step 4: Write the Lambda Code

Your Lambda needs some code to run. You'll put it in a `src` folder inside the dev environment.

**Step 4a:** In VS Code, **right-click the `infra/environments/dev` folder** → **New Folder** → name it exactly `src` → press Enter.

**Step 4b:** **Right-click the new `src` folder** → **New File** → name it exactly `handler.py` → press Enter. 📋 Paste this, then save:

```python
import json


def lambda_handler(event, context):
    print("Lambda invoked. Event received:", json.dumps(event))
    name = event.get("name", "world")
    message = f"Hello, {name}! This Lambda was deployed with OpenTofu."
    print(message)
    return {
        "statusCode": 200,
        "body": message
    }
```

> **What does this do?** A simple Python function that logs the event it receives and returns a greeting. Nothing fancy — the point of this lab is *deploying* it with IaC, not the code itself.

---

### Step 5: Define the Infrastructure in `main.tf`

You will replace your `main.tf` with a version that adds the `archive` provider and three new resources. Replacing the whole file (rather than editing pieces) avoids mistakes.

**Step 5a:** In VS Code, open `infra/environments/dev/main.tf`. Select everything (**Ctrl+A**), delete it, then 📋 paste this complete version and save:

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    archive = {
      source  = "hashicorp/archive"
      version = "~> 2.4"
    }
  }
}

provider "aws" {
  region = var.aws_region

  assume_role {
    role_arn = var.tofu_role_arn
  }

  default_tags {
    tags = {
      Environment = var.environment
      Project     = var.project
      ManagedBy   = "opentofu"
    }
  }
}

# ------------------------------------------------------------
# Zip the Lambda source code automatically (no manual zipping)
# ------------------------------------------------------------
data "archive_file" "lambda_zip" {
  type        = "zip"
  source_file = "${path.module}/src/handler.py"
  output_path = "${path.module}/build/handler.zip"
}

# ------------------------------------------------------------
# IAM role the Lambda runs as
# ------------------------------------------------------------
data "aws_iam_policy_document" "lambda_assume" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "lambda" {
  name               = "workshop-${var.environment}-hello-lambda"
  assume_role_policy = data.aws_iam_policy_document.lambda_assume.json
}

# ------------------------------------------------------------
# CloudWatch log group (with 7-day retention for cost control)
# ------------------------------------------------------------
resource "aws_cloudwatch_log_group" "lambda" {
  name              = "/aws/lambda/workshop-${var.environment}-hello"
  retention_in_days = 7
}

data "aws_iam_policy_document" "lambda_logs" {
  statement {
    actions   = ["logs:CreateLogStream", "logs:PutLogEvents"]
    resources = ["${aws_cloudwatch_log_group.lambda.arn}:*"]
  }
}

resource "aws_iam_role_policy" "lambda_logs" {
  name   = "cloudwatch-logs"
  role   = aws_iam_role.lambda.id
  policy = data.aws_iam_policy_document.lambda_logs.json
}

# ------------------------------------------------------------
# The Lambda function
# ------------------------------------------------------------
resource "aws_lambda_function" "hello" {
  function_name    = "workshop-${var.environment}-hello"
  role             = aws_iam_role.lambda.arn
  handler          = "handler.lambda_handler"
  runtime          = "python3.12"
  filename         = data.archive_file.lambda_zip.output_path
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256
  memory_size      = 128
  timeout          = 10

  depends_on = [
    aws_iam_role_policy.lambda_logs,
    aws_cloudwatch_log_group.lambda,
  ]
}
```

> **What does each piece do?**
>
> | Block | Purpose |
> |-------|---------|
> | `archive` provider | Added so OpenTofu can zip your code |
> | `data "archive_file"` | Zips `src/handler.py` into `build/handler.zip` automatically on every apply |
> | `aws_iam_role.lambda` | The identity the function runs as |
> | `aws_cloudwatch_log_group.lambda` | Where the function's logs go (7-day retention = cost control, from Session 6) |
> | `aws_iam_role_policy.lambda_logs` | Lets the function write to its log group |
> | `aws_lambda_function.hello` | The function itself — points at the zip and the role |
> | `source_code_hash` | Lets OpenTofu detect when your code changes so it redeploys |
> | `depends_on` | Makes sure the role policy and log group exist before the function |

> **💡 Notice the references:** `aws_lambda_function.hello` uses `aws_iam_role.lambda.arn`. OpenTofu sees this and automatically creates the role *before* the function. You never specify the order — OpenTofu works it out.

---

### Step 6: Recreate `outputs.tf`

In Lab 7A you deleted `outputs.tf`. Create it again. **Right-click the `dev` folder** → **New File** → `outputs.tf` → paste this, then save:

```hcl
output "lambda_function_name" {
  description = "Name of the deployed Lambda function"
  value       = aws_lambda_function.hello.function_name
}

output "lambda_function_arn" {
  description = "ARN of the deployed Lambda function"
  value       = aws_lambda_function.hello.arn
}
```

---

## PART 3 — Deploy and Test

### Step 7: Re-Initialize (You Added a New Provider)

Because you added the `archive` provider, you must run `tofu init` again to download it. This time, use your helper script.

**First, make sure your terminal is in the project root** (`workshop-iac`). 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
cd ~\Desktop\workshop-iac
.\infra\scripts\tofu.ps1 dev init
```

**macOS / Linux:**
```bash
cd ~/Desktop/workshop-iac
bash infra/scripts/tofu.sh dev init
```

**✅ You should see** OpenTofu install the `archive` provider and finish with `OpenTofu has been successfully initialized!`

> **💡 What just happened with the script?** It moved into `infra/environments/dev`, ran `tofu init` there, and brought you back. You ran it from the project root — no manual `cd` needed.

---

### Step 8: Preview the Plan

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
.\infra\scripts\tofu.ps1 dev plan
```

**macOS / Linux:**
```bash
bash infra/scripts/tofu.sh dev plan
```

**✅ You should see:** `Plan: 4 to add, 0 to change, 0 to destroy.`

> **The 4 resources:** the IAM role, the log group, the role policy, and the Lambda function. OpenTofu lists them in the order it will create them — role and log group first, then the policy and function that depend on them.

---

### Step 9: Apply and Invoke

**Step 9a: Deploy.** 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
.\infra\scripts\tofu.ps1 dev apply
```

**macOS / Linux:**
```bash
bash infra/scripts/tofu.sh dev apply
```

When prompted, **type `yes` and press Enter.**

**✅ You should see** the 4 resources created, ending with `Apply complete! Resources: 4 added` and your outputs (the function name and ARN).

**Step 9b: Invoke the Lambda to prove it works.**

First create a small input file. In VS Code, right-click the `dev` folder → New File → `invoke.json` → paste and save:

```json
{"name": "Solutions Architect"}
```

Now invoke it. 📋 Copy and paste (run from the `dev` folder so it finds the file):

**Windows (PowerShell):**
```powershell
cd ~\Desktop\workshop-iac\infra\environments\dev; aws lambda invoke --function-name workshop-dev-hello --payload file://invoke.json --cli-binary-format raw-in-base64-out --region us-east-1 response.json; Get-Content response.json
```

**macOS / Linux:**
```bash
cd ~/Desktop/workshop-iac/infra/environments/dev
aws lambda invoke --function-name workshop-dev-hello --payload file://invoke.json --cli-binary-format raw-in-base64-out --region us-east-1 response.json
cat response.json
```

**✅ You should see:**

```json
{"statusCode": 200, "body": "Hello, Solutions Architect! This Lambda was deployed with OpenTofu."}
```

**Step 9c: ✅ Console Checkpoint**

1. Go to [https://console.aws.amazon.com/](https://console.aws.amazon.com/) → search **Lambda**
2. Click **workshop-dev-hello**
3. Select **Configuration**. Notice the **Tags** (Environment, Project, ManagedBy=opentofu) — applied automatically
4. Scroll to **Configuration → Monitoring** or check **CloudWatch logs** — the `/aws/lambda/workshop-dev-hello` log group exists with 7-day retention

> **🎉 You deployed a complete Lambda stack — function, role, and logging — from code, with one command.** Compare this to Lab 6B where you ran six separate CLI commands and had to wait between them.

---

## PART 4 — Change Infrastructure by Changing Code

This is the heart of IaC: to change your infrastructure, you change the code and apply. OpenTofu figures out the difference.

### Step 10: Update the Lambda's Memory

**Step 10a:** In VS Code, open `infra/environments/dev/main.tf`. Find this line in the `aws_lambda_function` block:

```hcl
  memory_size      = 128
```

Change it to:

```hcl
  memory_size      = 256
```

**Save the file.**

**Step 10b: Preview the change.** 📋 Copy and paste (from the project root):

**Windows (PowerShell):**
```powershell
cd ~\Desktop\workshop-iac
.\infra\scripts\tofu.ps1 dev plan
```

**macOS / Linux:**
```bash
cd ~/Desktop/workshop-iac
bash infra/scripts/tofu.sh dev plan
```

**✅ You should see:** `Plan: 0 to add, 1 to change, 0 to destroy.`

> **💡 This is the key insight.** OpenTofu says **1 to change** — not "1 to destroy, 1 to add." It will modify the existing function *in place*. Your function keeps the same ARN; only the memory setting changes. The plan also shows the exact line changing: `memory_size = 128 -> 256`.

**Step 10c: Apply the change.** 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
.\infra\scripts\tofu.ps1 dev apply
```

**macOS / Linux:**
```bash
bash infra/scripts/tofu.sh dev apply
```

Type `yes` when prompted.

**✅ You should see:** `Apply complete! Resources: 0 added, 1 changed, 0 destroyed.`

> **🎉 You changed live infrastructure by editing one line of code.** No console clicking, no remembering which CLI flag updates memory. The code is the source of truth — change it, apply, done.

---

## PART 5 — Save Your Work

### Step 11: Commit to Git

Your infrastructure grew, so save a new snapshot. First, tell Git to ignore the build artifacts (the zip OpenTofu generates).

**Step 11a:** In VS Code, open the `.gitignore` file in your project root and add these two lines at the bottom, then save:

```
# OpenTofu build artifacts
infra/environments/*/build/
```

**Step 11b:** Commit. 📋 Copy and paste (from the project root `workshop-iac`):

**Windows (PowerShell):**
```powershell
cd ~\Desktop\workshop-iac
git add .
git commit -m "Add Lambda function (role, log group, function) via IaC"
```

**macOS / Linux:**
```bash
cd ~/Desktop/workshop-iac
git add .
git commit -m "Add Lambda function (role, log group, function) via IaC"
```

**✅ You should see** a summary of files changed.

> **💡 Notice** the `build/` zip, `.terraform` folder, and state files are NOT committed — your `.gitignore` handles that. Only your source code and `.tf` files are tracked.

---

## What You Just Did

You deployed and updated a complete Lambda stack entirely through code:

| What You Did | Why It Matters |
|--------------|----------------|
| Expanded the deploy role's permissions | Least privilege — grant access as needs grow, never more than required |
| Created a helper script | A repeatable workflow you run from the project root, every time |
| Declared a Lambda, role, and log group in code | The whole stack is defined once, deployed with one command |
| Let `archive_file` zip your code | No manual zipping — the IaC tool handles the build |
| Changed memory by editing one line | Declarative updates: change code → plan → apply. OpenTofu finds the difference |
| Committed to Git | Your infrastructure history is saved and shareable |

**Key takeaways:**
- **OpenTofu orders resources automatically** by reading references between them — you never specify the order.
- **In-place updates** mean changing a setting modifies the resource without destroying it (`1 to change`).
- **The code is the source of truth.** To change infrastructure, you change code and apply — not click in the console.

> **💡 What persists:** Your Lambda, role, and log group stay deployed — Lab 7C builds an event-driven architecture on top of this. **Do NOT delete the `workshop-iac` folder or your AWS resources** unless you are stopping the IaC track. (If you must stop, see Cleanup below.)

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

The SAA exam tests:
- Lambda execution roles and the trust relationship with `lambda.amazonaws.com`
- IAM least-privilege design (scoping actions and resources)
- CloudWatch Logs and retention
- Infrastructure as Code concepts: idempotency, plan/apply, declarative configuration

**Sample question type:** "An engineer changes a Lambda's memory setting in their IaC configuration and runs apply. What happens to the function?"  
**Answer:** It is updated in place — the same function is modified, not recreated. Its ARN and triggers are preserved.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `running scripts is disabled on this system` | PowerShell execution policy blocks `.ps1` | Run the command in Step 3c, then try again |
| `AccessDenied` on `lambda:CreateFunction` or `iam:CreateRole` | The deploy role permissions didn't update | Re-run Step 2b, wait 10 seconds, try again |
| `Error: creating Lambda Function ... InvalidParameterValueException: role` | The execution role hasn't propagated | Wait 10 seconds and run apply again |
| `tofu` not recognized inside the script | OpenTofu not on PATH | Close/reopen the terminal; verify `tofu --version` works |
| `Error acquiring the state lock` | A previous command was interrupted | Run `tofu force-unlock <LOCK_ID>` (the ID is in the error), or wait and retry |
| Plan shows the function will be replaced, not changed | You changed a property that requires replacement (like the function name) | That's expected for some fields; for memory/timeout it should say "change" |
| Invoke returns `AccessDeniedException` | Your CLI session expired | Re-run `aws sso login --profile <YOUR_PROFILE_NAME>` |

---

## Cleanup

> **⚠️ Recommended: keep everything for Lab 7C.** Lab 7C builds directly on this Lambda. Only clean up if you are stopping the IaC track here.

### If you are continuing to Lab 7C (recommended):

**No cleanup needed.** Leave your resources deployed and your `workshop-iac` folder in place.

### If you are stopping and want to remove what this lab created:

📋 Copy and paste (from the project root):

**Windows (PowerShell):**
```powershell
cd ~\Desktop\workshop-iac
.\infra\scripts\tofu.ps1 dev destroy
```

**macOS / Linux:**
```bash
cd ~/Desktop/workshop-iac
bash infra/scripts/tofu.sh dev destroy
```

Type `yes` when prompted. This removes the Lambda, role, log group, and policy.

> **💡 This does NOT remove your state backend or deploy role** (the bootstrap resources from Lab 7A). To remove those too, follow the full cleanup in Lab 7A.

---

## Help

If you get stuck, post in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran
3. The **full error message**
4. Your **operating system**
