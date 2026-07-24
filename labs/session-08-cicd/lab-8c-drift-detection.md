# Lab 8C: Drift Detection — Catching Manual Changes Automatically

**Session:** 8 — CI/CD for Infrastructure  
**Track:** Solutions Architecture + IaC  
**Difficulty:** Advanced  
**Estimated Time:** 40–50 minutes  
**Target Cert:** AWS Solutions Architect – Associate (SAA)

---

## Overview

Your pipeline (Lab 8B) keeps AWS matching your code — as long as *every* change goes through Git. But what happens when someone bypasses the process and changes something directly in the AWS console? That's called **drift**: reality no longer matches your code. Drift is dangerous — the next deploy might undo the manual change, or the manual change might hide a security hole nobody wrote down.

In this lab you'll build an automated **drift detection** workflow: a scheduled GitHub Actions job that runs `tofu plan` on a timer and **fails loudly** if reality has drifted from your code.

By the end you will:
1. Add a scheduled drift-detection workflow
2. Deliberately make a "sneaky" manual change in AWS (simulating someone bypassing Git)
3. Run the drift workflow and watch it **catch you** — the build turns red
4. Fix the drift and watch the build go green again

**The feedback loop for this lab:** You'll change a resource by hand (drift), run the detector and see it **fail** (drift caught), then resolve it and see it **pass** (drift gone). It viscerally proves the core rule of IaC: **the code is the single source of truth, and manual changes are the enemy.**

---

## Prerequisites

- ✅ Completed **Lab 8B** — you must have:
  - Your `workshop-iac` repo on GitHub with the working `infra.yml` pipeline
  - The demo bucket deployed (with versioning enabled, from 8B)
  - The OIDC provider and `github-actions-infra` role
- ✅ Your local `main` in sync (`git checkout main` then `git pull origin main`)

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| GitHub Actions | Scheduled workflow runner | Free minutes |
| AWS (S3 demo bucket) | The resource being checked | Within free tier |

**Estimated cost for this lab: $0.00**

---

## Concepts

**Drift** — when the real state of your infrastructure no longer matches what your code says. Caused by manual console edits, other tools, or out-of-band scripts.

**Why drift is dangerous** — your code stops being trustworthy. A future `apply` may silently revert someone's manual fix, or a manual change (like opening a security group) exists with no record in Git. Teams that don't detect drift get nasty surprises.

**`tofu plan -detailed-exitcode`** — a special mode of plan that communicates through its **exit code**:
- `0` = no changes (no drift) ✅
- `2` = changes detected (drift!) — the command "fails" on purpose
- `1` = an actual error

A pipeline can use that exit code: exit `2` fails the job, turning the build red and alerting the team.

**Scheduled workflow (`on: schedule`)** — GitHub Actions can run a workflow on a cron timer (e.g., every morning), not just on pushes. Perfect for a regular drift check.

**`workflow_dispatch`** — lets you trigger a workflow manually with a button, handy for testing without waiting for the schedule.

---

## ⚠️ Reminder: How to Read Commands

Files in **VS Code**, commands in your terminal, some steps on the **GitHub website**.

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account number | `123456789012` |

---

## Lab Steps

### Step 1: Open Your Project and Sync

**Windows (PowerShell):**
```powershell
cd ~\Desktop\workshop-iac
git checkout main
git pull origin main
code .
```

**macOS / Linux:**
```bash
cd ~/Desktop/workshop-iac
git checkout main
git pull origin main
code .
```

**✅ You should see** your local `main` up to date with the versioning change from Lab 8B.

---

### Step 2: Create the Drift Detection Workflow

**Step 2a:** In VS Code, right-click `.github/workflows` → **New File** → name it exactly `drift.yml`.

**Step 2b:** 📋 Paste this, **replacing `<YOUR_ACCOUNT_ID>`**, then save:

```yaml
name: Drift Detection

on:
  schedule:
    - cron: "0 8 * * 1"    # every Monday at 08:00 UTC
  workflow_dispatch: {}     # allow manual runs from the Actions tab

permissions:
  id-token: write
  contents: read

jobs:
  drift:
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
        with:
          tofu_wrapper: false

      - name: Tofu Init
        run: tofu init

      - name: Check for drift
        run: tofu plan -detailed-exitcode
```

**Replace `<YOUR_ACCOUNT_ID>`** (1 place). Save.

> **⚠️ The single most important line: `tofu_wrapper: false`.** By default, the setup-opentofu action wraps the `tofu` command in a helper that **swallows the exit code**. If you leave the wrapper on, `tofu plan -detailed-exitcode` will report exit `2` (drift!) but the step will still show green — your drift detection would silently never work. `tofu_wrapper: false` makes the real exit code reach GitHub, so exit `2` correctly fails the job.

> **What does this workflow do?**
>
> | Part | Meaning |
> |------|---------|
> | `on: schedule (cron)` | Runs automatically every Monday morning |
> | `on: workflow_dispatch` | Lets you trigger it by hand for testing |
> | `tofu_wrapper: false` | So the plan's exit code reaches GitHub (critical) |
> | `tofu plan -detailed-exitcode` | Exit 0 = clean, exit 2 = drift → job fails |

---

### Step 3: Push the Drift Workflow

📋 Copy and paste (from the project root):

```
git add .
git commit -m "Add scheduled drift detection workflow"
git push
```

> **💡 Heads up:** because your `infra.yml` pipeline also triggers on push to `main`, pushing this will run an **apply** too. That's harmless — there are no infrastructure changes in this commit, so the apply will say "no changes." Wait for both workflows to finish (Actions tab) before continuing.

**✅ Checkpoint:** Go to the **Actions** tab. You should now see **Drift Detection** listed as a workflow (in the left sidebar of the Actions page).

---

### Step 4: Confirm "No Drift" First (the green baseline)

Before creating drift, confirm the detector passes when everything matches.

**Step 4a:** Trigger it manually. On GitHub: **Actions** tab → **Drift Detection** (left sidebar) → **Run workflow** button → **Run workflow**.

> **💡 The "Run workflow" button appears** because you included `workflow_dispatch` in the YAML.

**Step 4b:** Wait ~1 minute, then refresh. Click into the run → the **drift** job.

**✅ You should see** the **Check for drift** step succeed (green). The plan found `No changes` — reality matches your code. This is your healthy baseline.

---

### Step 5: 🚨 Create Drift — Make a Sneaky Manual Change

Now play the role of someone who bypasses Git and changes AWS directly. You'll add a tag to the demo bucket straight from the CLI — a change that exists in AWS but NOT in your code.

📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>`** (this sets the bucket's tags directly, adding one your code doesn't know about — and changing a managed value):

**Windows (PowerShell):**
```powershell
aws s3api put-bucket-tagging --bucket workshop-dev-cicd-demo-<YOUR_ACCOUNT_ID> --tagging "TagSet=[{Key=Project,Value=workshop-iac},{Key=Environment,Value=dev},{Key=ManagedBy,Value=SOMEONE-CHANGED-THIS-BY-HAND}]"
```

**macOS / Linux:**
```bash
aws s3api put-bucket-tagging --bucket workshop-dev-cicd-demo-<YOUR_ACCOUNT_ID> --tagging "TagSet=[{Key=Project,Value=workshop-iac},{Key=Environment,Value=dev},{Key=ManagedBy,Value=SOMEONE-CHANGED-THIS-BY-HAND}]"
```

**✅ No output means success.**

> **What just happened?** Your code says the `ManagedBy` tag should be `opentofu`. Someone (you, playing the bad actor) just changed it to `SOMEONE-CHANGED-THIS-BY-HAND` directly in AWS, without touching Git. Your code and reality no longer agree. **That is drift.**

---

### Step 6: ✅ Watch the Detector Catch It

**Step 6a:** Trigger the drift workflow again: **Actions** tab → **Drift Detection** → **Run workflow** → **Run workflow**.

**Step 6b:** Wait ~1 minute, refresh, and click into the run.

**✅ You should see the run FAIL (red ✗).** Open the **Check for drift** step — the plan output shows something like:

```
  # aws_s3_bucket.demo will be updated in-place
      ~ tags = {
          ~ "ManagedBy" = "SOMEONE-CHANGED-THIS-BY-HAND" -> "opentofu"
        }
Plan: 0 to add, 1 to change, 0 to destroy.
```

and the step ends with a non-zero exit code, failing the job.

> **🎉 The detector caught the drift!** The scheduled job (or your manual run) noticed that reality no longer matches the code and turned the build red. In a real team, this failure would send an alert (email/Slack) so someone investigates. Without this, that sneaky manual change could sit undetected for months.

> **💡 This is why `tofu_wrapper: false` mattered.** With the wrapper on, this exact run would have shown **green** despite detecting the change — the failure signal would have been silently swallowed. Live-testing this is the only way to know.

---

### Step 7: ✅ Resolve the Drift — Back to Green

You have two honest choices when drift is found: (a) revert the manual change, or (b) update your code to intentionally adopt it. Here you'll **revert** — the code is the source of truth, so make reality match it again.

**Step 7a:** Re-run your pipeline's apply to reset AWS to match the code. The simplest way: make an empty commit to `main` to trigger the apply pipeline. 📋 Copy and paste (from the project root):

```
git commit --allow-empty -m "Trigger apply to correct drift"
git push
```

Wait for the **Infrastructure CI/CD** run (Actions tab) to finish its **Tofu Apply** — it will reset the `ManagedBy` tag back to `opentofu`.

> **💡 Alternatively**, you could run `tofu apply` locally from `infra/environments/dev`. Either way, apply reconciles reality back to the code.

**Step 7b:** Confirm the tag is corrected. 📋 In your terminal:

```
aws s3api get-bucket-tagging --bucket workshop-dev-cicd-demo-<YOUR_ACCOUNT_ID> --query "TagSet[?Key=='ManagedBy'].Value" --output text
```

**✅ You should see** `opentofu` again.

**Step 7c:** Run the drift workflow one more time (**Actions** → **Drift Detection** → **Run workflow**).

**✅ You should see** it PASS (green) — no more drift. Reality matches code again.

> **🎉 Full drift cycle complete:** clean → someone drifted it → the detector caught it (red) → you reconciled → clean again (green). This is how mature teams keep infrastructure honest.

---

## What You Just Did

You added automated drift detection and proved it works:

| What You Did | Why It Matters |
|--------------|----------------|
| Built a scheduled drift-detection workflow | Infrastructure is checked regularly, not just when someone remembers |
| Used `tofu plan -detailed-exitcode` | Turns "is there drift?" into a pass/fail signal a pipeline can act on |
| Set `tofu_wrapper: false` | So the exit code actually reaches GitHub — without it, detection silently fails |
| Created drift, caught it, and resolved it | Experienced the full detect-and-correct cycle real teams rely on |

**Key takeaways:**
- **Drift is inevitable** — people make manual changes. The question is whether you *detect* it.
- **The code is the source of truth.** When drift is found, you either revert reality or intentionally update the code — never leave them out of sync.
- **Exit codes are how pipelines make decisions.** `-detailed-exitcode` returning 2 is a feature, not an error.
- **Tooling details matter.** A wrapper swallowing an exit code can make a "working" pipeline that actually does nothing — always verify the failure path, not just the success path.

> **💡 Session 8 complete.** You've built keyless OIDC auth (8A), a plan/apply GitOps pipeline (8B), and automated drift detection (8C) — a professional CI/CD setup for infrastructure.

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

The SAA exam tests:
- Configuration drift and the importance of detecting it
- Infrastructure as Code as the source of truth
- Automated, scheduled compliance/consistency checks
- Why out-of-band (manual console) changes undermine automation

**Sample question type:** "A company manages infrastructure with IaC but engineers occasionally make manual console changes. How can they detect when infrastructure no longer matches their code?"  
**Answer:** Run a scheduled plan (drift detection) that compares actual state to the configuration and alerts when they differ.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| Drift run shows **green even after you made a manual change** | The `tofu_wrapper: false` line is missing or misplaced | Confirm it's under the `Setup OpenTofu` step's `with:` block. This is the #1 cause. |
| "Run workflow" button doesn't appear | `workflow_dispatch` missing, or the workflow file isn't on `main` yet | Confirm `workflow_dispatch: {}` is in the YAML and you pushed it to `main` |
| Drift run fails at OIDC/credentials step | Same trust/permission setup as 8B | Verify `id-token: write` and the pipeline role ARN; recheck Lab 8A |
| Plan shows drift you didn't cause | Something else changed the resource, or a previous apply didn't complete | Investigate what changed; run `apply` to reconcile if the code is correct |
| `Error acquiring the state lock` | An overlapping run holds the lock | Wait for other runs to finish; if stuck, `tofu force-unlock <LOCK_ID>` locally |
| Manual tag change didn't cause drift | You changed a tag the code doesn't manage (extra tags can be additive) | Use the exact command in Step 5, which changes a *managed* tag value (`ManagedBy`) |

---

## Cleanup

You've reached the end of the IaC + CI/CD track. If you want to remove everything you built across Sessions 7 and 8:

**Step 1 — Destroy the application resources** using the guarded destroy workflow you built in Lab 8B: on GitHub, **Actions** → **Destroy Infrastructure** → **Run workflow** → type `destroy` in the confirm box → **Run workflow**. Wait for it to finish (green), then confirm your demo bucket is gone with `aws s3 ls`.

(Alternatively, run `tofu destroy` locally from `infra/environments/dev`.)

**Step 2 — Remove the pipeline role and OIDC provider** (from Lab 8A's cleanup):

```
aws iam delete-role-policy --role-name github-actions-infra --policy-name pipeline-permissions
aws iam delete-role --role-name github-actions-infra
```

**Step 3 — Remove the state backend and deploy role** (from Lab 7A's cleanup): delete the state bucket, the `terraform-locks` table, and `workshop-tofu-deploy-role`.

**Step 4 — Delete the GitHub repo** from its **Settings → Delete this repository**, and remove the local `workshop-iac` folder if you wish.

> **💡 Want to keep your work as a portfolio piece?** Leave the GitHub repo in place (just run `tofu destroy` to stop any AWS charges). A working IaC + CI/CD repo is excellent to show in interviews.

---

## Help

If you get stuck, post in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (or the Actions run URL)
3. The **full error message**
4. Your **operating system**
