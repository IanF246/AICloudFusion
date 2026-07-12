# Lab 8A: Connect Your Repo to GitHub with OIDC (No Stored Keys)

**Session:** 8 — CI/CD for Infrastructure  
**Track:** Solutions Architecture + IaC  
**Difficulty:** Beginner  
**Estimated Time:** 35–45 minutes  
**Target Cert:** AWS Solutions Architect – Associate (SAA)

---

## Overview

So far, *you* have run `tofu plan` and `tofu apply` from your own laptop. In a real team, a **pipeline** does that automatically when code is pushed. But for a pipeline to touch AWS, it needs credentials — and the dangerous, old-fashioned way is to paste an AWS access key into GitHub. Keys leak, and a leaked key is a breach.

In this lab you'll set up the modern, secure alternative: **GitHub OIDC**. GitHub proves its identity to AWS with a short-lived token, and AWS hands back temporary credentials. **No long-lived keys are ever stored anywhere.**

By the end of this lab, you will have:

1. **Your `workshop-iac` project pushed to a GitHub repository**
2. **A GitHub OIDC identity provider** registered in AWS
3. **A pipeline IAM role** that only *your* repo's GitHub Actions can assume
4. Everything ready for Lab 8B to run a real pipeline

**The feedback loop for this lab:** You'll see the problem (storing AWS keys in GitHub is a breach waiting to happen), apply the fix (OIDC trust — no keys), and in Lab 8B verify a pipeline authenticating to AWS with zero stored secrets.

---

## Prerequisites

- ✅ Completed **Lab 7A–7C** — you must have the `workshop-iac` folder with your `infra/` structure, the state backend, and the `workshop-tofu-deploy-role`
- ✅ AWS CLI authenticated (`aws sts get-caller-identity`)
- ✅ A **GitHub account** — free at [github.com/signup](https://github.com/signup) if you don't have one
- ✅ **Git** installed and **VS Code**

> **⚠️ This lab builds on Session 7.** You need your `workshop-iac` project and its AWS bootstrap resources (state bucket, lock table, deploy role) intact.

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| AWS IAM | OIDC provider + pipeline role | Always Free |
| GitHub | Repository + Actions | Free for public repos and generous free minutes for private |

**Estimated cost for this lab: $0.00**

---

## Concepts

**CI/CD** — Continuous Integration / Continuous Delivery. In plain terms: automation that runs whenever you push code. For infrastructure, that means "when I push a change, a pipeline runs `tofu plan` and `tofu apply` for me" instead of me running them by hand.

**GitHub Actions** — GitHub's built-in automation. You describe jobs in a YAML file under `.github/workflows/`, and GitHub runs them on its own servers when events happen (a push, a pull request, a schedule).

**OIDC (OpenID Connect)** — a standard way for one system to prove its identity to another using short-lived tokens instead of shared passwords. GitHub Actions can present an OIDC token to AWS that says "I am a workflow running in *this specific repo*." AWS verifies it and grants temporary credentials.

**The problem OIDC solves** — the old way was to create an AWS access key and store it as a GitHub secret. That key is long-lived: if it leaks (in a log, a screenshot, a compromised dependency), an attacker has lasting access to your AWS account. OIDC issues credentials that last minutes and are scoped to one repo — nothing to steal.

**OIDC identity provider** — a configuration in AWS IAM that says "I trust tokens issued by GitHub's OIDC service." You register it once per account.

**Pipeline role** — an IAM role that GitHub Actions assumes. Its trust policy says "only OIDC tokens from *my specific repo* may assume this role." Its permissions say what the pipeline is allowed to do.

---

## ⚠️ Reminder: How to Read Commands

Files are created in **VS Code** (right-click the folder → New File → exact name). Commands go in your terminal. See Lab 7A's "How to Create Files" section if you need a refresher.

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account number | `123456789012` |
| `<YOUR_GITHUB_USERNAME>` | Your GitHub username | `janedoe` |

---

## Lab Steps

### Step 1: Set Your Profile and Open Your Project

**Step 1a:** Set your AWS profile and verify:

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

**✅ You should see** your account ID. **📝 Write it down** — you'll use it several times.

**Step 1b:** Open your project:

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

---

## PART 1 — Push Your Project to GitHub

In Lab 7A you made `workshop-iac` a local Git repository and committed to it. Now you'll put it on GitHub so a pipeline can run against it.

### Step 2: Create a GitHub Repository

1. Go to [github.com/new](https://github.com/new)
2. **Repository name:** `workshop-iac`
3. **Visibility:** choose **Private** (your infrastructure code is not something to share publicly)
4. **Do NOT** check "Add a README", ".gitignore", or "license" — your local project already has files, and adding these would cause a conflict
5. Click **Create repository**

**✅ You should see** a page with quick-start instructions and a URL like `https://github.com/<YOUR_GITHUB_USERNAME>/workshop-iac.git`. Keep this tab open.

> **💡 Why private?** Infrastructure code can reveal resource names, account structure, and configuration. Keep it private unless you have a reason to share it.

---

### Step 3: Connect Your Local Repo and Push

**Step 3a:** Make sure you're in the project root and everything is committed. 📋 Copy and paste (from the `workshop-iac` folder):

```
git status
```

**✅ You should see** "nothing to commit, working tree clean." If it lists changes, commit them first:

```
git add .
git commit -m "Session 8 starting point"
```

**Step 3b:** Connect your local repo to the GitHub repo and push. 📋 Copy and paste, **replacing `<YOUR_GITHUB_USERNAME>`**:

```
git remote add origin https://github.com/<YOUR_GITHUB_USERNAME>/workshop-iac.git
```

```
git branch -M main
```

```
git push -u origin main
```

> **💡 If prompted to log in:** a browser window or a prompt will ask you to authenticate to GitHub. Follow it (sign in / authorize). This links your computer to your GitHub account for pushing.

**✅ You should see** your files uploading, ending with a line like `branch 'main' set up to track 'origin/main'`.

**Step 3c: ✅ Checkpoint** — Refresh your GitHub repo page in the browser. You should now see your `infra/` folder, `.gitignore`, and policy files on GitHub.

> **⚠️ Confirm your state files did NOT upload.** Look through the repo on GitHub — you should NOT see any `.tfstate` files or a `.terraform` folder. Your `.gitignore` from Lab 7A keeps them out. State lives in S3, never in Git. If you see state files, stop and tell your facilitator.

---

## PART 2 — 🚨 See the Problem: Long-Lived Keys Are Dangerous

Before building the secure setup, understand what you're avoiding.

> **🚨 The old, dangerous way:** To let a pipeline deploy, teams used to run `aws iam create-access-key`, then paste that key ID and secret into GitHub as repository secrets. The pipeline used them to authenticate.
>
> **Why it's dangerous:** That access key is **long-lived** — it works until someone manually deletes it. If it leaks even once — printed in a build log, committed by accident, exposed through a compromised GitHub Action — an attacker has ongoing access to your AWS account. Real companies have been breached exactly this way.

**You will NOT do that.** Instead, you'll set up OIDC, where GitHub proves its identity per-run and AWS hands back credentials that expire in minutes. There is no key to leak.

---

## PART 3 — ✅ Fix: Set Up OIDC Trust

### Step 4: Create the GitHub OIDC Identity Provider

This tells AWS "I trust identity tokens issued by GitHub Actions." You do this once per AWS account.

📋 Copy and paste:

```
aws iam create-open-id-connect-provider --url https://token.actions.githubusercontent.com --client-id-list sts.amazonaws.com --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

**✅ You should see** JSON output with an `OpenIDConnectProviderArn`.

> **💡 If you get `EntityAlreadyExists`:** that's fine — a provider for GitHub already exists in your account (maybe from other work). Skip to Step 5; you'll reuse the existing one.

> **What does this do?**
> - `--url` — GitHub's OIDC token issuer
> - `--client-id-list sts.amazonaws.com` — the "audience": tokens must be intended for AWS STS
> - `--thumbprint-list` — a fingerprint of GitHub's certificate (AWS now verifies GitHub's certificate automatically, but the CLI still requires this value)

---

### Step 5: Create the Pipeline Trust Policy

This trust policy is the heart of OIDC security: it says **only GitHub Actions running in YOUR specific repo** may assume the pipeline role.

**Step 5a:** In VS Code, right-click the top-level `workshop-iac` folder → **New File** → name it exactly `pipeline-trust.json`. 📋 Paste this, **replacing `ACCOUNT_ID_HERE` and `YOUR_GITHUB_USERNAME_HERE`**, then save:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::ACCOUNT_ID_HERE:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                },
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:YOUR_GITHUB_USERNAME_HERE/workshop-iac:*"
                }
            }
        }
    ]
}
```

**Replace 2 things:** `ACCOUNT_ID_HERE` (your 12-digit account ID) and `YOUR_GITHUB_USERNAME_HERE` (your GitHub username). Save the file.

> **What does the `sub` condition do?** `repo:yourname/workshop-iac:*` means only workflows from *that exact repository* can assume this role. A workflow in any other repo — even yours — is rejected. This is what makes OIDC safe: the trust is scoped to one repo.

---

### Step 6: Create the Pipeline Role

📋 Copy and paste (from the `workshop-iac` folder, where the file is):

```
aws iam create-role --role-name github-actions-infra --assume-role-policy-document file://pipeline-trust.json
```

**✅ You should see** JSON output with the role's ARN. **📝 Write down the role ARN** (`arn:aws:iam::<ACCOUNT_ID>:role/github-actions-infra`) — you'll need it in Lab 8B.

---

### Step 7: Give the Pipeline Role Its Permissions

The pipeline role needs to do exactly two things: **assume your deploy role** (the same one you used locally in Session 7) and **read/write the state backend**. It does NOT need direct permissions on Lambda, S3 apps, etc. — because the actual resource work happens through the deploy role it assumes. This keeps the pipeline role minimal.

**Step 7a:** In VS Code, right-click the `workshop-iac` folder → **New File** → `pipeline-permissions.json`. 📋 Paste this, **replacing `ACCOUNT_ID_HERE` in 3 places**, then save:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AssumeDeployRole",
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::ACCOUNT_ID_HERE:role/workshop-tofu-deploy-role"
        },
        {
            "Sid": "StateBucketAccess",
            "Effect": "Allow",
            "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket", "s3:DeleteObject"],
            "Resource": [
                "arn:aws:s3:::workshop-tofu-state-ACCOUNT_ID_HERE",
                "arn:aws:s3:::workshop-tofu-state-ACCOUNT_ID_HERE/*"
            ]
        },
        {
            "Sid": "StateLockAccess",
            "Effect": "Allow",
            "Action": ["dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:DeleteItem"],
            "Resource": "arn:aws:dynamodb:us-east-1:ACCOUNT_ID_HERE:table/terraform-locks"
        }
    ]
}
```

**Replace `ACCOUNT_ID_HERE` in 3 places** (the assume-role ARN, and the state bucket appears in the S3 resource lines and DynamoDB ARN). Note the state **bucket name** also includes your account ID — so `workshop-tofu-state-ACCOUNT_ID_HERE` becomes e.g. `workshop-tofu-state-123456789012`. Save the file.

**Step 7b:** Attach the policy. 📋 Copy and paste:

```
aws iam put-role-policy --role-name github-actions-infra --policy-name pipeline-permissions --policy-document file://pipeline-permissions.json
```

**✅ No output means success.**

> **💡 Why can the pipeline role assume the deploy role?** Your `workshop-tofu-deploy-role` (from Lab 7A) trusts your whole account. Any identity in the account that has `sts:AssumeRole` permission on it — like this pipeline role — can assume it. That's why the pipeline role only needs the small `AssumeDeployRole` permission, and the deploy role does the heavy lifting. **The same deploy role you used on your laptop is what the pipeline will use.**

---

### Step 8: Commit the New Files

📋 Copy and paste (from the project root):

```
git add .
git commit -m "Add GitHub OIDC pipeline role and trust policy"
git push
```

**✅ You should see** the push succeed.

---

## What You Just Did

You set up secure, keyless authentication between GitHub and AWS:

| What You Built | Why It Matters |
|---------------|----------------|
| Pushed `workshop-iac` to GitHub | Your infrastructure code now lives where a pipeline can run against it |
| Registered a GitHub OIDC provider | AWS now trusts identity tokens from GitHub Actions |
| Created a pipeline role scoped to your repo | Only *your* repo's workflows can assume it — nothing else |
| Gave it minimal permissions (assume deploy role + state) | Least privilege — the pipeline role can't do more than run deployments |

**Key takeaways:**
- **Never store long-lived AWS keys in a pipeline.** OIDC gives short-lived, per-run credentials with nothing to leak.
- **Trust is scoped to one repo** via the `sub` condition — the single most important line in the trust policy.
- **The pipeline reuses your deploy role**, so the same code and permissions work whether you run locally or in CI.

> **💡 What persists:** Your GitHub repo, the OIDC provider, and both roles all stay — Lab 8B uses them to run an actual pipeline. Do not delete them.

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

The SAA (and Security Specialty) exams test:
- Federated identity and OIDC as an alternative to long-lived IAM access keys
- IAM roles, trust policies, and `sts:AssumeRoleWithWebIdentity`
- Least-privilege role design
- Why short-lived credentials are more secure than static keys

**Sample question type:** "A CI/CD pipeline in GitHub needs to deploy to AWS. What is the MOST secure way to grant access?"  
**Answer:** Configure an IAM OIDC identity provider for GitHub and have the pipeline assume a role via `AssumeRoleWithWebIdentity` — no long-lived access keys stored in the pipeline.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `git push` rejected / auth failed | Not authenticated to GitHub | Follow the login prompt; or use a personal access token. See GitHub's docs on authenticating from the command line. |
| Push says "updates were rejected" | The GitHub repo isn't empty (you added a README) | Recreate the repo without a README/gitignore, or run `git pull --rebase origin main` then push |
| `EntityAlreadyExists` on the OIDC provider | A GitHub OIDC provider already exists in your account | That's fine — reuse it. Skip to Step 5. |
| `MalformedPolicyDocument` creating the role | Placeholder not replaced in `pipeline-trust.json` | Open the file; confirm the account ID and username are filled in and it's valid JSON |
| You see `.tfstate` files on GitHub | Your `.gitignore` isn't excluding them | Confirm `.gitignore` from Lab 7A is in the repo root; remove the state files from Git with `git rm --cached` and re-commit |
| `NoSuchEntity` referencing the deploy role in Step 7 | The deploy role from 7A is missing | You need `workshop-tofu-deploy-role` from Lab 7A — redo that part if it was deleted |

---

## Cleanup

> **⚠️ Keep everything for Lab 8B.** The GitHub repo, OIDC provider, and pipeline role are all needed next. Do not clean up unless you are stopping the CI/CD track.

**If you must stop entirely and remove what this lab created:**

```
aws iam delete-role-policy --role-name github-actions-infra --policy-name pipeline-permissions
aws iam delete-role --role-name github-actions-infra
```

To remove the OIDC provider (only if you created it and nothing else uses it), replacing `<YOUR_ACCOUNT_ID>`:
```
aws iam delete-open-id-connect-provider --open-id-connect-provider-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com
```

You can delete the GitHub repo from its **Settings → General → Delete this repository**.

---

## Help

If you get stuck, post in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran
3. The **full error message**
4. Your **operating system**
