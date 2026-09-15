# Lab 5C: Build Automated Incident Response with Lambda

**Session:** 5 — Incident Response on AWS  
**Track:** Cloud Security & IR  
**Difficulty:** Advanced  
**Estimated Time:** 55–70 minutes  
**Target Cert:** AWS Security Specialty

---

## Overview

In this lab, you will:

1. **Check for an existing CloudTrail trail** — confirm or create one so EventBridge can receive API call events
2. **Create a Lambda function** that automatically deactivates newly created access keys
3. **Test the function manually** — prove it works by invoking it with a simulated event
4. **Wire up EventBridge** — create a rule that triggers the Lambda whenever a new access key is created
5. **Understand automated remediation** — how companies enforce "no long-lived credentials" policies

By the end of this lab, you will have built a real automated incident response system. This is **SOAR** (Security Orchestration, Automation, and Response) — the same pattern used by enterprise security teams to respond to threats in seconds instead of hours.

---

## Prerequisites

- ✅ Completed **Lab 1A** (AWS CLI installed and configured)
- ✅ Completed **Lab 5A** (understand credential revocation)
- ✅ AWS CLI authenticated — run `aws sts get-caller-identity` and confirm it returns your account info
- ✅ **VS Code** installed (from Lab 1B) — any text editor works, but these labs assume VS Code

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| IAM | Identity and Access Management | Always Free |
| AWS Lambda | Serverless compute | Free: 1M requests + 400,000 GB-seconds per month |
| Amazon EventBridge | Event routing | Free for AWS service events |
| AWS CloudTrail | API activity logging | Free for one management event trail per region |
| Amazon S3 | Storage for CloudTrail logs | ~$0.00 for the tiny amount used in this lab |

**Estimated cost for this lab: $0.00**

---

## Concepts

**SOAR (Security Orchestration, Automation, and Response)** is the practice of automating security operations. Instead of a human manually deactivating compromised credentials, a system detects the threat and responds automatically — in seconds, not hours.

**Automated Remediation** means fixing security issues without human intervention. Examples:
- Automatically deactivating access keys when they are created (enforce SSO-only)
- Automatically isolating EC2 instances when GuardDuty detects compromise
- Automatically revoking overly permissive security group rules

**EventBridge** is AWS's event bus. It routes events from AWS services to targets like Lambda functions.

**Why CloudTrail is required for EventBridge:** The AWS Console's CloudTrail Event History shows the last 90 days of API calls and is always on — but it does **not** feed events into EventBridge. For EventBridge to receive `"AWS API Call via CloudTrail"` events, you must have an active CloudTrail *trail* configured. Without a trail, EventBridge never sees the events and your Lambda will never be triggered, even though the event appears in the console.

**The Full Pipeline:**
```
Someone creates an access key
       ↓ (seconds)
CloudTrail trail records the event
       ↓ (1–5 minutes)
EventBridge matches the event pattern
       ↓
Lambda function is triggered
       ↓
Access key is automatically deactivated
```

---

## ⚠️ Reminder: How to Read Commands in This Lab

Commands are inside gray code boxes. **📋 Copy and paste** them into your terminal.

**Placeholders** look like `<THIS>` — replace them (including the `<` and `>`) with your actual values.

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name from Lab 1A | `AdministratorAccess-123456789012` |
| `<ACCOUNT_ID>` | Your 12-digit AWS account ID | `123456789012` |
| `<LAMBDA_ARN>` | The ARN of your Lambda function (Step 8) | `arn:aws:lambda:us-east-1:123456789012:function:workshop-key-revoker` |
| `<RULE_ARN>` | The ARN of your EventBridge rule (Step 11) | `arn:aws:events:us-east-1:123456789012:rule/workshop-auto-revoke` |
| `<TEST_KEY_ID>` | The Access Key ID of the test user (Step 9) | `AKIAIOSFODNN7EXAMPLE` |
| `<TRAIL_BUCKET_NAME>` | A globally unique S3 bucket name for CloudTrail logs | `jdoe-lab5c-cloudtrail-logs` |

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

> **📝 Note your Account ID** (the 12-digit number): ______________________________

---

### Step 2: Create Your Project Folder

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
mkdir ~\Desktop\workshop-lab-5c
cd ~\Desktop\workshop-lab-5c
```

**macOS / Linux:**

```bash
mkdir ~/Desktop/workshop-lab-5c
cd ~/Desktop/workshop-lab-5c
```

Verify you are in the right folder:

```
pwd
```

**✅ You should see** a path ending in `workshop-lab-5c`.

> **💡 Save ALL files you create in this lab to this folder.**

**Open the folder in VS Code.** 📋 Copy and paste:

```
code .
```

> This opens VS Code with `workshop-lab-5c` as its **file tree**, so the JSON files and the Lambda code you create land in the right place. (You set up the `code` command in Lab 1B — if you see `'code' is not recognized`, close and reopen your terminal, or revisit Lab 1B, Step 6.)

---

### Step 3: Verify (or Create) the Security-Track Trail

> **Why this step matters:** EventBridge only receives `"AWS API Call via CloudTrail"` events when an active CloudTrail **trail** exists — Event History alone does NOT feed EventBridge. This lab reuses the **`workshop-security-trail`** you created back in Lab 3C.

📋 Check whether it's running:

```
aws cloudtrail get-trail-status --name workshop-security-trail --query "IsLogging"
```

**✅ If you see `true`** — your security-track trail is active. **Skip Steps 4 and 5** and continue to **Step 6**.

**If you get a `TrailNotFoundException` (or `false`)** — you either skipped Lab 3C or already tore the trail down. Do **Steps 4 and 5** below to create it, then carry on.

> **💡 Already have a *different* trail from another project?** Any active multi-region trail will feed EventBridge, so you can leave it and skip Steps 4–5 — just note that a second active trail beyond your first incurs a small cost.

---

### Step 4: (If needed) Create an S3 Bucket for CloudTrail Logs

> **Only do Steps 4–5 if Step 3 showed you have no active trail.** If `workshop-security-trail` is already logging, skip to **Step 6**.

CloudTrail requires an S3 bucket to store log files. You will create a dedicated bucket for the trail.

**Step 4a: Create the bucket**:

```
aws s3api create-bucket --bucket <TRAIL_BUCKET_NAME> --region us-east-1
```

**✅ You should see** JSON with the bucket location.

> **What does this do?** Creates a private S3 bucket where CloudTrail will write its log files. The bucket name must be globally unique — you will get an error if the bucket name already exists.

**Step 4b: Create the bucket policy**

In the VS Code file tree, create a **New File** named `bucket-policy.json`.

📋 Copy and paste this entire block into it, **replacing `<TRAIL_BUCKET_NAME>` in both places and `<ACCOUNT_ID>` in one place**:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AWSCloudTrailAclCheck",
            "Effect": "Allow",
            "Principal": {"Service": "cloudtrail.amazonaws.com"},
            "Action": "s3:GetBucketAcl",
            "Resource": "arn:aws:s3:::<TRAIL_BUCKET_NAME>"
        },
        {
            "Sid": "AWSCloudTrailWrite",
            "Effect": "Allow",
            "Principal": {"Service": "cloudtrail.amazonaws.com"},
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::<TRAIL_BUCKET_NAME>/AWSLogs/<ACCOUNT_ID>/*",
            "Condition": {
                "StringEquals": {"s3:x-amz-acl": "bucket-owner-full-control"}
            }
        }
    ]
}
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

**Step 4c: Apply the bucket policy** (replace `<TRAIL_BUCKET_NAME>`):

```
aws s3api put-bucket-policy --bucket <TRAIL_BUCKET_NAME> --policy file://bucket-policy.json
```

**✅ No output means success.**

> **What does this do?** Grants the CloudTrail service permission to check and write to your bucket. Without this policy, CloudTrail will refuse to use the bucket.

---

### Step 5: Create and Start the CloudTrail Trail

**Step 5a: Create the trail** (replace `<TRAIL_BUCKET_NAME>`):

```
aws cloudtrail create-trail --name workshop-security-trail --s3-bucket-name <TRAIL_BUCKET_NAME> --is-multi-region-trail
```

**✅ You should see** JSON with the trail details including a `TrailARN`.

> **What does `--is-multi-region-trail` do?** Makes the trail capture API calls from all AWS regions, not just us-east-1. Since IAM is a global service, its events appear in us-east-1 — but the multi-region flag ensures nothing is missed.

**Step 5b: Start logging**

📋 Copy and paste:

```
aws cloudtrail start-logging --name workshop-security-trail
```

**✅ No output means success.**

**Step 5c: Confirm logging is active**

📋 Copy and paste:

```
aws cloudtrail get-trail-status --name workshop-security-trail --region us-east-1
```

**✅ You should see** `"IsLogging": true`.

> **🎉 EventBridge can now receive CloudTrail events.** Any API call in your account will flow through the pipeline: CloudTrail trail → EventBridge → Lambda.

---

### Step 6: Create the Lambda IAM Role

The Lambda function needs permission to deactivate access keys. You will create an IAM role that grants this.

**Step 6a: Create the trust policy**

In the VS Code file tree, create a **New File** named `trust-policy.json`.

📋 Copy and paste this entire block into it:

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

**Step 6b: Create the IAM role and attach the policy**

📋 Copy and paste:

```
aws iam create-role --role-name workshop-key-revoker-role --assume-role-policy-document file://trust-policy.json
```

**✅ You should see** JSON output with the role details including an ARN.

**Step 6c: Attach the IAMFullAccess policy**

📋 Copy and paste:

```
aws iam attach-role-policy --role-name workshop-key-revoker-role --policy-arn arn:aws:iam::aws:policy/IAMFullAccess
```

**✅ No output means success.**

> **⚠️ In production**, you would use a least-privilege policy that only allows `iam:UpdateAccessKey`. We use `IAMFullAccess` here for simplicity in a lab environment.

**Step 6d: Attach the Lambda basic execution policy**

📋 Copy and paste:

```
aws iam attach-role-policy --role-name workshop-key-revoker-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

**✅ No output means success.**

> **What does this do?** Allows the Lambda function to write logs to CloudWatch so you can see its output and debug errors.

---

### Step 7: Write the Lambda Function

**Step 7a:** In the VS Code file tree, create a **New File** named `revoke_key.py`.

**Step 7b:** 📋 Copy and paste this entire Python function into it:

```python
import json
import boto3

iam_client = boto3.client('iam')

def lambda_handler(event, context):
    # Extract the username and access key ID from the CloudTrail event
    detail = event.get('detail', {})
    request_params = detail.get('requestParameters', {})
    response_elements = detail.get('responseElements', {})

    username = request_params.get('userName', 'unknown')

    # The access key ID is in the response elements
    access_key_info = response_elements.get('accessKey', {})
    access_key_id = access_key_info.get('accessKeyId', '')

    if not access_key_id:
        print(f"No access key ID found in event for user {username}")
        return {'statusCode': 400, 'body': 'No access key ID found'}

    # Deactivate the access key
    try:
        iam_client.update_access_key(
            UserName=username,
            AccessKeyId=access_key_id,
            Status='Inactive'
        )
        message = f"SECURITY ALERT: Deactivated access key {access_key_id} for user {username}"
        print(message)
        return {'statusCode': 200, 'body': message}
    except Exception as e:
        error_msg = f"Failed to deactivate key {access_key_id} for user {username}: {str(e)}"
        print(error_msg)
        return {'statusCode': 500, 'body': error_msg}
```

**Step 7c:** **Save** the file (**Ctrl+S** / **Cmd+S**).

---

### Step 8: Package and Deploy the Lambda Function

**Step 8a: Zip the function code**

**Windows (PowerShell):**

```powershell
Compress-Archive -Path revoke_key.py -DestinationPath revoke_key.zip -Force
```

**macOS / Linux:**

```bash
zip revoke_key.zip revoke_key.py
```

**Step 8b: Deploy the Lambda function** (replace `<ACCOUNT_ID>`):

```
aws lambda create-function --function-name workshop-key-revoker --runtime python3.12 --role arn:aws:iam::<ACCOUNT_ID>:role/workshop-key-revoker-role --handler revoke_key.lambda_handler --zip-file fileb://revoke_key.zip --region us-east-1 --timeout 30
```

**✅ You should see** JSON output with a `FunctionArn`.

> **📝 Write down the FunctionArn:** ______________________________
>
> It looks like: `arn:aws:lambda:us-east-1:<ACCOUNT_ID>:function:workshop-key-revoker`

> **💡 If you get a role error:** Wait 10 seconds and try again — IAM role propagation can take a few seconds.

---

### Step 9: Test the Lambda Function (Manual Invoke)

Before wiring up EventBridge, prove the Lambda works by invoking it manually with a simulated event.

**Step 9a: Create a test IAM user**

```
aws iam create-user --user-name auto-revoke-test
```

**Step 9b: Create an access key for the test user**

```
aws iam create-access-key --user-name auto-revoke-test
```

**✅ You should see** JSON with `AccessKeyId` and `SecretAccessKey`.

> **📝 Write down the AccessKeyId:** ______________________________

**Step 9c: Verify the key is Active**

```
aws iam list-access-keys --user-name auto-revoke-test --query "AccessKeyMetadata[].{Id:AccessKeyId,Status:Status}" --output table
```

**Step 9d: Create the test event**

In the VS Code file tree, create a **New File** named `test-event.json`.

📋 Copy and paste this entire block into it, **replacing `<TEST_KEY_ID>`** with the AccessKeyId from Step 9b:

```json
{
    "detail": {
        "eventName": "CreateAccessKey",
        "requestParameters": {
            "userName": "auto-revoke-test"
        },
        "responseElements": {
            "accessKey": {
                "accessKeyId": "<TEST_KEY_ID>",
                "status": "Active",
                "userName": "auto-revoke-test"
            }
        }
    }
}
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

**Step 9e: Invoke the Lambda function**

```
aws lambda invoke --function-name workshop-key-revoker --payload file://test-event.json --cli-binary-format raw-in-base64-out response.json --region us-east-1
```

**✅ You should see** `"StatusCode": 200`.

**Step 9f: Check the response**

**Windows (PowerShell):**

```powershell
Get-Content response.json
```

**macOS / Linux:**

```bash
cat response.json
```

**✅ You should see:**

```json
{"statusCode": 200, "body": "SECURITY ALERT: Deactivated access key <TEST_KEY_ID> for user auto-revoke-test"}
```

**Step 9g: Verify the key is now Inactive**

```
aws iam list-access-keys --user-name auto-revoke-test --query "AccessKeyMetadata[].{Id:AccessKeyId,Status:Status}" --output table
```

**✅ You should see** Status `Inactive`.

> **🎉 The Lambda function works!** This proves your automated remediation logic is correct.

---

### Step 10: Delete the Test Key Before Wiring Up EventBridge

> **⚠️ Important:** Delete the test user's deactivated key before continuing. The next step will make the pipeline live — any key created after that will be automatically deactivated.

**Delete the inactive test key** (replace `<TEST_KEY_ID>`):

```
aws iam delete-access-key --user-name auto-revoke-test --access-key-id <TEST_KEY_ID>
```

**✅ No output means success.**

---

### Step 11: Wire Up EventBridge (Automation Layer)

**Step 11a: Create the event pattern**

In the VS Code file tree, create a **New File** named `eventbridge-pattern.json`.

📋 Copy and paste this into it:

```json
{
    "source": ["aws.iam"],
    "detail-type": ["AWS API Call via CloudTrail"],
    "detail": {
        "eventSource": ["iam.amazonaws.com"],
        "eventName": ["CreateAccessKey"]
    }
}
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

**Step 11b: Create the EventBridge rule**

```
aws events put-rule --name workshop-auto-revoke --event-pattern file://eventbridge-pattern.json --state ENABLED --region us-east-1
```

**✅ You should see** a `RuleArn` in the output.

> **📝 Write down the RuleArn:** ______________________________

**Step 11c: Add Lambda's FunctionArn as the target** (replace `<LAMBDA_ARN>`):

```
aws events put-targets --rule workshop-auto-revoke --targets "Id=lambda-target,Arn=<LAMBDA_ARN>" --region us-east-1
```

**✅ You should see** `"FailedEntryCount": 0`.

**Step 11d: Grant EventBridge permission to invoke Lambda** (replace `<RULE_ARN>`):

```
aws lambda add-permission --function-name workshop-key-revoker --statement-id eventbridge-invoke --action lambda:InvokeFunction --principal events.amazonaws.com --source-arn <RULE_ARN> --region us-east-1
```

**✅ You should see** JSON with a `Statement` field.

---

### Step 12: Trigger the Automated Pipeline

Now create a new access key for your test user and watch the full pipeline run automatically.

**Step 12a: Create a new access key**

```
aws iam create-access-key --user-name auto-revoke-test
```

> **📝 Write down this new AccessKeyId:** ______________________________

**Step 12b: Wait 1–5 minutes**, then check whether the key was automatically deactivated:

```
aws iam list-access-keys --user-name auto-revoke-test --query "AccessKeyMetadata[].{Id:AccessKeyId,Status:Status}" --output table
```

**✅ You should see** the key with Status `Inactive` — deactivated automatically by Lambda.

**Step 12c: Check the Lambda logs to see the execution**

```
aws logs tail /aws/lambda/workshop-key-revoker --region us-east-1 --since 10m
```

**✅ You should see** a log line like:

```
SECURITY ALERT: Deactivated access key AKIA... for user auto-revoke-test
```

> **🎉 The full automated pipeline is working!** CloudTrail recorded the key creation, EventBridge matched it, and Lambda deactivated the key — all without human intervention.

---

### Step 13: Understanding What You Built

```
Someone creates an access key
       ↓ (seconds)
CloudTrail trail records the event
       ↓ (1–5 minutes)
EventBridge matches the event pattern
       ↓ (milliseconds)
Lambda function is triggered
       ↓ (milliseconds)
Access key is automatically deactivated
```

**In production, this means:**
- No human needs to be awake or available
- Response happens in minutes, not hours
- Every new access key is automatically deactivated
- Users are forced to use SSO/roles instead of long-lived keys

---

### Step 14: Console Checkpoint

1. Go to [https://console.aws.amazon.com/](https://console.aws.amazon.com/)
2. Search for **Lambda** → click `workshop-key-revoker` → check the **Monitor** tab for invocation metrics
3. Search for **CloudTrail** → click **Trails** → confirm `workshop-security-trail` is active
4. Search for **EventBridge** → click **Rules** → confirm `workshop-auto-revoke` is Enabled with the Lambda target

---

## What You Just Did

1. **Confirmed or created a CloudTrail trail** — the missing piece that enables EventBridge to receive API call events
2. **Created a Lambda function** that deactivates access keys on demand
3. **Tested it manually** — proved the logic works with a simulated event
4. **Wired up EventBridge** — connected live CloudTrail events to automatic Lambda execution
5. **Watched the full pipeline run** — created a real key and saw it auto-deactivated

---

## Cert Prep Callout

**Target Certification:** AWS Security Specialty (SCS-C02)

Key facts the exam tests on this pattern:
- CloudTrail Event History does **not** send events to EventBridge — an active trail is required
- EventBridge + Lambda is the standard automated remediation pattern
- CloudTrail → EventBridge delay is typically 1–5 minutes for management events
- SOAR replaces human response time (30–60 min) with automated response (1–5 min)

**Sample question:** "A company wants to automatically disable any newly created IAM access keys. Their EventBridge rule is configured correctly but Lambda is never triggered. What is the most likely cause?"  
**Answer:** No active CloudTrail trail is configured. CloudTrail Event History does not send events to EventBridge.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| Lambda never triggers (no CloudWatch logs) | No active CloudTrail trail | Complete Steps 4–5 to create and start a trail |
| `IsLogging: false` on the trail | Trail exists but logging was stopped | Run `aws cloudtrail start-logging --name workshop-security-trail` |
| Lambda invoke returns `statusCode: 400` | Test event JSON is malformed or missing key ID | Verify you replaced `<TEST_KEY_ID>` in `test-event.json` |
| Lambda invoke returns `statusCode: 500` | Lambda role lacks IAM permissions | Verify you attached `IAMFullAccess` in Step 6c |
| `put-targets` returns `FailedEntryCount: 1` | Lambda ARN is wrong | Double-check the ARN from Step 8b |
| `add-permission` error: statement already exists | You already ran this command | The permission exists — this is fine, continue |
| Key still Active after 10 minutes | Pipeline not fully connected | Check CloudWatch logs: `aws logs tail /aws/lambda/workshop-key-revoker --region us-east-1 --since 15m` |

---

## Cleanup

**⚠️ Follow this cleanup order exactly.** Disable EventBridge first to prevent the Lambda from interfering with keys you create during cleanup.

> **🔗 Two parts:** the **Cleanup** steps below remove this lab's own resources (EventBridge rule, Lambda, IAM role, test user). The shared **`workshop-security-trail`** backbone is removed separately in the **Final Track Teardown** at the very end — run that only when you're done with the whole security track.

### Cleanup Step 1: Remove EventBridge Targets

```
aws events remove-targets --rule workshop-auto-revoke --ids lambda-target --region us-east-1
```

### Cleanup Step 2: Delete the EventBridge Rule

```
aws events delete-rule --name workshop-auto-revoke --region us-east-1
```

**✅ No output means success.**

### Cleanup Step 3: Delete the Lambda Function

```
aws lambda delete-function --function-name workshop-key-revoker --region us-east-1
```

**✅ You should see** JSON with a `"StatusCode": 204` which means the command was executed successfully.

### Cleanup Step 4: Detach Policies and Delete the IAM Role

```
aws iam detach-role-policy --role-name workshop-key-revoker-role --policy-arn arn:aws:iam::aws:policy/IAMFullAccess
```

```
aws iam detach-role-policy --role-name workshop-key-revoker-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

```
aws iam delete-role --role-name workshop-key-revoker-role
```
**✅ No output for each of these 3 commands means success.**

### Cleanup Step 5: Delete the Test User and Access Keys

First, list any remaining keys (replace if needed):

```
aws iam list-access-keys --user-name auto-revoke-test
```

Delete each key listed (replace `<KEY_ID>`):

```
aws iam delete-access-key --user-name auto-revoke-test --access-key-id <KEY_ID>
```

Delete the user:

```
aws iam delete-user --user-name auto-revoke-test
```

### Cleanup Step 6: Delete Local Files

> **⚠️ Close VS Code first.** If VS Code still has the `workshop-lab-5c` folder open, the delete will fail — especially on Windows. Choose **File → Close Folder** or quit VS Code before running the commands below.

**Windows (PowerShell):**

```powershell
cd ~
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-5c
```

**macOS / Linux:**

```bash
cd ~
rm -rf ~/Desktop/workshop-lab-5c
```

---

## Final Track Teardown

> **🔗 Do this only when you are finished with the entire Cloud Security track (Sessions 3–5).** These commands remove the **shared audit backbone** — the `workshop-security-trail` CloudTrail trail, its CloudWatch Log Group and IAM role (added in Lab 5B), and the S3 log bucket — that Labs 3C, 5B, and 5C all relied on. (Labs 3C and 5B told you to leave these running and tear them down here.)

**1. Stop and delete the trail:**

```
aws cloudtrail stop-logging --name workshop-security-trail
```

```
aws cloudtrail delete-trail --name workshop-security-trail
```

Verify it's gone (you should see `"trailList": []`):

```
aws cloudtrail describe-trails --region us-east-1
```

**2. Delete the CloudWatch Log Group** (created in Lab 5B — skip if you never did 5B):

```
aws logs delete-log-group --log-group-name security-track-cloudtrail-logs --region us-east-1
```

**3. Delete the CloudTrail → CloudWatch IAM role** (created in Lab 5B — skip if you never did 5B):

```
aws iam delete-role-policy --role-name security-track-cloudtrail-role --policy-name CloudWatchLogsWrite
```

```
aws iam delete-role --role-name security-track-cloudtrail-role
```

**4. Empty and delete the log bucket** (replace `<TRAIL_BUCKET_NAME>`):

```
aws s3 rm s3://<TRAIL_BUCKET_NAME> --recursive
```

```
aws s3api delete-bucket --bucket <TRAIL_BUCKET_NAME> --region us-east-1
```

**✅ Your security-track audit backbone is fully removed.** Nothing from Sessions 3–5 is left running.

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran
3. The **full error message** you received
4. Your **operating system** (Windows, Mac, or Linux)
