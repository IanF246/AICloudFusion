# Lab 8B: A CI/CD Pipeline — Plan on Pull Request, Apply on Merge

**Session:** 8 — CI/CD for Infrastructure  
**Track:** Solutions Architecture + IaC  
**Difficulty:** Intermediate  
**Estimated Time:** 40–50 minutes  
**Target Cert:** AWS Solutions Architect – Associate (SAA)

---

## Overview

In Lab 8A you set up keyless OIDC trust between GitHub and AWS. Now you'll build the actual **pipeline**: a GitHub Actions workflow that

- runs `tofu plan` automatically on every **pull request** (so changes are previewed and reviewed before merging), and
- runs `tofu apply` automatically when a PR is **merged to `main`** (so approved changes deploy themselves).

This is the **GitOps** workflow used by real teams: your Git repository becomes the single source of truth, and the pipeline keeps AWS matching it.

By the end you will have opened a pull request, watched a plan run automatically, merged it, and watched your infrastructure deploy — without running a single command on your own machine.

**The feedback loop for this lab:** Previously, changes were applied from someone's laptop with no review. Now you'll *see* every change previewed in the PR before it can merge, and *see* it deploy automatically on merge — the reviewed, automated GitOps cycle.

---

## Prerequisites

- ✅ Completed **Lab 8A** — you must have:
  - Your `workshop-iac` repo pushed to GitHub
  - The GitHub OIDC provider and the `github-actions-infra` pipeline role
  - The state backend and `workshop-tofu-deploy-role` from Session 7
- ✅ Your **pipeline role ARN** from Lab 8A (`arn:aws:iam::<ACCOUNT_ID>:role/github-actions-infra`)

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| GitHub Actions | Pipeline runner | Free minutes (generous free tier) |
| AWS Lambda / S3 / IAM / Logs | Resources the pipeline deploys | Within Always Free / 12-month free tier |

**Estimated cost for this lab: $0.00**

---

## Concepts

**Workflow** — a YAML file in `.github/workflows/` describing automation. It has triggers (`on:`), and one or more jobs made of steps.

**Trigger (`on:`)** — the events that start the workflow. You'll use `pull_request` (someone opens/updates a PR) and `push` to `main` (a PR is merged).

**Pull request (PR)** — a proposal to merge changes from one branch into another (here, into `main`). It's where code review happens. Running `plan` on the PR lets reviewers see exactly what will change in AWS before approving.

**Branch** — a parallel line of work. You make changes on a branch, open a PR from it into `main`, and merge when approved. `main` represents "what is deployed."

**`permissions: id-token: write`** — the line that lets a workflow request an OIDC token. Without it, OIDC authentication to AWS fails. It's the most commonly forgotten piece.

**Conditional steps (`if:`)** — a step can run only for certain events. You'll run `plan` only on pull requests and `apply` only on pushes to `main`.

---

## ⚠️ Reminder: How to Read Commands

Files are created in **VS Code**. Commands go in your terminal. Some steps happen in the **GitHub website** (opening and merging PRs) — those are click-by-click.

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account number | `123456789012` |

---

## Lab Steps

### Step 1: Open Your Project

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

Make sure your work is committed and pushed. 📋 Copy and paste:

```
git status
```

**✅ You should see** "working tree clean." If not, `git add . ; git commit -m "sync" ; git push` (use `;` on PowerShell, `&&` on Mac/Linux).

---

### Step 2: Set a Simple Resource for the Pipeline to Manage

So the pipeline has something predictable to deploy, you'll set your dev environment to manage one simple S3 bucket. (This replaces whatever was left from Session 7.)

**Step 2a:** In VS Code, open `infra/environments/dev/main.tf`. Select all (**Ctrl+A**), delete, and 📋 paste this, then save:

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
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

data "aws_caller_identity" "current" {}

resource "aws_s3_bucket" "demo" {
  bucket = "workshop-${var.environment}-cicd-demo-${data.aws_caller_identity.current.account_id}"
}
```

**Step 2b:** If your `outputs.tf` still references old resources (from 7C), replace its contents with this and save:

```hcl
output "demo_bucket" {
  value = aws_s3_bucket.demo.bucket
}
```

> **💡 Notice the provider still uses `assume_role` with `var.tofu_role_arn`** — exactly as in Session 7. Your code does not change for CI/CD. The only difference is *who* runs it: locally it's you, in the pipeline it's GitHub assuming the pipeline role (which assumes this same deploy role).

---

### Step 3: Write the Pipeline Workflow

**Step 3a:** In VS Code, right-click the top-level `workshop-iac` folder → **New Folder** → `.github` → Enter. Then right-click `.github` → **New Folder** → `workflows`. Then right-click `workflows` → **New File** → `infra.yml`.

> **⚠️ Exact names matter:** the folder must be `.github` (with the leading dot) then `workflows`, and GitHub only runs workflow files from that exact path.

**Step 3b:** 📋 Paste this into `infra.yml`, **replacing `<YOUR_ACCOUNT_ID>`**, then save:

```yaml
name: Infrastructure CI/CD

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  id-token: write   # required for OIDC authentication
  contents: read

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infra/environments/dev
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<YOUR_ACCOUNT_ID>:role/github-actions-infra
          aws-region: us-east-1

      - name: Setup OpenTofu
        uses: opentofu/setup-opentofu@v1

      - name: Tofu Init
        run: tofu init

      - name: Tofu Plan
        if: github.event_name == 'pull_request'
        run: tofu plan

      - name: Tofu Apply
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        run: tofu apply -auto-approve
```

**Replace `<YOUR_ACCOUNT_ID>`** with your account ID (1 place). Save.

> **What does this workflow do?**
>
> | Part | Meaning |
> |------|---------|
> | `on: pull_request` / `push` | Run on PRs to main, and on merges (pushes) to main |
> | `permissions: id-token: write` | **Required** so the job can request an OIDC token — forget this and OIDC fails |
> | `configure-aws-credentials` | Assumes your `github-actions-infra` role via OIDC — no stored keys |
> | `setup-opentofu` | Installs OpenTofu on the runner |
> | `Tofu Plan` with `if: pull_request` | Preview only, runs on PRs |
> | `Tofu Apply` with `if: push` to main | Deploys, runs only when something merges to main |

---

### Step 4: Push the Workflow — Your First Automated Deploy

Committing this workflow to `main` is itself a push to `main`, so the pipeline will run its **apply** job and deploy your demo bucket automatically.

📋 Copy and paste (from the project root):

```
git add .
git commit -m "Add CI/CD pipeline: plan on PR, apply on merge"
git push
```

**Step 4b: Watch it run.**

1. Go to your repo on GitHub → click the **Actions** tab
2. You'll see a workflow run named after your commit, in progress (yellow dot)
3. Click it → click the **terraform** job to watch the steps run live
4. Wait for it to finish (green check)

**✅ You should see** all steps succeed, including **Tofu Apply** creating your bucket. The **Tofu Plan** step will show as skipped (because this was a push, not a PR).

**Step 4c: ✅ Verify the deploy actually happened.** 📋 In your terminal:

```
aws s3 ls | Select-String "cicd-demo"
```
(Mac/Linux: `aws s3 ls | grep cicd-demo`)

**✅ You should see** `workshop-dev-cicd-demo-<YOUR_ACCOUNT_ID>`.

> **🎉 GitHub just deployed to AWS with no stored credentials.** The pipeline authenticated via OIDC, assumed your deploy role, and applied — all triggered by a `git push`.

---

### Step 5: 🎯 Make a Change via Pull Request — Watch the Plan

Now the real workflow: propose a change on a branch, open a PR, and watch the pipeline preview it.

**Step 5a: Create a branch.** 📋 Copy and paste (from the project root):

```
git checkout -b add-versioning
```

**Step 5b: Make a change.** In VS Code, open `infra/environments/dev/main.tf`. Scroll to the very bottom of the file — **after the closing `}` of the `aws_s3_bucket "demo"` block** — and on a new line paste this block, then save:

```hcl
resource "aws_s3_bucket_versioning" "demo" {
  bucket = aws_s3_bucket.demo.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

**Step 5c: Commit and push the branch.** 📋 Copy and paste:

```
git add .
git commit -m "Enable versioning on the demo bucket"
git push -u origin add-versioning
```

**Step 5d: Open a Pull Request.**

1. Go to your repo on GitHub — you'll see a banner "add-versioning had recent pushes" with a **Compare & pull request** button. Click it.
2. (Or: **Pull requests** tab → **New pull request** → base `main`, compare `add-versioning`.)
3. Click **Create pull request**.

**Step 5e: Watch the plan run on the PR.**

1. On the PR page, select the **Checks** tab section near the top — you'll see your workflow running
2. Click **Details** → watch the **terraform** job
3. This time the **Tofu Plan** step runs (and **Tofu Apply** is skipped, because this is a PR, not a merge)

**✅ You should see** the plan output showing `aws_s3_bucket_versioning.demo will be created` and `Plan: 1 to add`.

**Step 5f: ✅ Verify nothing was applied yet.** 📋 In your terminal:

```
aws s3api get-bucket-versioning --bucket workshop-dev-cicd-demo-<YOUR_ACCOUNT_ID> --query "Status" --output text
```

**✅ You should see** empty output (versioning NOT enabled). **The PR only planned — it did not change anything.** This is the review gate: reviewers see what *would* happen before approving.

---

### Step 6: Merge the PR — Watch the Apply

**Step 6a:** On the PR page in GitHub, click **Merge pull request** → **Confirm merge**.

> **💡 In a real team**, a teammate would review the plan and approve first. Here you're both author and reviewer — but the workflow is identical.

**Step 6b: Watch the apply run.**

1. Go to the **Actions** tab
2. You'll see a new run triggered by the merge (a `push` to `main`)
3. Click it → the **terraform** job → this time **Tofu Apply** runs

**✅ You should see** `aws_s3_bucket_versioning.demo: Creating...` and `Apply complete! Resources: 1 added`.

**Step 6c: ✅ Verify the change is now live.** 📋 In your terminal:

```
aws s3api get-bucket-versioning --bucket workshop-dev-cicd-demo-<YOUR_ACCOUNT_ID> --query "Status" --output text
```

**✅ You should see** `Enabled`.

> **🎉 The full GitOps cycle worked!** You changed code on a branch → the PR previewed it with a plan → merging deployed it automatically. You never touched AWS directly. Your Git `main` branch is now the source of truth, and the pipeline keeps AWS matching it.

**Step 6d: Sync your local `main`.** Your local repo is behind (the merge happened on GitHub). 📋 Copy and paste:

```
git checkout main
git pull origin main
```

**✅ You should see** the merge commit pulled down. Keep your local `main` in sync like this after every merge.

---

### Step 7: Add a Guarded "Destroy" Workflow

Your pipeline deploys automatically — but tearing infrastructure down should **never** be automatic. `destroy` is irreversible, so you'll build a separate workflow that only a human can trigger, and only after typing a confirmation word. This teaches an important principle: **guardrails scale with blast radius.** Additive changes (apply) are automated and reviewed; destructive changes (destroy) require deliberate human confirmation.

It also gives you a clean one-click teardown for when you're done — helpful for keeping your account at $0.

**Step 7a:** In VS Code, right-click `.github/workflows` → **New File** → name it exactly `destroy.yml`. 📋 Paste this, **replacing `<YOUR_ACCOUNT_ID>`**, then save:

```yaml
name: Destroy Infrastructure

on:
  workflow_dispatch:
    inputs:
      confirm:
        description: 'Type "destroy" to confirm tearing down the dev environment'
        required: true
        type: string

permissions:
  id-token: write
  contents: read

jobs:
  destroy:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infra/environments/dev
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Verify confirmation
        run: |
          if [ "${{ github.event.inputs.confirm }}" != "destroy" ]; then
            echo "Confirmation text was not exactly 'destroy'. Aborting - nothing was changed."
            exit 1
          fi
          echo "Confirmation received. Proceeding to destroy the dev environment."

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<YOUR_ACCOUNT_ID>:role/github-actions-infra
          aws-region: us-east-1

      - name: Setup OpenTofu
        uses: opentofu/setup-opentofu@v1

      - name: Tofu Init
        run: tofu init

      - name: Tofu Destroy
        run: tofu destroy -auto-approve
```

**Replace `<YOUR_ACCOUNT_ID>`** (1 place). Save.

> **What makes this safe?**
>
> | Part | Why it matters |
> |------|----------------|
> | `workflow_dispatch` only (no `push`/`schedule`) | It can NEVER run automatically — only a human clicking the button starts it |
> | `inputs: confirm` | Forces the person to type a confirmation word before the run starts |
> | `Verify confirmation` step | Runs first (after checkout) and **aborts** unless the word is exactly `destroy` — before touching AWS |

**Step 7b:** Commit and push. 📋 Copy and paste (from the project root):

```
git add .
git commit -m "Add guarded manual destroy workflow"
git push
```

> **💡 This push triggers the apply pipeline too, but there are no infrastructure changes, so it will report "No changes." Wait for it to finish.**

**Step 7c: Test the guard (the abort path).**

1. On GitHub: **Actions** → **Destroy Infrastructure** → **Run workflow**
2. In the **confirm** box, type something wrong like `no` → **Run workflow**
3. Wait ~30 seconds, refresh, and open the run

**✅ You should see** the run **FAIL** at the **Verify confirmation** step with "Confirmation text was not exactly 'destroy'. Aborting." Your infrastructure is untouched — the guard stopped it before AWS was contacted.

**Step 7d: ✅ Verify nothing was destroyed.** 📋 In your terminal:

```
aws s3 ls | Select-String "cicd-demo"
```
(Mac/Linux: `aws s3 ls | grep cicd-demo`)

**✅ You should still see** your demo bucket — the wrong confirmation blocked the destroy.

> **🎉 Guardrail proven.** A destructive pipeline that refuses to run without explicit human confirmation. You'll use this same workflow (with the correct word, `destroy`) to clean up at the end of Lab 8C.

---

## What You Just Did

You built and used a complete CI/CD pipeline for infrastructure:

| What You Did | Why It Matters |
|--------------|----------------|
| Wrote a GitHub Actions workflow with OIDC auth | The pipeline deploys with no stored AWS keys |
| Ran `plan` automatically on pull requests | Every change is previewed and reviewable before it happens |
| Ran `apply` automatically on merge to `main` | Approved changes deploy themselves — consistent and hands-off |
| Made a real change through the PR → merge cycle | You practiced the exact GitOps workflow real teams use |
| Added a guarded, manual-only destroy workflow | Destructive actions require deliberate human confirmation — guardrails scale with blast radius |

**Key takeaways:**
- **Git is the source of truth.** What's on `main` is what's deployed; the pipeline enforces it.
- **Plan-on-PR is a safety and review gate** — nobody merges a change without seeing its effect first.
- **`id-token: write` is mandatory** for OIDC — the most common reason a pipeline fails to authenticate.
- **The same IaC code runs locally and in CI** — only the authentication method changes.
- **Guardrails scale with blast radius** — apply is automated; destroy is manual-only and requires a typed confirmation.

> **💡 What persists:** Everything stays for Lab 8C, which adds automated drift detection on top of this pipeline. Do not delete your repo or AWS resources.

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

The SAA exam tests:
- Automated, repeatable deployment pipelines and their benefits
- OIDC federation vs. static credentials
- The role of change review/approval in deployment safety
- Event-driven automation (pipelines triggered by repository events)

**Sample question type:** "A team wants infrastructure changes reviewed before deployment and applied automatically once approved. What approach achieves this?"  
**Answer:** A CI/CD pipeline that runs a plan on pull requests for review and applies on merge to the main branch.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| Workflow fails at "Configure AWS credentials" with an OIDC error | Missing `id-token: write` permission, or the trust policy `sub` doesn't match your repo | Confirm `permissions: id-token: write` is in the workflow, and the repo in Lab 8A's trust policy matches `yourname/workshop-iac` exactly |
| `Not authorized to perform sts:AssumeRole` | The pipeline role can't assume the deploy role | Check Lab 8A Step 8 — the `AssumeDeployRole` permission and the deploy role ARN must be correct |
| `Error acquiring the state lock` | A previous run was interrupted | In your terminal, `cd` into the dev folder and run `tofu force-unlock <LOCK_ID>` |
| Plan runs on merge, or apply runs on PR | The `if:` conditions are wrong | Compare your `infra.yml` to Step 3 exactly — plan is `if: pull_request`, apply is `if: push` to main |
| Workflow doesn't run at all | Wrong file path | It must be `.github/workflows/infra.yml` exactly, committed to `main` |
| Backend/init fails with S3 access denied | Pipeline role missing state bucket permission | Recheck Lab 8A Step 7 — the state bucket name includes your account ID |
| Destroy workflow's "Run workflow" button has no confirm box | The `inputs:` block is missing or malformed | Compare `destroy.yml` to Step 7a; `workflow_dispatch.inputs.confirm` must be present and it must be on `main` |
| Destroy workflow destroyed things even with a wrong word | The `Verify confirmation` step is missing or after the AWS steps | It must run right after checkout, before the AWS/tofu steps, exactly as in Step 7a |

---

## Cleanup

> **⚠️ Keep everything for Lab 8C** (drift detection builds on this pipeline).

**If you must stop entirely**, tear down the demo resource using the destroy workflow you just built: **Actions** → **Destroy Infrastructure** → **Run workflow** → type `destroy` → **Run workflow**. (Or run `tofu destroy` locally from `infra/environments/dev`.) Then follow Lab 8A's cleanup for the roles, and Lab 7A's cleanup for the state backend, if you're fully done.

---

## Help

If you get stuck, post in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (or the Actions run URL)
3. The **full error message**
4. Your **operating system**
