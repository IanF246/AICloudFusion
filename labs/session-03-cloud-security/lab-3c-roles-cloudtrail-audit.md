# Lab 3C: Build a Secure Multi-User Environment with Audit Logging

**Session:** 3 — Cloud Security Fundamentals  
**Track:** Cloud Security & IR  
**Difficulty:** Advanced  
**Estimated Time:** 45–55 minutes  
**Target Cert:** AWS Security Specialty

---

## Overview

In this lab, you will:

1. **Set up CloudTrail** — AWS's audit logging service that records every action in your account
2. **Create IAM roles** — temporary identities that engineers "assume" to get specific permissions
3. **Assume the developer role** — test what a developer can and cannot do
4. **Assume the auditor role** — review the audit trail and prove you can see what happened
5. **Demonstrate separation of duties** — developers cannot cover their tracks, auditors cannot make changes

By the end of this lab, you will understand how real organizations implement role-based access control (RBAC) and audit logging — two pillars of cloud security.

---

## Prerequisites

- ✅ Completed **Lab 1A** (AWS CLI installed and configured)
- ✅ Completed **Lab 3A** and **Lab 3B** (familiar with IAM users, policies, and groups)
- ✅ AWS CLI authenticated — run `aws sts get-caller-identity` and confirm it returns your account info
- ✅ **VS Code** installed (from Lab 1B) — any text editor works, but these labs assume VS Code

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| CloudTrail | Audit logging for AWS actions | First active trail is Always Free |
| IAM | Identity and Access Management | Always Free |
| Amazon S3 | Storage for CloudTrail logs | $0.023 per GB/month |

**Estimated cost for this lab: $0.00** — the first CloudTrail trail is free, and the logs bucket holds only a few small files and is deleted in cleanup.

---

## Concepts

Before we start, here is what each concept means:

**CloudTrail** is AWS's audit logging service. It records every API call made in your account — who did what, when, and from where. Think of it as a security camera for your cloud environment. If someone creates a resource, deletes a file, or changes a permission, CloudTrail logs it. This is essential for security investigations, compliance, and accountability.

**IAM Role** is an identity with permissions that anyone (or any service) can temporarily "assume." Unlike a user (which has permanent credentials), a role gives you temporary credentials that expire. Engineers switch between roles depending on what they need to do — like putting on different hats for different jobs.

**Assume Role** is the act of switching to a role's identity. When you assume a role, you get temporary credentials (Access Key + Secret Key + Session Token) that last for a limited time (usually 1 hour). This is more secure than permanent access keys because even if the credentials are stolen, they expire quickly.

**Separation of Duties** means no single person should have all the power. Developers can build and deploy, but they cannot modify security settings. Auditors can review logs and investigate incidents, but they cannot change anything. This prevents both accidents and insider threats.

**Trust Policy** defines WHO is allowed to assume a role. It is like a guest list — only the identities listed in the trust policy can use that role. In this lab, we allow the account root (meaning any admin in the account) to assume both roles.

**Privilege Escalation** is when a user gains more permissions than they should have — for example, a developer creating a new admin user to bypass restrictions. Good policies explicitly deny actions that could lead to privilege escalation.

---

## ⚠️ Reminder: How to Read Commands in This Lab

Commands are inside gray code boxes. **📋 Copy and paste** them into your terminal.

**Placeholders** look like `<THIS>` — replace them (including the `<` and `>`) with your actual values.

Here are the placeholders you will use in this lab:

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name from Lab 1A | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account ID (from `aws sts get-caller-identity`) | `123456789012` |
| `<YOUR_BUCKET_NAME>` | A globally unique S3 bucket name for CloudTrail logs | `jane-doe-cloudtrail-logs` |

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

**Re-establish your SSO login**

```
aws sso login --profile <YOUR_PROFILE_NAME>
```
This opens a browser window for you to authenticate to your account.


**Verify it works and get your Account ID.** 📋 Copy and paste:

```
aws sts get-caller-identity
```

**✅ You should see** your account ID and role.

> **📝 Write down your Account ID** (the 12-digit number in the `"Account"` field): ______________________________
>
> You will need this in several steps below.

---

### Step 2: Create Your Project Folder

**Step 2a: Create the folder**

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
mkdir ~\Desktop\workshop-lab-3c
cd ~\Desktop\workshop-lab-3c
```

**macOS / Linux:**

📋 Copy and paste:

```bash
mkdir ~/Desktop/workshop-lab-3c
cd ~/Desktop/workshop-lab-3c
```

> **What does this do?** Creates a new folder on your Desktop called `workshop-lab-3c` and moves your terminal into that folder.

**Step 2b: Verify you're in the right folder**

📋 Copy and paste:

```
pwd
```

**✅ You should see** a path ending in `workshop-lab-3c`.

> **💡 From now on, save ALL files you create in this lab to this folder.**

**Step 2c: Open the folder in VS Code**

📋 Copy and paste:

```
code .
```

> **What does this do?** This opens VS Code with `workshop-lab-3c` as its **file tree** on the left. This lab creates several JSON policy files — opening the folder now means each one lands in the right place. (You set up the `code` command in Lab 1B — if you see `'code' is not recognized`, close and reopen your terminal, or revisit Lab 1B, Step 6.)

---

### Step 3: Create an S3 Bucket for CloudTrail Logs

CloudTrail needs an S3 bucket to store its log files. Every action in your account will be recorded as a JSON file in this bucket.

📋 Copy and paste this command, **replacing `<YOUR_BUCKET_NAME>`**:

```
aws s3 mb s3://<YOUR_BUCKET_NAME> --region us-east-1
```

> **🔄 Example:**
> ```
> aws s3 mb s3://jane-doe-cloudtrail-logs --region us-east-1
> ```

**✅ You should see:**

```
make_bucket: <YOUR_BUCKET_NAME>
```

---

### Step 4: Write and Apply the CloudTrail Bucket Policy

CloudTrail needs permission to write log files to your bucket. You must add a bucket policy that explicitly allows the CloudTrail service to put objects in the bucket.

**Step 4a:** In the VS Code file tree, click the **New File** icon and name the file `cloudtrail-bucket-policy.json`.

**Step 4b:** 📋 Copy and paste this entire block into it, **replacing `<YOUR_BUCKET_NAME>`** (2 places) and **`<YOUR_ACCOUNT_ID>`** (1 place):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AWSCloudTrailAclCheck",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudtrail.amazonaws.com"
            },
            "Action": "s3:GetBucketAcl",
            "Resource": "arn:aws:s3:::<YOUR_BUCKET_NAME>"
        },
        {
            "Sid": "AWSCloudTrailWrite",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudtrail.amazonaws.com"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::<YOUR_BUCKET_NAME>/AWSLogs/<YOUR_ACCOUNT_ID>/*",
            "Condition": {
                "StringEquals": {
                    "s3:x-amz-acl": "bucket-owner-full-control"
                }
            }
        }
    ]
}
```

> **🔄 Example:** If your bucket is `jane-doe-cloudtrail-logs` and your account ID is `123456789012`:
> - `"Resource": "arn:aws:s3:::jane-doe-cloudtrail-logs"`
> - `"Resource": "arn:aws:s3:::jane-doe-cloudtrail-logs/AWSLogs/123456789012/*"`

**Step 4c:** **Save** the file (**Ctrl+S** / **Cmd+S**). You should see `cloudtrail-bucket-policy.json` appear in the file tree.

> **⚠️ Common mistakes:** Make sure you replaced `<YOUR_BUCKET_NAME>` in all 2 places and `<YOUR_ACCOUNT_ID>` in 1 place. The account ID must be exactly 12 digits with no dashes or spaces.

> **What does this file do?** It grants the CloudTrail service two permissions:
> - **GetBucketAcl** — CloudTrail checks that it has permission to write to the bucket before starting
> - **PutObject** — CloudTrail writes log files to the `AWSLogs/<ACCOUNT_ID>/` path in your bucket
>
> The `Condition` ensures CloudTrail gives you (the bucket owner) full control over the log files it creates.

**Step 4d: Apply the bucket policy**

📋 Copy and paste, **replacing `<YOUR_BUCKET_NAME>`**:

```
aws s3api put-bucket-policy --bucket <YOUR_BUCKET_NAME> --policy file://cloudtrail-bucket-policy.json
```

**✅ No output means success.**

---

### Step 5: Create a CloudTrail Trail

Now create the trail that will record all API activity in your account.

📋 Copy and paste, **replacing `<YOUR_BUCKET_NAME>`**:

```
aws cloudtrail create-trail --name workshop-audit-trail --s3-bucket-name <YOUR_BUCKET_NAME> --is-multi-region-trail
```

**What does this do?**
- `create-trail` — creates a new CloudTrail trail
- `--name workshop-audit-trail` — the name of your trail
- `--s3-bucket-name` — where to store the log files
- `--is-multi-region-trail` — records activity from ALL AWS regions, not just us-east-1

**✅ You should see** JSON output with the trail details, including `"Name": "workshop-audit-trail"`.

---

### Step 6: Start Logging

Creating a trail does not automatically start recording. You must explicitly start it.

📋 Copy and paste:

```
aws cloudtrail start-logging --name workshop-audit-trail
```

**✅ No output means success.** CloudTrail is now recording every API call in your account.

**✅ Console Checkpoint:**
1. Go to [https://console.aws.amazon.com/](https://console.aws.amazon.com/)
2. Search for **CloudTrail** in the top search bar and click it
3. Click **Trails** in the left sidebar
4. You should see **workshop-audit-trail** with status **Logging**

> **💡 CloudTrail events take 5–15 minutes to appear.** If you check the event history later and do not see recent events, that is normal. Keep going with the lab — the events will appear by the time you reach the auditor testing section.

---

### Step 7: Write the Trust Policy

Before creating roles, you need a **trust policy** that defines who can assume them. This policy allows any identity in your account to assume the roles.

**Step 7a:** In the VS Code file tree, click the **New File** icon and name the file `role-trust-policy.json`.

**Step 7b:** 📋 Copy and paste this entire block into it, **replacing `<YOUR_ACCOUNT_ID>`**:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowAccountToAssume",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<YOUR_ACCOUNT_ID>:root"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

> **🔄 Example:** If your account ID is `123456789012`:
> ```json
> "AWS": "arn:aws:iam::123456789012:root"
> ```

**Step 7c:** **Save** the file (**Ctrl+S** / **Cmd+S**). You should see `role-trust-policy.json` appear in the file tree.

> **⚠️ Common mistakes:** Make sure you replaced `<YOUR_ACCOUNT_ID>` with your 12-digit account number. The word `root` here does NOT mean the root user — it means "any authenticated identity in this account."

> **What does this file do?** It tells AWS: "Any authenticated user or role in this account is allowed to assume this role." In a real environment, you would restrict this to specific users or groups.

---

### Step 8: Create the Developer Role

The developer role allows S3 and Lambda access but blocks IAM modifications (preventing privilege escalation).

**Step 8a: Write the developer policy**

In the VS Code file tree, create a **New File** named `developer-policy.json`. 📋 Copy and paste this entire block into it:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowDeveloperActions",
            "Effect": "Allow",
            "Action": [
                "s3:*",
                "lambda:InvokeFunction"
            ],
            "Resource": "*"
        },
        {
            "Sid": "DenyIAMModifications",
            "Effect": "Deny",
            "Action": [
                "iam:CreateUser",
                "iam:DeleteUser",
                "iam:CreateRole",
                "iam:DeleteRole",
                "iam:AttachUserPolicy",
                "iam:AttachRolePolicy",
                "iam:PutUserPolicy",
                "iam:PutRolePolicy",
                "iam:CreateAccessKey"
            ],
            "Resource": "*"
        }
    ]
}
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

> **What does this file do?** It gives the developer:
> - **Full S3 access** — create buckets, upload files, read data
> - **Lambda invoke** — run serverless functions
> - **Explicit Deny on IAM** — cannot create users, roles, or access keys (prevents privilege escalation)

---

**Step 8b: Create the developer role**

📋 Copy and paste:

```
aws iam create-role --role-name workshop-developer-role --assume-role-policy-document file://role-trust-policy.json
```

**✅ You should see** JSON output with `"RoleName": "workshop-developer-role"`.

**Step 8c: Attach the developer policy to the role**

📋 Copy and paste:

```
aws iam put-role-policy --role-name workshop-developer-role --policy-name DeveloperPolicy --policy-document file://developer-policy.json
```

**✅ No output means success.**

---

### Step 9: Create the Auditor Role

The auditor role allows reading CloudTrail logs and S3 data but blocks ALL modifications.

**Step 9a: Write the auditor policy**

In the VS Code file tree, create a **New File** named `auditor-policy.json`. 📋 Copy and paste this entire block into it:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowAuditReadAccess",
            "Effect": "Allow",
            "Action": [
                "cloudtrail:LookupEvents",
                "cloudtrail:GetTrailStatus",
                "cloudtrail:DescribeTrails",
                "cloudtrail:ListTrails",
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": "*"
        },
        {
            "Sid": "DenyAllModifications",
            "Effect": "Deny",
            "Action": [
                "s3:PutObject",
                "s3:DeleteObject",
                "s3:DeleteBucket",
                "iam:*",
                "lambda:*",
                "ec2:*"
            ],
            "Resource": "*"
        }
    ]
}
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

> **What does this file do?** It gives the auditor:
> - **CloudTrail read access** — can look up events, check trail status, and review logs
> - **S3 read access** — can read the log files stored in the CloudTrail bucket
> - **Explicit Deny on modifications** — cannot write to S3, cannot touch IAM, Lambda, or EC2
>
> This is **separation of duties** — the auditor can see everything but change nothing.

**Step 9b: Create the auditor role**

📋 Copy and paste:

```
aws iam create-role --role-name workshop-auditor-role --assume-role-policy-document file://role-trust-policy.json
```

**✅ You should see** JSON output with `"RoleName": "workshop-auditor-role"`.

**Step 9c: Attach the auditor policy to the role**

📋 Copy and paste:

```
aws iam put-role-policy --role-name workshop-auditor-role --policy-name AuditorPolicy --policy-document file://auditor-policy.json
```

**✅ No output means success.**

---

### Step 10: Assume the Developer Role

Now you will "become" the developer by assuming the developer role. This gives you temporary credentials with the developer's permissions.

**Step 10a: Assume the role**

📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>`**:

```
aws sts assume-role --role-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-developer-role --role-session-name dev-session
```

**✅ You should see** JSON output containing temporary credentials:

```json
{
    "Credentials": {
        "AccessKeyId": "ASIA...",
        "SecretAccessKey": "...",
        "SessionToken": "...(very long string)...",
        "Expiration": "2026-..."
    },
    "AssumedRoleUser": {
        "Arn": "arn:aws:sts::<ACCOUNT_ID>:assumed-role/workshop-developer-role/dev-session"
    }
}
```

> **📝 Write down these THREE values from the output:**
>
> - **AccessKeyId:** ______________________________
> - **SecretAccessKey:** ______________________________
> - **SessionToken:** ______________________________ (this is very long — copy the entire thing)

**Step 10b: Set the developer credentials**

**Windows (PowerShell):**

📋 Copy and paste, **replacing the placeholders** with the values from Step 10a:

```powershell
$env:AWS_ACCESS_KEY_ID="<ACCESS_KEY_ID_FROM_STEP_10A>"
$env:AWS_SECRET_ACCESS_KEY="<SECRET_ACCESS_KEY_FROM_STEP_10A>"
$env:AWS_SESSION_TOKEN="<SESSION_TOKEN_FROM_STEP_10A>"
$env:AWS_DEFAULT_REGION="us-east-1"
Remove-Item Env:\AWS_PROFILE
```

**macOS / Linux:**

📋 Copy and paste, **replacing the placeholders** with the values from Step 10a:

```bash
export AWS_ACCESS_KEY_ID="<ACCESS_KEY_ID_FROM_STEP_10A>"
export AWS_SECRET_ACCESS_KEY="<SECRET_ACCESS_KEY_FROM_STEP_10A>"
export AWS_SESSION_TOKEN="<SESSION_TOKEN_FROM_STEP_10A>"
export AWS_DEFAULT_REGION="us-east-1"
unset AWS_PROFILE
```

> **💡 What is the Session Token?** When you assume a role, you get THREE credentials instead of two. The session token is an extra security measure that proves these are temporary credentials from a role assumption. It expires (usually after 1 hour).

**Step 10c: Verify you are the developer**

📋 Copy and paste:

```
aws sts get-caller-identity
```

**✅ You should see** output showing `workshop-developer-role/dev-session` — confirming you are now operating as the developer.

---

**Step 10d: Test S3 write — should WORK ✅**

The developer has full S3 access. Let's prove it by uploading a file to the CloudTrail logs bucket.

**Windows (PowerShell):**

📋 Copy and paste, **replacing `<YOUR_BUCKET_NAME>`**:

```powershell
"developer was here" | Out-File -Encoding utf8 dev-test.txt
aws s3 cp dev-test.txt s3://<YOUR_BUCKET_NAME>/dev-test.txt
```

**macOS / Linux:**

📋 Copy and paste, **replacing `<YOUR_BUCKET_NAME>`**:

```bash
echo "developer was here" > dev-test.txt
aws s3 cp dev-test.txt s3://<YOUR_BUCKET_NAME>/dev-test.txt
```

**✅ You should see:**

```
upload: ./dev-test.txt to s3://<YOUR_BUCKET_NAME>/dev-test.txt
```

The developer CAN write to S3. This action is being recorded by CloudTrail.

---

**Step 10e: Test privilege escalation — should be DENIED ❌**

Now try to create a new IAM user. A malicious developer might try this to create a backdoor account.

📋 Copy and paste:

```
aws iam create-user --user-name hacker-attempt 2>&1
```

**✅ You should see an error:** `An error occurred (AccessDenied)`. The developer CANNOT create IAM users. **Privilege escalation is blocked!**

> **💡 Why this matters:** If a developer's credentials are stolen, the attacker cannot create new admin users, attach new policies, or escalate their access. The explicit Deny on IAM actions prevents this entire class of attack.

---

**Step 10f: Try to turn off the audit trail — should be DENIED ❌**

A developer trying to hide their activity might attempt to stop CloudTrail from recording. Try it:

📋 Copy and paste:

```
aws cloudtrail stop-logging --name workshop-audit-trail 2>&1
```

**✅ You should see an error:** `An error occurred (AccessDenied)`. The developer role has **no CloudTrail permissions**, so it cannot turn logging off. Every action it takes stays on the record — **it cannot cover its tracks.**

> **⚠️ A caveat worth knowing:** the developer role here grants `s3:*`, which technically still allows deleting log *objects* from the CloudTrail bucket. In a real environment you would harden the log bucket separately — a bucket policy that denies deletes, S3 Object Lock or MFA-delete, CloudTrail log-file validation, or storing logs in a separate, locked-down account. Turning **logging itself** off, though, is firmly blocked here.

---

### Step 11: Switch Back to Admin (Between Role Tests)

**⚠️ Important:** You must clear ALL credentials (including the session token) before assuming the next role.

**Windows (PowerShell):**

📋 Copy and paste, **replacing `<YOUR_PROFILE_NAME>`**:

```powershell
Remove-Item Env:\AWS_ACCESS_KEY_ID
Remove-Item Env:\AWS_SECRET_ACCESS_KEY
Remove-Item Env:\AWS_SESSION_TOKEN
Remove-Item Env:\AWS_DEFAULT_REGION
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**macOS / Linux:**

📋 Copy and paste, **replacing `<YOUR_PROFILE_NAME>`**:

```bash
unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
unset AWS_SESSION_TOKEN
unset AWS_DEFAULT_REGION
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**Verify you are back to admin:**

📋 Copy and paste:

```
aws sts get-caller-identity
```

**✅ You should see** your admin role (with `AdministratorAccess` in the ARN). If you still see the developer role, close your terminal and open a new one, then set your profile again.

---

### Step 12: Assume the Auditor Role

Now you will "become" the auditor and review what the developer did.

**Step 12a: Assume the role**

📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>`**:

```
aws sts assume-role --role-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-auditor-role --role-session-name auditor-session
```

**✅ You should see** JSON output with temporary credentials (AccessKeyId, SecretAccessKey, SessionToken).

> **📝 Write down the THREE values** (AccessKeyId, SecretAccessKey, SessionToken) from the output.

**Step 12b: Set the auditor credentials**

**Windows (PowerShell):**

📋 Copy and paste, **replacing the placeholders** with the values from Step 12a:

```powershell
$env:AWS_ACCESS_KEY_ID="<ACCESS_KEY_ID_FROM_STEP_12A>"
$env:AWS_SECRET_ACCESS_KEY="<SECRET_ACCESS_KEY_FROM_STEP_12A>"
$env:AWS_SESSION_TOKEN="<SESSION_TOKEN_FROM_STEP_12A>"
$env:AWS_DEFAULT_REGION="us-east-1"
Remove-Item Env:\AWS_PROFILE
```

**macOS / Linux:**

📋 Copy and paste, **replacing the placeholders** with the values from Step 12a:

```bash
export AWS_ACCESS_KEY_ID="<ACCESS_KEY_ID_FROM_STEP_12A>"
export AWS_SECRET_ACCESS_KEY="<SECRET_ACCESS_KEY_FROM_STEP_12A>"
export AWS_SESSION_TOKEN="<SESSION_TOKEN_FROM_STEP_12A>"
export AWS_DEFAULT_REGION="us-east-1"
unset AWS_PROFILE
```

**Step 12c: Verify you are the auditor**

📋 Copy and paste:

```
aws sts get-caller-identity
```

**✅ You should see** output showing `workshop-auditor-role/auditor-session`.

---

**Step 12d: Query CloudTrail — should WORK ✅**

Now look at the audit trail. This shows recent API calls in your account.

📋 Copy and paste:

```
aws cloudtrail lookup-events --max-results 5 --query "Events[].{Time:EventTime,Name:EventName,User:Username}" --output table
```

**✅ You should see** a table of recent events, including the developer's activity — `PutObject` (the file upload), `CreateUser` (the privilege-escalation attempt), and the `AssumeRole` calls. Note that this summary table shows **that** each action happened, but **not** whether it succeeded or failed — you will confirm the failure next.

> **💡 If the table is empty or shows very few events:** CloudTrail events take 5–15 minutes to appear. If you just completed the developer section, wait a few minutes and try again. This is normal behavior — CloudTrail is not real-time.

> **💡 What you are seeing:** Every action in your AWS account is logged — successful AND failed. The table above proves the developer's `CreateUser` attempt was *recorded*. But to see that it was *denied*, you have to open the event's full detail, where the error code lives.

**Now confirm the CreateUser attempt was denied.** 📋 Copy and paste:

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateUser --max-results 1 --query "Events[0].CloudTrailEvent" --output text
```

This prints the full CloudTrail record for the most recent `CreateUser` event as one long block of JSON. Read through it and look for these two fields:

```
"errorCode": "AccessDenied",
"errorMessage": "User: arn:aws:sts::...:assumed-role/workshop-developer-role/dev-session is not authorized to perform: iam:CreateUser ... with an explicit deny ..."
```

**✅ That `errorCode` is the proof.** CloudTrail recorded not just *that* the developer tried to create a user, but that the attempt was **blocked** — and exactly why. This is what a security team looks for when investigating suspicious activity.

> **💡 Why wasn't the failure in the first table?** The summary table only projects a few top-level fields (event name, time, user). Whether an action succeeded or failed lives inside each event's full record — the `CloudTrailEvent` field — which is why this second command dumps the whole thing.

> **💡 If the command returns nothing:** the `CreateUser` event may not have propagated yet (CloudTrail lag of 5–15 minutes). Wait a few minutes and try again.

---

**Step 12e: Try to tamper with the evidence — should be DENIED ❌**

The auditor can see everything but must not be able to change anything — including destroying evidence. Try to delete the file the developer uploaded earlier.

📋 Copy and paste, **replacing `<YOUR_BUCKET_NAME>`**:

```
aws s3 rm s3://<YOUR_BUCKET_NAME>/dev-test.txt 2>&1
```

**✅ You should see an error:** `delete failed` with `An error occurred (AccessDenied)`. The auditor can *read* the data and the logs, but the explicit Deny blocks any write or delete — so it **cannot alter or destroy evidence**. This is the other half of separation of duties: **the investigator can look, but never touch.**

---

### Step 13: Switch Back to Admin (Final)

**⚠️ Important:** Clear ALL credentials and restore your admin profile before cleanup.

**Windows (PowerShell):**

📋 Copy and paste, **replacing `<YOUR_PROFILE_NAME>`**:

```powershell
Remove-Item Env:\AWS_ACCESS_KEY_ID
Remove-Item Env:\AWS_SECRET_ACCESS_KEY
Remove-Item Env:\AWS_SESSION_TOKEN
Remove-Item Env:\AWS_DEFAULT_REGION
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**macOS / Linux:**

📋 Copy and paste, **replacing `<YOUR_PROFILE_NAME>`**:

```bash
unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
unset AWS_SESSION_TOKEN
unset AWS_DEFAULT_REGION
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**Verify you are back to admin:**

📋 Copy and paste:

```
aws sts get-caller-identity
```

**✅ You should see** your admin role (with `AdministratorAccess` in the ARN).

---

### Step 14: Console Checkpoint — CloudTrail Event History

Let's see the audit trail in the console:

1. Go to [https://console.aws.amazon.com/](https://console.aws.amazon.com/)
2. Search for **CloudTrail** in the top search bar and click it
3. Click **Event history** in the left sidebar
4. You should see a list of recent events in your account
5. Look for events like:
   - `AssumeRole` — when you switched to the developer and auditor roles
   - `PutObject` — when the developer uploaded a file
   - `CreateUser` — the failed privilege escalation attempt (look for "Access denied" in the error code)

**✅ Checkpoint:** You can see every action that was taken in your account, who did it, and when. This is exactly how security teams investigate breaches and prove compliance.

> **💡 If you do not see all events:** CloudTrail events can take up to 15 minutes to appear. If some events are missing, that is normal. The important thing is that you understand the concept — every action is logged.

---

## What You Just Did

You built a complete security architecture with three key components:

1. **Audit logging (CloudTrail)** — every action in your account is recorded permanently
2. **Role-based access control** — engineers assume specific roles with limited permissions
3. **Separation of duties** — developers can build but not modify security; auditors can investigate but not change anything
4. **Privilege escalation prevention** — explicit Deny on IAM actions blocks backdoor creation

This is how real organizations operate:
- **Developers** get temporary credentials to deploy code, but cannot touch security settings
- **Auditors** can review all activity, but cannot modify anything (preventing evidence tampering)
- **CloudTrail** ensures accountability — no action goes unrecorded

---

## Cert Prep Callout

**Target Certification:** AWS Security Specialty (SCS-C02)

This lab covers several high-weight exam topics:
- **CloudTrail configuration:** Trail creation, S3 bucket policies for CloudTrail, multi-region trails
- **IAM roles and trust policies:** How assume-role works, temporary credentials, session tokens
- **Separation of duties:** Designing role structures that prevent privilege escalation
- **Incident investigation:** Using CloudTrail to determine who did what and when

**Sample question type:** "A security team needs to investigate unauthorized access to an S3 bucket. Which AWS service provides a record of all API calls made to the bucket, including the identity of the caller and the time of the call?"  
**Answer:** AWS CloudTrail

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `An error occurred (InsufficientS3BucketPolicyException)` when creating the trail | The bucket policy does not grant CloudTrail permission to write | Open `cloudtrail-bucket-policy.json` and verify: (1) bucket name is correct in all 2 places, (2) account ID is correct, (3) the policy was applied with `put-bucket-policy` |
| `An error occurred (TrailAlreadyExistsException)` | The trail already exists from a previous attempt | Delete it first: `aws cloudtrail delete-trail --name workshop-audit-trail` then try again |
| `assume-role` returns an error about permissions | Your admin profile may not have permission to assume roles | Verify you are using your admin profile (`get-caller-identity` should show AdministratorAccess) |
| `get-caller-identity` still shows the previous role after switching | Env vars were not fully cleared | Make sure you cleared ALL THREE: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, AND `AWS_SESSION_TOKEN`. Close and reopen your terminal if needed. |
| CloudTrail `lookup-events` returns empty results | Events take 5–15 minutes to appear | Wait 10–15 minutes and try again. This is normal CloudTrail behavior. |
| `The security token included in the request is expired` | Your temporary role credentials expired (1 hour limit) | Switch back to admin (Step 11 or 13) and assume the role again |
| `An error occurred (MalformedPolicyDocument)` | JSON syntax error in one of your policy files | Check for missing commas, brackets, or quotes. Verify all placeholders were replaced. |
| S3 upload fails as developer | The developer policy should allow S3 writes | Verify you are using the developer role credentials (check `get-caller-identity`) |

---

## Cleanup

**⚠️ Important:** Always clean up resources after completing a lab. Follow these steps in order. Make sure you are using your **admin credentials** (not a role).

### Step 1: Stop CloudTrail Logging

📋 Copy and paste:

```
aws cloudtrail stop-logging --name workshop-audit-trail
```

**✅ No output means success.**

### Step 2: Delete the CloudTrail Trail

📋 Copy and paste:

```
aws cloudtrail delete-trail --name workshop-audit-trail
```

**✅ No output means success.**

### Step 3: Delete the Developer Role Policy and Role

📋 Copy and paste these commands one at a time:

```
aws iam delete-role-policy --role-name workshop-developer-role --policy-name DeveloperPolicy
```

```
aws iam delete-role --role-name workshop-developer-role
```

**✅ No output means success for each command.**

### Step 4: Delete the Auditor Role Policy and Role

📋 Copy and paste these commands one at a time:

```
aws iam delete-role-policy --role-name workshop-auditor-role --policy-name AuditorPolicy
```

```
aws iam delete-role --role-name workshop-auditor-role
```

**✅ No output means success for each command.**

### Step 5: Empty and Delete the S3 Bucket

📋 Copy and paste, **replacing `<YOUR_BUCKET_NAME>`**:

```
aws s3 rm s3://<YOUR_BUCKET_NAME> --recursive
```

```
aws s3 rb s3://<YOUR_BUCKET_NAME>
```

**✅ You should see** `remove_bucket: <YOUR_BUCKET_NAME>`.

### Step 6: Delete Local Files

> **⚠️ Close VS Code first.** If VS Code still has the `workshop-lab-3c` folder open, the delete will fail — especially on Windows. Choose **File → Close Folder** or quit VS Code before running the commands below.

Remove the project folder:

**Windows (PowerShell):**

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-3c
```

**macOS / Linux:**

```bash
cd ~/Desktop
rm -rf ~/Desktop/workshop-lab-3c
```

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** you received (copy and paste it)
4. Your **operating system** (Windows, Mac, or Linux)
