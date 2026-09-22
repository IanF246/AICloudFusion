# Lab 7C: Event-Driven Architecture with Reusable Modules

**Session:** 7 — Infrastructure as Code with OpenTofu  
**Track:** Solutions Architecture  
**Difficulty:** Advanced  
**Estimated Time:** 45–55 minutes  
**Target Cert:** AWS Solutions Architect – Associate (SAA)

---

## Overview

In Lab 7B you deployed a single Lambda function. Now you will do two professional things real teams do:

1. **Refactor your Lambda into a reusable module** — packaging the function, role, and log group into one building block you can reuse anywhere. This finally uses the `modules/` folder you scaffolded in Lab 7A.
2. **Build an event-driven architecture** — wire an S3 bucket so that **uploading a file automatically triggers your Lambda.** No manual invoke; the upload itself fires the function.

By the end you will have a complete, wired system defined entirely in code, and you'll watch a file upload trigger your function in real time.

**The feedback loop for this lab:** You will deploy the wired architecture, **upload a file to S3**, then check the Lambda's logs and *see your own function ran automatically* in response to the upload. That observable "it fired on its own" moment is the payoff — event-driven infrastructure, built and proven from code.

---

## Prerequisites

- ✅ Completed **Lab 7B** — you must have:
  - The `workshop-iac` repo with the dev environment and helper script
  - The state backend and `workshop-tofu-deploy-role` (with the expanded permissions from 7B)
  - The Lambda from 7B deployed (or at least the code in place)
- ✅ AWS CLI authenticated (`aws sts get-caller-identity`)
- ✅ VS Code and Git

> **⚠️ This lab builds directly on 7B.** If you destroyed your 7B resources, that's okay — you'll redeploy here. But you need the `workshop-iac` folder and the deploy role intact.

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| AWS Lambda | Serverless compute | Always Free: 1M requests/month |
| Amazon S3 | Upload bucket | $0.023 per GB/month |
| Amazon CloudWatch Logs | Function logs | Free within 5 GB/month |
| AWS IAM | Lambda execution role | Always Free |

**Estimated cost for this lab: $0.00**

---

## Concepts

**Module** — a reusable package of OpenTofu resources. Instead of copy-pasting the Lambda + role + log group into every environment, you define them once in a module, then *call* that module wherever you need a Lambda. Think of it like a function in programming: define once, call many times with different inputs.

**Module inputs and outputs** — a module declares `variable` blocks (its inputs) and `output` blocks (the values it gives back, like the function ARN). The calling code passes inputs and reads outputs — just like function parameters and return values.

**Event-driven architecture** — instead of something running on a schedule or being invoked manually, a resource reacts to an *event*. Here, the event is "a file was uploaded to S3," and the reaction is "run the Lambda."

**`aws_lambda_permission`** — by default, other AWS services cannot invoke your Lambda. This resource grants S3 permission to invoke it.

**`aws_s3_bucket_notification`** — configures the bucket to send an event to your Lambda whenever a matching action (like an object upload) happens.

**`force_destroy`** — by default S3 refuses to delete a bucket that still contains files. Setting `force_destroy = true` lets OpenTofu empty and delete the bucket during `destroy` — important for clean teardown in a lab.

---

## ⚠️ Reminder: How to Read Commands

Files are created in **VS Code** (right-click the correct folder → New File → exact name). Commands go in your terminal. Run helper-script commands from the project root (`workshop-iac`).

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name | `AdministratorAccess-123456789012` |
| `<INITIALS>` | Your first and last name initials | `if` |

---

### Step 1: Set Your AWS Profile and Reopen Your Project

Before running any AWS commands, set your default profile so you do not have to type it on every command.

**Windows (PowerShell):**

📋 Copy and paste this command, **replacing `<YOUR_PROFILE_NAME>`** with your actual profile name from Lab 1A:

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**macOS / Linux:**

```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**Verify it works.** 📋 Copy and paste this and press Enter:

```
aws sts get-caller-identity
```

**✅ You should see** your account ID and role. If you get an error about an expired token, run `aws sso login --profile <YOUR_PROFILE_NAME>` first, then set the profile again.

**Step 1b:** Open your project and keep your terminal there:

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

> **💡 Continuity from 7B:** Your helper script (`infra/scripts/tofu.ps1` or `tofu.sh`) is still there from Lab 7B, and on Windows the script-execution setting you applied in 7B still holds — so you can use the script right away in this lab.

---

## PART 1 — Build a Reusable Lambda Module

You will package the Lambda + role + log group into a module under `infra/modules/lambda/`. A module has the same kinds of files as an environment, but it declares **inputs** (variables) instead of hard-coding values.

### Step 2: Create the Module Files

**Step 2a: Create `variables.tf`** — the module's inputs.

In VS Code, **right-click `infra/modules/lambda`** → **New File** → `variables.tf` → paste and save:

```hcl
variable "function_name" {
  description = "Name of the Lambda function"
  type        = string
}

variable "source_file" {
  description = "Path to the Lambda source .py file"
  type        = string
}

variable "handler" {
  description = "Lambda handler (file.function)"
  type        = string
  default     = "handler.lambda_handler"
}

variable "memory_size" {
  description = "Memory in MB"
  type        = number
  default     = 128
}

variable "retention_days" {
  description = "CloudWatch log retention in days"
  type        = number
  default     = 7
}
```

**Step 2b: Create `main.tf`** — the module's resources.

Right-click `infra/modules/lambda` → **New File** → `main.tf` → paste and save:

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

data "archive_file" "zip" {
  type        = "zip"
  source_file = var.source_file
  output_path = "${path.module}/build/${var.function_name}.zip"
}

data "aws_iam_policy_document" "assume" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "this" {
  name               = "${var.function_name}-role"
  assume_role_policy = data.aws_iam_policy_document.assume.json
}

resource "aws_cloudwatch_log_group" "this" {
  name              = "/aws/lambda/${var.function_name}"
  retention_in_days = var.retention_days
}

data "aws_iam_policy_document" "logs" {
  statement {
    actions   = ["logs:CreateLogStream", "logs:PutLogEvents"]
    resources = ["${aws_cloudwatch_log_group.this.arn}:*"]
  }
}

resource "aws_iam_role_policy" "logs" {
  name   = "cloudwatch-logs"
  role   = aws_iam_role.this.id
  policy = data.aws_iam_policy_document.logs.json
}

resource "aws_lambda_function" "this" {
  function_name    = var.function_name
  role             = aws_iam_role.this.arn
  handler          = var.handler
  runtime          = "python3.12"
  filename         = data.archive_file.zip.output_path
  source_code_hash = data.archive_file.zip.output_base64sha256
  memory_size      = var.memory_size
  timeout          = 10

  depends_on = [
    aws_iam_role_policy.logs,
    aws_cloudwatch_log_group.this,
  ]
}
```

**Step 2c: Create `outputs.tf`** — what the module returns.

Right-click `infra/modules/lambda` → **New File** → `outputs.tf` → paste and save:

```hcl
output "function_name" {
  value = aws_lambda_function.this.function_name
}

output "function_arn" {
  value = aws_lambda_function.this.arn
}
```

> **What did you just build?** A self-contained "Lambda" building block. It takes a name, a source file, and optional memory/retention, and produces a fully working function with its role and logging. Notice it uses `var.function_name` everywhere instead of a hard-coded name — that's what makes it reusable.

---

## PART 2 — Compose the Event-Driven Architecture

Now you'll rewrite the dev environment to **call the module** and wire an S3 bucket to trigger it.

### Step 3: Update the Lambda Code to Handle S3 Events

Open `infra/environments/dev/src/handler.py`, select all (**Ctrl+A**), delete, and 📋 paste this version, then save:

```python
import json


def lambda_handler(event, context):
    print("Lambda invoked. Event received:", json.dumps(event))

    # If triggered by an S3 upload, report the file.
    records = event.get("Records", [])
    if records:
        for record in records:
            bucket = record["s3"]["bucket"]["name"]
            key = record["s3"]["object"]["key"]
            print(f"New file uploaded to S3: bucket={bucket}, key={key}")
        return {"statusCode": 200, "body": f"Processed {len(records)} S3 event(s)."}

    # Otherwise treat it as a direct invoke.
    name = event.get("name", "world")
    message = f"Hello, {name}! This Lambda was deployed with OpenTofu."
    print(message)
    return {"statusCode": 200, "body": message}
```

> **What changed?** The function now detects S3 events. When S3 sends it an upload event, it logs the bucket and file name. This is how you'll *see* the trigger working.

---

### Step 4: Rewrite `main.tf` to Use the Module + S3 Trigger

Open `infra/environments/dev/main.tf`, select all (**Ctrl+A**), delete, and 📋 paste this complete version, then save:

>[!WARNING]
>Be sure to change the <INITIALS> placeholder in the code.

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
# Lambda function (via our reusable module)
# ------------------------------------------------------------
module "processor" {
  source = "../../modules/lambda"

  function_name = "workshop-${var.environment}-processor"
  source_file   = "${path.root}/src/handler.py"
  memory_size   = 128
  retention_days = 7
}

# ------------------------------------------------------------
# S3 bucket that triggers the Lambda on upload
# ------------------------------------------------------------
resource "aws_s3_bucket" "uploads" {
  bucket        = "workshop-${var.environment}-uploads-<INITIALS>"
  force_destroy = true
}

# Allow S3 to invoke the Lambda
resource "aws_lambda_permission" "allow_s3" {
  statement_id  = "AllowS3Invoke"
  action        = "lambda:InvokeFunction"
  function_name = module.processor.function_name
  principal     = "s3.amazonaws.com"
  source_arn    = aws_s3_bucket.uploads.arn
}

# Wire the bucket to fire the Lambda on every new object
resource "aws_s3_bucket_notification" "uploads" {
  bucket = aws_s3_bucket.uploads.id

  lambda_function {
    lambda_function_arn = module.processor.function_arn
    events              = ["s3:ObjectCreated:*"]
  }

  depends_on = [aws_lambda_permission.allow_s3]
}
```

> **What does each piece do?**
>
> | Block | Purpose |
> |-------|---------|
> | `module "processor"` | Calls your Lambda module — passes a name and the source file, gets back a working function |
> | `aws_s3_bucket.uploads` | The bucket users upload to (`force_destroy` lets you tear it down even with files in it) |
> | `aws_lambda_permission.allow_s3` | Grants S3 permission to invoke the function |
> | `aws_s3_bucket_notification.uploads` | The wiring: "on every object upload, invoke the Lambda" |
> | `module.processor.function_arn` | Reads an **output** from your module — this is how the bucket finds the function |
> | `depends_on` | Ensures the permission exists before the notification is created |

> **💡 Notice what you did NOT do this lab: expand the deploy role.** In 7B you had to widen the role's permissions before deploying a Lambda. This lab adds a whole new bucket, a notification, and a cross-service permission — yet no permission step. That's because the scopes you set in 7B (`workshop-*` for S3, Lambda, IAM, and logs) were deliberately broad enough to cover future `workshop-` resources. Scoping to a naming prefix instead of one exact resource is a common real-world balance: tight enough to be least-privilege, loose enough that routine growth doesn't need a policy change every time.

---

### Step 5: Update `outputs.tf`

Open `infra/environments/dev/outputs.tf`, replace its contents with this, and save:

```hcl
output "lambda_function_name" {
  description = "Name of the deployed Lambda function"
  value       = module.processor.function_name
}

output "uploads_bucket" {
  description = "Name of the S3 bucket that triggers the Lambda"
  value       = aws_s3_bucket.uploads.bucket
}
```

---

## PART 3 — Deploy and Watch the Event Fire

### Step 6: Re-Initialize (You Added a Module)

> **💡 Starting from a fresh terminal?** Two things don't carry between terminals. On **Windows**, if you didn't make the PATH change permanent in 7A, re-add OpenTofu: `$env:Path += ";C:\OpenTofu"`. And on **any OS**, re-set your AWS profile from Step 1 (`$env:AWS_PROFILE="..."` on Windows, `export AWS_PROFILE="..."` on Mac/Linux). If you've kept the same terminal since Step 1, skip this.

Whenever you add a module, you must run `tofu init` again so OpenTofu loads it. 📋 From the project root:

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

**✅ You should see** OpenTofu initialize the `module.processor` module and finish successfully.

---

### Step 7: Plan and Apply

**Step 7a: Plan.** 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
.\infra\scripts\tofu.ps1 dev plan
```

**macOS / Linux:**
```bash
bash infra/scripts/tofu.sh dev plan
```

> **💡 If you completed 7B, expect both `add` and `destroy` lines — this is normal.** OpenTofu will remove your old standalone `workshop-dev-hello` function from 7B and create the new module-based `workshop-dev-processor` plus the S3 resources. You are *refactoring*: replacing the hand-written function with a modular, event-driven one. Don't worry about the exact counts — as long as the plan succeeds and lists the new `module.processor` resources and the S3 bucket/notification, you're on track.

> **💡 Why a rebuild and not an in-place update?** Two things changed at once: the function **moved into a module** (its address went from `aws_lambda_function.hello` to `module.processor.aws_lambda_function.this`) *and* it was **renamed** (`workshop-dev-hello` → `workshop-dev-processor`). OpenTofu can't update across a rename — a new function name means a new function — so it destroys the old and creates the new. In production, when you're *only* relocating a resource into a module (same name, same settings), you avoid the destroy/recreate with a **`moved` block** or `tofu state mv`, which tells OpenTofu "this is the same resource, just at a new address." Here the rename makes a rebuild unavoidable — which is fine for a dev function with no data to lose.

**Step 7b: Apply.** 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
.\infra\scripts\tofu.ps1 dev apply
```

**macOS / Linux:**
```bash
bash infra/scripts/tofu.sh dev apply
```

Type `yes` when prompted.

**✅ You should see** `Apply complete!` with outputs showing your function name and the uploads bucket name.

> **📝 Write down the `uploads_bucket` value** — you'll upload to it next.

---

### Step 8: 🎯 Trigger the Lambda by Uploading a File

This is the moment everything has been building toward.

**Step 8a: Create a test file and upload it.** 📋 Copy and paste, **replacing `<INITIALS>`**:

**Windows (PowerShell):**
```powershell
"This file should trigger my Lambda." | Set-Content -Path "$env:TEMP\test-upload.txt"; aws s3 cp "$env:TEMP\test-upload.txt" s3://workshop-dev-uploads-<INITIALS>/test-upload.txt
```

**macOS / Linux:**
```bash
echo "This file should trigger my Lambda." > /tmp/test-upload.txt; aws s3 cp /tmp/test-upload.txt s3://workshop-dev-uploads-<INITIALS>/test-upload.txt
```

**✅ You should see** an `upload:` confirmation.

**Step 8b: Wait ~10 seconds**, then check the Lambda's logs to prove it ran. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
$stream = aws logs describe-log-streams --log-group-name "/aws/lambda/workshop-dev-processor" --order-by LastEventTime --descending --max-items 1 --query "logStreams[0].logStreamName" --output text --region us-east-1
```

```
aws logs get-log-events --log-group-name "/aws/lambda/workshop-dev-processor" --log-stream-name $stream --query "events[].message" --output text --region us-east-1
```

**macOS / Linux:**
```bash
STREAM=$(aws logs describe-log-streams --log-group-name "/aws/lambda/workshop-dev-processor" --order-by LastEventTime --descending --max-items 1 --query "logStreams[0].logStreamName" --output text --region us-east-1)

aws logs get-log-events --log-group-name "/aws/lambda/workshop-dev-processor" --log-stream-name "$STREAM" --query "events[].message" --output text --region us-east-1
```

**✅ You should see** a log line like:

```
New file uploaded to S3: bucket=workshop-dev-uploads-<INITIALS>, key=test-upload.txt
```

> **🎉 Your Lambda fired automatically!** You never invoked it — uploading the file did. That is event-driven architecture: a file lands in S3, S3 notifies Lambda, your code runs. And you built the entire wiring in code.

**Step 8c: ✅ Console Checkpoint**

1. AWS Console → **Lambda** → **workshop-dev-processor** → **Monitor** tab → you should see an invocation spike
2. AWS Console → **S3** → your uploads bucket → your file is there
3. AWS Console → **IAM** → **Access Management** → **Roles** tab → you should see the function's role is named `workshop-dev-processor-role` — created by your **module**, not hand-written

---

## PART 4 — Tear Down and Save

### Step 9: Destroy the Whole Architecture with One Command

Watch how IaC tears down a multi-resource, wired system in the correct reverse order — automatically.

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

Type `yes` when prompted.

**✅ You should see** OpenTofu remove the notification, permission, bucket (emptying it first thanks to `force_destroy`), function, role, and log group — ending with `Destroy complete!`

> **🎉 One command tore down an entire event-driven architecture** — bucket, function, role, logging, and all the wiring — in the correct dependency order. Imagine doing that by hand in the console without forgetting something.

---

### Step 10: Commit Your Work

📋 Copy and paste. First make sure module build artifacts are ignored:

**Step 10a:** Open `.gitignore` and confirm it includes these lines (add the second if missing), then save:

```
infra/environments/*/build/
infra/modules/*/build/
```

**Step 10b:** Commit. 📋 Copy and paste (from the project root):

```
git add .
git commit -m "Add Lambda module + event-driven S3 trigger architecture"
```

**✅ You should see** a summary of files changed.

---

## What You Just Did

You built and tore down a complete event-driven architecture, defined entirely in code:

| What You Did | Why It Matters |
|--------------|----------------|
| Packaged the Lambda into a reusable module | Define once, reuse everywhere — no copy-paste across environments |
| Used module inputs and outputs | Clean interfaces between building blocks, like function parameters and return values |
| Wired S3 → Lambda with a notification | Event-driven design — resources react to events, not manual triggers |
| Granted cross-service permission | `aws_lambda_permission` lets S3 invoke your function securely |
| Proved the trigger by uploading a file | Observed your function run automatically — the event-driven payoff |
| Tore it all down with one command | OpenTofu handles dependency order and bucket emptying for you |

**Key takeaways:**
- **Modules make infrastructure reusable and consistent** — the same building block deploys identically everywhere.
- **Event-driven architectures are wiring** — a producer (S3), a permission, and a notification. OpenTofu makes that wiring explicit and repeatable.
- **OpenTofu manages dependency order on both create and destroy** — you describe relationships; it figures out the sequence.

> **💡 Where's staging, prod, and policies?** Back in Lab 7A you scaffolded `environments/staging`, `environments/prod`, and `policies/` — and this workshop only ever deployed to `dev`. That's intentional: the whole point of the structure is that **promoting to another environment is just a copy**. You'd copy `dev/`'s files into `staging/`/`prod/`, change `backend.tf`'s state `key` and the values in `terraform.tfvars` (names, sizes), and call the *same* `modules/lambda` — identical code, different inputs. The `policies/` folder is where policy-as-code (OPA/Sentinel rules like "no public S3 buckets") would live to block bad deployments before apply. You won't fill those in here, but your repo is already shaped for them — that's what "professional structure" buys you.

> **💡 What persists:** Your `workshop-iac` repo and the state backend / deploy role remain. You destroyed the *application* resources in Step 9, but your foundation is intact for future work and Session 8 (CI/CD).

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

The SAA exam tests:
- Event-driven architectures (S3 event notifications invoking Lambda)
- Resource-based policies and cross-service permissions (`aws_lambda_permission`)
- Infrastructure as Code modularity and reuse
- Dependency management and safe teardown

**Sample question type:** "A company wants a Lambda function to process every file uploaded to an S3 bucket. What configuration is required?"  
**Answer:** An S3 event notification for `s3:ObjectCreated:*` targeting the Lambda, plus a Lambda permission allowing the S3 service principal to invoke the function.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `Module not installed` when planning | You added the module but didn't re-init | Run the `init` command in Step 6 |
| Upload succeeds but no log appears | Logs can take 10–30 seconds; or the trigger isn't wired | Wait and re-run the log command; confirm `tofu apply` created the notification |
| `Error putting S3 notification: ... not authorized to perform: lambda:InvokeFunction` | The `aws_lambda_permission` wasn't created first | The `depends_on` handles this; re-run `apply` |
| `BucketAlreadyExists` | The uploads bucket name is taken | The name ends in your `<INITIALS>`, which may not be globally unique — change that suffix in `main.tf` (add more characters, e.g. a number), then re-run `apply` |
| `Error deleting S3 Bucket: BucketNotEmpty` | `force_destroy` missing | Confirm `force_destroy = true` is on the bucket, re-apply, then destroy |
| `Error acquiring the state lock` | A previous command was interrupted | Run `tofu force-unlock <LOCK_ID>` (ID in the error) and retry |

---

## Cleanup

If you ran **Step 9 (destroy)**, your application resources are already gone. 

**To keep building (recommended):** leave the `workshop-iac` folder and your state backend / deploy role in place for Session 8.

**To remove the foundation entirely** (only if you are done with the IaC track): follow the full cleanup in **Lab 7A** (delete the state bucket, the `terraform-locks` table, and the `workshop-tofu-deploy-role`), then delete the local `workshop-iac` folder.

---

## Help

If you get stuck, post in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran
3. The **full error message**
4. Your **operating system**
