# Lab 6A: Architecting S3 for Security & Reliability

**Session:** 6 — AWS Well-Architected Framework  
**Track:** Solutions Architecture  
**Difficulty:** Beginner  
**Estimated Time:** 30–35 minutes  
**Target Cert:** AWS Solutions Architect – Associate (SAA)

---

## Overview

In this lab, you will play the role of a **Solutions Architect** who must secure an S3 bucket containing confidential HR data. Your job is to apply two pillars of the **AWS Well-Architected Framework**:

- **Security pillar** — control WHO can access your data (least-privilege access)
- **Reliability pillar** — protect data from accidental loss (versioning)

The key idea of this lab is the **feedback loop**: for each problem, you will first SEE the bad state, then apply the fix, then VERIFY the good state. You learn *why* each configuration matters by experiencing the consequence of not having it.

**The scenario:** Your company stores confidential salary data in an S3 bucket. Two teams use Lambda functions to access company data: the **HR team** (who should see salary data) and the **Analytics team** (who should NOT). Right now, both teams can read the file. Your job: restrict access so only authorized roles can read the confidential data, and protect against accidental deletion.

---

## Prerequisites

- ✅ Completed **Lab 1A** (AWS CLI installed and configured)
- ✅ AWS CLI authenticated — run `aws sts get-caller-identity` and confirm it returns your account info
- ✅ **VS Code** installed (from Lab 1B) — any text editor works, but these labs assume VS Code

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| Amazon S3 | Cloud storage | $0.023 per GB/month, $0.0005 per 1,000 PUT/GET requests |
| AWS Lambda | Serverless compute | Always Free: 1M requests + 400,000 GB-seconds/month |
| AWS IAM | Identity and access management | Always Free |

**Estimated cost for this lab: $0.00**

---

## Concepts

Before we start, here is what each concept means:

**AWS Well-Architected Framework (WAF)** is a set of best practices for building good cloud systems. It is organized into six "pillars." This lab focuses on two of them:

- **Security pillar** — "Protect data, systems, and assets." This includes controlling who can access your data using the principle of **least privilege** — give each role only the permissions it needs, nothing more.
- **Reliability pillar** — "Ensure a workload performs its function correctly and recovers from failure." This includes protecting against accidental data loss.

> **💡 The WAF is a review *process*, not a one-time checklist.** Each pillar is a set of questions you ask about a workload — "who can access this?", "what happens if this fails?", "what does this cost?" — so you can spot risks and prioritize fixes. AWS provides a free console tool, the **AWS Well-Architected Tool**, that walks you through those questions for a real workload and produces an improvement plan. Session 6 has you practice the core loop by hand: **see a risk → fix it → verify the improvement**, one pillar at a time.

**Least Privilege** means giving each user, role, or service the absolute minimum permissions needed to do its job. If the Analytics team doesn't need salary data, they shouldn't be able to read it — even if they have general S3 read access.

**Bucket Policy** is a JSON document attached to an S3 bucket that controls who can access the files. You can use it to explicitly **Deny** access to certain roles, or **Allow** access only to specific roles. An explicit Deny always wins over an Allow.

**IAM Role** is an identity in AWS that a service (like Lambda) assumes to get permissions. Each Lambda function runs under a specific role. By controlling what the bucket policy allows for each role, you control which functions can access which data.

**Versioning** is an S3 feature that keeps a copy of every version of a file. When ON, deleting a file does not actually remove it — it hides it behind a "delete marker," and you can recover the original. When OFF, a delete is permanent.

---

## ⚠️ Reminder: How to Read Commands in This Lab

Commands are inside gray code boxes. **📋 Copy and paste** them into your terminal.

**Placeholders** look like `<THIS>` — replace them (including the `<` and `>`) with your actual values.

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name from Lab 1A | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account number | `123456789012` |
| `<YOUR_UNIQUE_BUCKET_NAME>` | A globally unique S3 bucket name you choose | `jane-doe-waf-lab6a` |

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

**✅ You should see** your account ID and role. If you get an error about an expired token, run `aws sso login --profile <YOUR_PROFILE_NAME>` first.

> **📝 Write down your Account ID** (the 12-digit number). You will need it later.

---

### Step 2: Create Your Project Folder

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
mkdir ~\Desktop\workshop-lab-6a
cd ~\Desktop\workshop-lab-6a
```

**macOS / Linux:**

📋 Copy and paste:

```bash
mkdir ~/Desktop/workshop-lab-6a
cd ~/Desktop/workshop-lab-6a
```

**Verify:** `pwd` should show a path ending in `workshop-lab-6a`.

> **💡 From now on, save ALL files you create in this lab to this folder.**

**Open the folder in VS Code.** 📋 Copy and paste:

```
code .
```

> This opens VS Code with `workshop-lab-6a` as its **file tree**, so the JSON files and the Lambda code you create land in the right place. (You set up the `code` command in Lab 1B — if you see `'code' is not recognized`, close and reopen your terminal, or revisit Lab 1B, Step 6.)

---

### Step 3: Save Your Bucket Name as a Variable

To make this lab easier, you will save your bucket name **once**. Every command after this will use it automatically.

**Windows (PowerShell):**

📋 Copy and paste this command, **replacing `<YOUR_UNIQUE_BUCKET_NAME>`** with a unique name (lowercase, numbers, and hyphens only):

```powershell
$BUCKET="<YOUR_UNIQUE_BUCKET_NAME>"
```

**macOS / Linux:**

📋 Copy and paste, **replacing `<YOUR_UNIQUE_BUCKET_NAME>`**:

```bash
BUCKET="<YOUR_UNIQUE_BUCKET_NAME>"
```

**Verify the variable is set:**

**Windows (PowerShell):**
```powershell
echo $BUCKET
```

**macOS / Linux:**
```bash
echo $BUCKET
```

**✅ You should see** the bucket name you chose printed back to you.

> **⚠️ Important:** If you close your terminal at any point during this lab, you must set this variable again (repeat this step). The variable only lasts for the current terminal session.

> **💡 Naming rules:** Bucket names must be 3–63 characters, globally unique, lowercase letters/numbers/hyphens only. If you get a `BucketAlreadyExists` error later, someone else has that name — pick a different one and redo this step.

---

### Step 4: Create the S3 Bucket and Upload the Confidential File

**Step 4a: Create the bucket**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3 mb s3://$BUCKET --region us-east-1
```

**macOS / Linux:**
```bash
aws s3 mb s3://$BUCKET --region us-east-1
```

**✅ You should see:** `make_bucket: <your bucket name>`

**Step 4b: Create the confidential file**

In the VS Code file tree, create a **New File** named `employee-salaries.txt`. 📋 Copy and paste this into it:

```
CONFIDENTIAL - HR Department
Employee Salary Report Q4 2025

Name            | Role              | Annual Salary
----------------|-------------------|-------------
Jane Smith      | Cloud Engineer    | $125,000
John Doe        | Security Analyst  | $115,000
Alex Johnson    | DevOps Engineer   | $130,000

This document is classified INTERNAL ONLY.
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

**Step 4c: Upload it to your bucket**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3 cp employee-salaries.txt s3://$BUCKET/employee-salaries.txt
```

**macOS / Linux:**
```bash
aws s3 cp employee-salaries.txt s3://$BUCKET/employee-salaries.txt
```

**✅ You should see:** `upload: ./employee-salaries.txt to s3://<your bucket>/employee-salaries.txt`

---

### Step 5: Create Two IAM Roles (HR Team and Analytics Team)

You will create two Lambda functions that represent two teams:
- **HR Team** — authorized to read salary data
- **Analytics Team** — NOT authorized to read salary data

Each team's Lambda function runs under its own IAM role. Right now, both roles will have S3 read access (the "bad" state).

**Step 5a: Create the trust policy**

In the VS Code file tree, create a **New File** named `trust-policy.json`. 📋 Copy and paste this entire block into it:

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

> **What does this file do?** It tells AWS "Lambda functions are allowed to use this role."

**Step 5b: Create the HR Team role**

📋 Copy and paste:

```
aws iam create-role --role-name workshop-hr-team-role --assume-role-policy-document file://trust-policy.json
```

**✅ You should see** JSON output with the role details.

**Step 5c: Create the Analytics Team role**

📋 Copy and paste:

```
aws iam create-role --role-name workshop-analytics-team-role --assume-role-policy-document file://trust-policy.json
```

**✅ You should see** JSON output with the role details.

**Step 5d: Give BOTH roles S3 read access and Lambda logging**

📋 Copy and paste these four commands one at a time:

```
aws iam attach-role-policy --role-name workshop-hr-team-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

```
aws iam attach-role-policy --role-name workshop-hr-team-role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

```
aws iam attach-role-policy --role-name workshop-analytics-team-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

```
aws iam attach-role-policy --role-name workshop-analytics-team-role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

**✅ No output means success** for each command.

> **What did you just do?** Both roles now have `AmazonS3ReadOnlyAccess` — meaning both can read ANY S3 bucket in the account. This is the **overly permissive** state. The Analytics team should NOT be able to read HR salary data, but right now they can.

**⏳ Wait 10 seconds** before continuing. IAM roles take a moment to become usable.

---

### Step 6: Deploy Two Lambda Functions (One Per Team)

Both functions run the same code — they try to read the salary file from S3. The only difference is which IAM role they use.

**Step 6a: Create the Lambda function code**

In the VS Code file tree, create a **New File** named `s3_reader.py`. 📋 Copy and paste this entire code block into it:

```python
import json
import boto3

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    bucket = event.get('bucket')
    key = event.get('key', 'employee-salaries.txt')

    try:
        response = s3.get_object(Bucket=bucket, Key=key)
        content = response['Body'].read().decode('utf-8')
        return {
            'statusCode': 200,
            'body': f"ACCESS GRANTED - File contents:\n{content}"
        }
    except Exception as e:
        return {
            'statusCode': 403,
            'body': f"ACCESS DENIED - {str(e)}"
        }
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

> **What does this function do?** It tries to read a file from S3. If it succeeds, it returns the file contents with "ACCESS GRANTED." If it fails (because of a Deny policy), it returns "ACCESS DENIED" with the error message.

**Step 6b: Zip the code**

**Windows (PowerShell):**
```powershell
Compress-Archive -Path s3_reader.py -DestinationPath s3reader.zip -Force
```

**macOS / Linux:**
```bash
zip s3reader.zip s3_reader.py
```

**Step 6c: Deploy the HR Team function**

📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>`**:

**Windows (PowerShell):**
```powershell
aws lambda create-function --function-name workshop-hr-reader --runtime python3.12 --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-hr-team-role" --handler s3_reader.lambda_handler --zip-file fileb://s3reader.zip --timeout 10 --region us-east-1
```

**macOS / Linux:**
```bash
aws lambda create-function \
    --function-name workshop-hr-reader \
    --runtime python3.12 \
    --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-hr-team-role" \
    --handler s3_reader.lambda_handler \
    --zip-file fileb://s3reader.zip \
    --timeout 10 \
    --region us-east-1
```

**✅ You should see** JSON output with the function details.

**Step 6d: Deploy the Analytics Team function**

📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>`**:

**Windows (PowerShell):**
```powershell
aws lambda create-function --function-name workshop-analytics-reader --runtime python3.12 --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-analytics-team-role" --handler s3_reader.lambda_handler --zip-file fileb://s3reader.zip --timeout 10 --region us-east-1
```

**macOS / Linux:**
```bash
aws lambda create-function \
    --function-name workshop-analytics-reader \
    --runtime python3.12 \
    --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-analytics-team-role" \
    --handler s3_reader.lambda_handler \
    --zip-file fileb://s3reader.zip \
    --timeout 10 \
    --region us-east-1
```

**✅ You should see** JSON output with the function details.

> **💡 If you get an error** about the role not being found: wait 10 more seconds and try again.

---

## PART 1 — The Security Pillar

> **🔗 You've already been building this pillar.** The entire **Security track (Sessions 3–5)** — least-privilege IAM, explicit Deny beating Allow, CloudTrail, GuardDuty — was the Security pillar in practice. This lab adds one more layer: control at the **data layer** (a policy on the bucket itself), so access doesn't depend on IAM permissions alone.

### Step 7: 🚨 SEE THE PROBLEM — Both Teams Can Read Confidential Data

Let's prove that the Analytics team (who should NOT see salary data) can read it.

**Step 7a: Create the test payload**

In the VS Code file tree, create a **New File** named `read-payload.json`. 📋 Copy and paste this into it, **replacing `<YOUR_UNIQUE_BUCKET_NAME>`** with your actual bucket name:

```json
{"bucket": "<YOUR_UNIQUE_BUCKET_NAME>", "key": "employee-salaries.txt"}
```

For example, if your bucket is `jane-doe-waf-lab6a`:
```json
{"bucket": "jane-doe-waf-lab6a", "key": "employee-salaries.txt"}
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

**Step 7b: Test the HR Team function**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-hr-reader --payload file://read-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 hr-response.json; Write-Output "=== HR TEAM RESULT ===" (Get-Content hr-response.json | ConvertFrom-Json).body
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-hr-reader --payload file://read-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 hr-response.json; echo "=== HR TEAM RESULT ==="; cat hr-response.json
```

**✅ You should see:**

```
=== HR TEAM RESULT ===
ACCESS GRANTED - File contents:
CONFIDENTIAL - HR Department
Employee Salary Report Q4 2025
...
```

Good — the HR team SHOULD be able to read this. ✅

**Step 7c: Test the Analytics Team function**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-analytics-reader --payload file://read-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 analytics-response.json; Write-Output "=== ANALYTICS TEAM RESULT ===" (Get-Content analytics-response.json | ConvertFrom-Json).body
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-analytics-reader --payload file://read-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 analytics-response.json; echo "=== ANALYTICS TEAM RESULT ==="; cat analytics-response.json
```

**✅ You should see:**

```
=== ANALYTICS TEAM RESULT ===
ACCESS GRANTED - File contents:
CONFIDENTIAL - HR Department
Employee Salary Report Q4 2025
...
```

> **🚨 The Analytics team can read the full salary report!** They have no business need to see employee compensation data. This is a **Security pillar violation** — access is not restricted to least privilege. Anyone with generic S3 read access can see confidential HR data.

---

### Step 8: ✅ FIX THE PROBLEM — Apply a Restrictive Bucket Policy

You will add a bucket policy that **explicitly denies** access to everyone EXCEPT the HR team role (and your admin role, so you can still manage the bucket).

**Step 8a: Create the restrictive bucket policy**

In the VS Code file tree, create a **New File** named `restrict-policy.json`. 📋 Copy and paste this into it, **replacing `<YOUR_UNIQUE_BUCKET_NAME>` with your bucket name** and **replacing `<YOUR_ACCOUNT_ID>` with your 12-digit account ID** (in both places):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenyNonHRAccess",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::<YOUR_UNIQUE_BUCKET_NAME>/*",
            "Condition": {
                "StringNotLike": {
                    "aws:PrincipalArn": [
                        "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-hr-team-role",
                        "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/AWSReservedSSO_AdministratorAccess_*"
                    ]
                }
            }
        }
    ]
}
```

**Replace in 2 places:**
1. Replace `<YOUR_UNIQUE_BUCKET_NAME>` with your bucket name (e.g., `jane-doe-waf-lab6a`)
2. Replace `<YOUR_ACCOUNT_ID>` with your account ID (e.g., `123456789012`) — this appears **twice** in the file

**Save** the file (**Ctrl+S** / **Cmd+S**).

> **⚠️ Common mistakes:** Make sure you replaced ALL instances of `<YOUR_UNIQUE_BUCKET_NAME>` (1 place) and `<YOUR_ACCOUNT_ID>` (2 places). The account ID must be exactly 12 digits with no dashes.

> **What does this policy do?** Let's break it down:
>
> | Field | Value | What It Means |
> |-------|-------|---------------|
> | `Effect` | `Deny` | This rule **blocks** access |
> | `Principal` | `*` | Applies to **everyone** |
> | `Action` | `s3:GetObject` | Blocks reading/downloading files |
> | `Condition: StringNotLike` | HR role + admin role | **EXCEPT** these two roles |
>
> **In plain English:** "Deny everyone from reading files in this bucket, UNLESS they are the HR team role or an admin."
>
> The key insight: an explicit **Deny always wins** over an Allow in AWS. Even though the Analytics team has `AmazonS3ReadOnlyAccess` (an Allow), this bucket-level Deny overrides it. This is how you enforce least privilege at the data layer.

**Step 8b: Apply the policy to your bucket**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3api put-bucket-policy --bucket $BUCKET --policy file://restrict-policy.json
```

**macOS / Linux:**
```bash
aws s3api put-bucket-policy --bucket $BUCKET --policy file://restrict-policy.json
```

**✅ No output means success.** The bucket now enforces least-privilege access.

---

### Step 9: ✅ VERIFY THE FIX — Only the HR Team Can Read the File

**Step 9a: Test the HR Team (should still work)**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-hr-reader --payload file://read-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 hr-response2.json; Write-Output "=== HR TEAM (AFTER POLICY) ===" (Get-Content hr-response2.json | ConvertFrom-Json).statusCode
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-hr-reader --payload file://read-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 hr-response2.json; echo "=== HR TEAM (AFTER POLICY) ==="; cat hr-response2.json
```

**✅ You should see:** `200` — ACCESS GRANTED. The HR team can still read the file because they are in the exception list.

**Step 9b: Test the Analytics Team (should be BLOCKED)**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-analytics-reader --payload file://read-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 analytics-response2.json; Write-Output "=== ANALYTICS TEAM (AFTER POLICY) ===" (Get-Content analytics-response2.json | ConvertFrom-Json).body
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-analytics-reader --payload file://read-payload.json --cli-binary-format raw-in-base64-out --region us-east-1 analytics-response2.json; echo "=== ANALYTICS TEAM (AFTER POLICY) ==="; cat analytics-response2.json
```

**✅ You should see:**

```
=== ANALYTICS TEAM (AFTER POLICY) ===
ACCESS DENIED - An error occurred (AccessDenied) when calling the GetObject operation...
```

> **🎉 Security pillar achieved!** The same bucket, the same file, but now access is controlled by identity:
>
> | Team | Before Policy | After Policy |
> |------|--------------|--------------|
> | HR Team | ✅ ACCESS GRANTED | ✅ ACCESS GRANTED |
> | Analytics Team | ⚠️ ACCESS GRANTED | 🚫 ACCESS DENIED |
>
> The Analytics team still has `AmazonS3ReadOnlyAccess` — but the explicit Deny in the bucket policy overrides it. This is **least privilege enforced at the data layer.**

> **💡 Key insight:** In Lab 1C, you learned about public vs. private access. But real-world security is more nuanced — it's not just "can the internet see it?" but "which specific roles within your own company can see it?" A bucket policy with Conditions lets you answer that question precisely.

> **💡 In production you'd go further.** Access control is one of three core Security-pillar controls for S3. You'd also encrypt data **at rest** (`SSE-KMS`) and require encrypted **transport** (deny requests where `aws:SecureTransport` is false). This lab covers the first; the other two are single-policy additions built on the same idea.

---

## PART 2 — The Reliability Pillar

Now you will see why **versioning** matters for the Reliability pillar. First you will experience permanent data loss (no versioning), then you will see how versioning protects you.

> **💡 Versioning is *one* reliability tool, not the whole pillar.** Reliability also covers running across multiple Availability Zones, automated backups and recovery, and designing to a target recovery time (RTO) and recovery point (RPO). Here you focus on protecting a single object from accidental loss — the simplest, most common reliability win.

### Step 10: 🚨 SEE THE PROBLEM — Delete Without Versioning

Right now your bucket has NO versioning. Let's see what happens when a file is accidentally deleted.

**Step 10a: Confirm the file is there**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3 ls s3://$BUCKET/
```

**macOS / Linux:**
```bash
aws s3 ls s3://$BUCKET/
```

**✅ You should see** `employee-salaries.txt` listed.

**Step 10b: Delete the file (simulating an accident)**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3 rm s3://$BUCKET/employee-salaries.txt
```

**macOS / Linux:**
```bash
aws s3 rm s3://$BUCKET/employee-salaries.txt
```

**✅ You should see:** `delete: s3://<your bucket>/employee-salaries.txt`

**Step 10c: Confirm it is gone**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3 ls s3://$BUCKET/
```

**macOS / Linux:**
```bash
aws s3 ls s3://$BUCKET/
```

**✅ You should see** nothing — no output at all. The bucket is empty.

> **🚨 The data is gone forever.** There is no way to recover it. If that salary report was the only copy, it is lost permanently. This breaks the Reliability pillar — your system did not protect against accidental data loss.

---

### Step 11: ✅ FIX THE PROBLEM — Enable Versioning

**Step 11a: Enable versioning on the bucket**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3api put-bucket-versioning --bucket $BUCKET --versioning-configuration Status=Enabled
```

**macOS / Linux:**
```bash
aws s3api put-bucket-versioning --bucket $BUCKET --versioning-configuration Status=Enabled
```

**✅ No output means success.**

**Step 11b: Verify versioning is on**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3api get-bucket-versioning --bucket $BUCKET --query "Status" --output text
```

**macOS / Linux:**
```bash
aws s3api get-bucket-versioning --bucket $BUCKET --query "Status" --output text
```

**✅ You should see:** `Enabled`

**Step 11c: Re-upload the file (since it was lost)**

You still have your local copy in the project folder.

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3 cp employee-salaries.txt s3://$BUCKET/employee-salaries.txt
```

**macOS / Linux:**
```bash
aws s3 cp employee-salaries.txt s3://$BUCKET/employee-salaries.txt
```

**✅ You should see** the upload confirmation.

---

### Step 12: ✅ VERIFY THE FIX — Delete and Recover

Now let's prove that with versioning ON, a deleted file can be recovered.

**Step 12a: Delete the file (again)**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3 rm s3://$BUCKET/employee-salaries.txt
```

**macOS / Linux:**
```bash
aws s3 rm s3://$BUCKET/employee-salaries.txt
```

**✅ You should see** the delete confirmation.

**Step 12b: Confirm the normal listing shows it gone**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3 ls s3://$BUCKET/
```

**macOS / Linux:**
```bash
aws s3 ls s3://$BUCKET/
```

**✅ You should see** nothing — the file appears gone, just like before.

**Step 12c: But look at the VERSIONS — the file is still there!**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3api list-object-versions --bucket $BUCKET --query "Versions[].{Key:Key,VersionId:VersionId}" --output table
```

**macOS / Linux:**
```bash
aws s3api list-object-versions --bucket $BUCKET --query "Versions[].{Key:Key,VersionId:VersionId}" --output table
```

**✅ You should see** a table with `employee-salaries.txt` and a `VersionId` (a long string of letters and numbers).

> **📝 Copy the VersionId** from the table — you will need it in the next step. It looks something like `BWxsoyftUEzIL_d8dzJ28l.h89dXsV4d`.

**Step 12d: ✅ Console Checkpoint**

1. Go to your bucket in the S3 Console
2. The bucket looks empty
3. **Toggle the "Show versions" switch** at the top of the object list (turn it ON)
4. **Now you can see the file and its versions**, including a "delete marker"

> **💡 What is a delete marker?** When versioning is on, deleting a file just adds a "delete marker" on top — it hides the file but keeps the actual data underneath. That is why you can recover it.

**Step 12e: Recover the file**

📋 Copy and paste this command, **replacing `<VERSION_ID>`** with the VersionId you copied in Step 12c:

**Windows (PowerShell):**
```powershell
aws s3api copy-object --bucket $BUCKET --copy-source "$BUCKET/employee-salaries.txt?versionId=<VERSION_ID>" --key employee-salaries.txt
```

**macOS / Linux:**
```bash
aws s3api copy-object --bucket $BUCKET --copy-source "$BUCKET/employee-salaries.txt?versionId=<VERSION_ID>" --key employee-salaries.txt
```

**✅ You should see** JSON output with a `CopyObjectResult` and a `LastModified` timestamp.

**Step 12f: Confirm the file is back**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3 ls s3://$BUCKET/
```

**macOS / Linux:**
```bash
aws s3 ls s3://$BUCKET/
```

**✅ You should see** `employee-salaries.txt` listed again. **You recovered the "lost" file!**

> **🎉 Reliability pillar achieved!** Without versioning, the delete in Step 10 was permanent. With versioning, the delete in Step 12 was recoverable. This is how real systems protect against accidental deletion, ransomware, and human error.

---

## What You Just Did

You acted as a Solutions Architect and applied two Well-Architected Framework pillars to a real S3 bucket:

| Pillar | Problem You Saw | Fix You Applied | How You Verified |
|--------|----------------|-----------------|------------------|
| **Security** | Both teams (HR + Analytics) could read confidential salary data | Bucket policy with explicit Deny + Condition restricting to HR role only | HR: ACCESS GRANTED, Analytics: ACCESS DENIED |
| **Reliability** | Deleted file gone forever (no versioning) | Enabled versioning | Deleted file recovered from version history |

**Key takeaways:**
- **Least privilege is not just about IAM policies** — bucket policies with Conditions let you enforce access control at the data layer. An explicit Deny always wins over an Allow.
- **Versioning is your safety net** — it costs almost nothing but protects against the most common data loss scenarios (accidental delete, overwrites, ransomware).
- **The Well-Architected Framework is practical** — each pillar produces a real, measurable improvement you can demonstrate.

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

The SAA exam tests these concepts heavily:
- S3 bucket policies with Conditions (least privilege)
- The relationship between IAM policies and resource-based policies (explicit Deny wins)
- S3 versioning and how delete markers work (Reliability/Durability)
- The Well-Architected Framework pillars and how to apply them

**Sample question type:** "An S3 bucket has sensitive data. Users have AmazonS3ReadOnlyAccess but only one team should access the bucket. What is the MOST effective way to restrict access?"  
**Answer:** Add a bucket policy with an explicit Deny and a Condition that excludes the authorized role (StringNotLike on aws:PrincipalArn). Explicit Deny overrides the IAM Allow.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `$BUCKET` commands fail with "bucket name required" | The variable was lost (you closed the terminal) | Repeat Step 3 to set the `$BUCKET` variable again |
| `BucketAlreadyExists` | Someone else has that bucket name | Pick a new name and redo Step 3, then Step 4 |
| `create-function` fails with "role cannot be assumed" | IAM role hasn't propagated yet | Wait 10 seconds and try again |
| Analytics team still gets ACCESS GRANTED after policy | The bucket policy may have a typo | Open `restrict-policy.json` and verify: bucket name, account ID (appears twice), and that both are replaced |
| HR team gets ACCESS DENIED after policy | The HR role ARN in the policy doesn't match | Verify the role name in the policy is exactly `workshop-hr-team-role` |
| `MalformedPolicy` error | JSON syntax is wrong | Check for missing commas, brackets, or unreplaced placeholders |
| `list-object-versions` shows nothing | Versioning was not enabled before the delete | Versioning only protects deletes that happen AFTER it is enabled. Make sure you did Step 11 before Step 12. |

---

## Cleanup

**⚠️ Important:** A versioned bucket cannot be deleted until ALL versions and delete markers are removed. Follow these steps carefully.

### Step 1: Delete the Lambda Functions

📋 Copy and paste these two commands:

```
aws lambda delete-function --function-name workshop-hr-reader --region us-east-1
```

```
aws lambda delete-function --function-name workshop-analytics-reader --region us-east-1
```

**✅ Output with StatusCode: 204 means success for each.**

### Step 2: Delete the Lambda Log Groups

📋 Copy and paste these two commands:

```
aws logs delete-log-group --log-group-name /aws/lambda/workshop-hr-reader --region us-east-1
```

```
aws logs delete-log-group --log-group-name /aws/lambda/workshop-analytics-reader --region us-east-1
```

**✅ No output means success for each.**

### Step 3: Delete the IAM Roles

📋 Copy and paste these six commands one at a time:

```
aws iam detach-role-policy --role-name workshop-hr-team-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

```
aws iam detach-role-policy --role-name workshop-hr-team-role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

```
aws iam delete-role --role-name workshop-hr-team-role
```

```
aws iam detach-role-policy --role-name workshop-analytics-team-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

```
aws iam detach-role-policy --role-name workshop-analytics-team-role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

```
aws iam delete-role --role-name workshop-analytics-team-role
```

**✅ No output means success for each.**

### Step 4: Empty and Delete the S3 Bucket

A versioned bucket keeps hidden versions and delete markers. You must remove all of them.

**Step 4a: List everything in the bucket**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3api list-object-versions --bucket $BUCKET --query "[Versions[].{Key:Key,VersionId:VersionId},DeleteMarkers[].{Key:Key,VersionId:VersionId}]" --output table
```

**macOS / Linux:**
```bash
aws s3api list-object-versions --bucket $BUCKET --query "[Versions[].{Key:Key,VersionId:VersionId},DeleteMarkers[].{Key:Key,VersionId:VersionId}]" --output table
```

**✅ You should see** a table listing VersionIds. Copy each one.

**Step 4b: Delete each version**

For **each** `VersionId` from the table above, run this command, **replacing `<VERSION_ID>`**:

📋 Copy and paste (run once per VersionId):

**Windows (PowerShell):**
```powershell
aws s3api delete-object --bucket $BUCKET --key employee-salaries.txt --version-id <VERSION_ID>
```

**macOS / Linux:**
```bash
aws s3api delete-object --bucket $BUCKET --key employee-salaries.txt --version-id <VERSION_ID>
```

**✅ Each command** returns a small JSON confirmation.

> **💡 Tip (macOS / Linux only):** You can delete everything in one command instead:
> ```bash
> aws s3api delete-objects --bucket $BUCKET --delete "$(aws s3api list-object-versions --bucket $BUCKET --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}' --output json)"
> aws s3api delete-objects --bucket $BUCKET --delete "$(aws s3api list-object-versions --bucket $BUCKET --query '{Objects: DeleteMarkers[].{Key:Key,VersionId:VersionId}}' --output json)"
> ```

**Step 4c: Verify the bucket is empty**

📋 Copy and paste:

```
aws s3api list-object-versions --bucket $BUCKET --query "[Versions,DeleteMarkers]" --output json
```

**✅ You should see** `[null,null]` or empty lists.

**Step 4d: Delete the bucket policy (required before deletion)**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3api delete-bucket-policy --bucket $BUCKET
```

**macOS / Linux:**
```bash
aws s3api delete-bucket-policy --bucket $BUCKET
```

**✅ No output means success.**

**Step 4e: Delete the bucket**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3 rb s3://$BUCKET
```

**macOS / Linux:**
```bash
aws s3 rb s3://$BUCKET
```

**✅ You should see:** `remove_bucket: <your bucket>`

### Step 5: Delete Local Files

> **⚠️ Close VS Code first.** If VS Code still has the `workshop-lab-6a` folder open, the delete will fail — especially on Windows. Choose **File → Close Folder** or quit VS Code before running the commands below.

**Windows (PowerShell):**
```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-6a
```

**macOS / Linux:**
```bash
cd ~/Desktop
rm -rf ~/Desktop/workshop-lab-6a
```

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** you received (copy and paste it)
4. Your **operating system** (Windows, Mac, or Linux)
