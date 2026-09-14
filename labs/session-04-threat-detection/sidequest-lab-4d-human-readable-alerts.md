# Lab 4D: Human-Readable GuardDuty Alerts with Lambda

**Session:** 4 — Threat Detection & AWS Security Services  
**Track:** Cloud Security & IR  
**Difficulty:** Advanced  
**Estimated Time:** 45–55 minutes  
**Target Cert:** AWS Security Specialty

---

## Overview

In Lab 4C, you built an automated alerting pipeline where EventBridge routed GuardDuty findings directly to SNS — which delivered raw JSON to your inbox. That JSON is accurate, but it is hard to act on under pressure. In this lab, you will insert a **Lambda function** between EventBridge and SNS to transform that raw JSON into a clear, structured security alert with severity labels, affected resource details, recommended actions, and a direct link to the GuardDuty console.

**Architecture change:**

| Lab 4C | Lab 4D |
|--------|--------|
| GuardDuty → EventBridge → SNS (raw JSON email) | GuardDuty → EventBridge → Lambda → SNS (formatted alert email) |

In this lab, you will:

1. **Write a Lambda function** — Python code that parses a GuardDuty finding and formats it into a human-readable alert
2. **Create an IAM role for Lambda** — attach a custom SNS publish policy and an AWS managed policy for logging
3. **Deploy the Lambda function** — package and upload the code to AWS
4. **Rewire EventBridge** — replace the SNS target with your Lambda function
5. **Test the pipeline** — generate a sample finding and verify you receive a formatted alert email

---

## Prerequisites

- ✅ Completed **Lab 4C** — you need an active GuardDuty detector, SNS topic with confirmed email subscription, and EventBridge rule
- ✅ AWS CLI authenticated — run `aws sts get-caller-identity` to confirm
- ✅ **VS Code** installed (from Lab 1B) — any text editor works, but these labs assume VS Code

> **💡 If you cleaned up Lab 4C:** Re-run Steps 3–11 of Lab 4C to recreate the GuardDuty detector, SNS topic, email subscription, and EventBridge rule before continuing here. You do not need to test the pipeline again — just have those resources in place.

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| AWS Lambda | Serverless function execution | Free — 1M requests/month on the free tier |
| Amazon CloudWatch Logs | Lambda execution logs | Free — first 5 GB of log ingestion/month |
| Amazon SNS | Notification delivery | Free within 1,000 email notifications/month |
| Amazon GuardDuty | Threat detection | Covered by your 30-day free trial |

**Estimated cost for this lab: $0.00**

---

## Concepts

Before we start, here is what each new concept means:

**AWS Lambda** is a serverless compute service — you upload code and AWS runs it in response to events, with no servers to manage. Lambda functions can be triggered by EventBridge, API Gateway, S3, and dozens of other AWS services. You pay only when your function runs (and the first 1M invocations per month are free).

**IAM Execution Role** is the identity your Lambda function uses when it runs. Because Lambda runs as a managed service (not as you directly), it needs its own IAM role with permissions for everything it needs to do — in this case, publishing to SNS. You will also attach an AWS managed policy to handle logging automatically.

**Trust Policy** is the part of an IAM role that controls *who* can assume the role. For Lambda functions, the trust policy allows the `lambda.amazonaws.com` service to assume the role so that it can execute your code.

**Permissions Policy** is the part of an IAM role that controls *what* the role can do. Your Lambda function needs `sns:Publish` to send formatted alerts. You will write this as a custom policy scoped to your specific SNS topic.

**AWS Managed Policy** is a pre-built permissions policy maintained by AWS that you can attach to any role without writing JSON. `AWSLambdaBasicExecutionRole` is the standard managed policy for Lambda functions — it grants the CloudWatch Logs permissions every Lambda function needs to record its execution output. Using it means you do not have to write or maintain those permissions yourself.

**Resource-Based Policy** is an access control document attached directly to a resource (like a Lambda function) that defines who can invoke it. This is separate from the IAM execution role. EventBridge must be listed in the Lambda function's resource-based policy before it can trigger the function.

**Environment Variables** are key-value pairs you set on a Lambda function at deploy time. Instead of hardcoding your SNS topic ARN inside the Python code, you store it as an environment variable — making the function reusable across accounts and regions without code changes.

**Deployment Package** is the ZIP file you upload to Lambda containing your function code. For simple Python functions with no external libraries, the ZIP contains just your `.py` file.

---

## ⚠️ Reminder: How to Read Commands in This Lab

Commands are inside gray code boxes. **📋 Copy and paste** them into your terminal.

**Placeholders** look like `<THIS>` — replace them (including the `<` and `>`) with your actual values.

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name from Lab 1A | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account ID | `123456789012` |
| `<TOPIC_ARN>` | Your SNS topic ARN from Lab 4C | `arn:aws:sns:us-east-1:123456789012:workshop-security-alerts` |
| `<DETECTOR_ID>` | Your GuardDuty detector ID from Lab 4C | `a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6` |
| `<ROLE_ARN>` | The Lambda IAM role ARN (from Step 5) | `arn:aws:iam::123456789012:role/workshop-lambda-guardduty-role` |
| `<LAMBDA_ARN>` | The Lambda function ARN (from Step 7) | `arn:aws:lambda:us-east-1:123456789012:function:workshop-guardduty-formatter` |

---

## Lab Steps

### Step 1: Set Your AWS Profile

**Windows (PowerShell):**

📋 Copy and paste, **replacing `<YOUR_PROFILE_NAME>`**:

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**macOS / Linux:**

```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

Verify it works and confirm your Account ID:

```
aws sts get-caller-identity
```

**✅ You should see** your account ID and role.

---

### Step 2: Create Your Project Folder

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
mkdir ~\Desktop\workshop-lab-4d
cd ~\Desktop\workshop-lab-4d
```

**macOS / Linux:**

📋 Copy and paste:

```bash
mkdir ~/Desktop/workshop-lab-4d
cd ~/Desktop/workshop-lab-4d
```

Verify you are in the right folder:

```
pwd
```

**✅ You should see** a path ending in `workshop-lab-4d`.

> **💡 Save ALL files you create in this lab to this folder.**

**Open the folder in VS Code.** 📋 Copy and paste:

```
code .
```

> This opens VS Code with `workshop-lab-4d` as its **file tree**, so the Python and JSON files you create land in the right place. (You set up the `code` command in Lab 1B — if you see `'code' is not recognized`, close and reopen your terminal, or revisit Lab 1B, Step 6.)

---

### Step 3: Write the Lambda Function

This is the Python code that transforms raw GuardDuty JSON into a structured, human-readable email.

**Step 3a:** In the VS Code file tree, click the **New File** icon and name the file `format_finding.py`.

**Step 3b:** 📋 Copy and paste this entire block into it:

```python
import json
import boto3
import os


def lambda_handler(event, context):
    sns = boto3.client('sns')

    # Pull the GuardDuty finding out of the EventBridge wrapper
    detail = event.get('detail', {})

    # Extract the key fields
    finding_type = detail.get('type', 'Unknown')
    severity     = detail.get('severity', 0)
    description  = detail.get('description', 'No description available.')
    region       = detail.get('region', event.get('region', 'Unknown'))
    account_id   = detail.get('accountId', event.get('account', 'Unknown'))
    finding_id   = detail.get('id', 'Unknown')
    updated_at   = detail.get('updatedAt', 'Unknown')

    # Assign a severity label and urgency line based on the numeric score
    if severity >= 7:
        severity_label = 'HIGH'
        urgency = '*** URGENT — Immediate action required ***'
    elif severity >= 4:
        severity_label = 'MEDIUM'
        urgency = '** MEDIUM — Investigate within 4 hours **'
    else:
        severity_label = 'LOW'
        urgency = '* LOW — Review during business hours *'

    # Pull resource details (structure varies by finding type)
    resource      = detail.get('resource', {})
    resource_type = resource.get('resourceType', 'Unknown')

    resource_lines = [f'Resource Type:  {resource_type}']

    if resource_type == 'Instance':
        inst = resource.get('instanceDetails', {})
        resource_lines.append(f'Instance ID:    {inst.get("instanceId", "Unknown")}')
        resource_lines.append(f'Instance Type:  {inst.get("instanceType", "Unknown")}')
        # Show the Name tag if present
        tags = inst.get('tags', [])
        name_tag = next((t['value'] for t in tags if t.get('key') == 'Name'), None)
        if name_tag:
            resource_lines.append(f'Instance Name:  {name_tag}')

    elif resource_type == 'AccessKey':
        ak = resource.get('accessKeyDetails', {})
        resource_lines.append(f'Access Key ID:  {ak.get("accessKeyId", "Unknown")}')
        resource_lines.append(f'User Name:      {ak.get("userName", "Unknown")}')
        resource_lines.append(f'User Type:      {ak.get("userType", "Unknown")}')

    elif resource_type == 'S3Bucket':
        buckets = resource.get('s3BucketDetails', [])
        if buckets:
            resource_lines.append(f'Bucket Name:    {buckets[0].get("name", "Unknown")}')
            resource_lines.append(f'Bucket ARN:     {buckets[0].get("arn", "Unknown")}')

    elif resource_type == 'EKSCluster':
        eks = resource.get('eksClusterDetails', {})
        resource_lines.append(f'Cluster Name:   {eks.get("name", "Unknown")}')

    # Choose a recommended action checklist matched to the severity level
    if severity >= 7:
        actions = [
            '1. Immediately isolate the affected resource (stop instance / disable access key).',
            '2. Revoke any credentials associated with this finding.',
            '3. Review CloudTrail logs for unauthorized API calls in the past 24 hours.',
            '4. Escalate to your security team now.',
            '5. Open an incident response ticket and begin containment.',
        ]
    elif severity >= 4:
        actions = [
            '1. Review CloudTrail and VPC Flow Logs for the affected resource.',
            '2. Confirm whether the activity is authorized or expected.',
            '3. If unauthorized, isolate the resource and revoke credentials.',
            '4. Update your incident response ticket with findings.',
        ]
    else:
        actions = [
            '1. Review the finding during normal business hours.',
            '2. Determine whether the activity is expected behavior.',
            '3. If unexpected, investigate and update your security baseline.',
        ]

    # Build a direct link to this finding in the GuardDuty console
    console_link = (
        f'https://console.aws.amazon.com/guardduty/home?region={region}'
        f'#/findings?macros=current&fId={finding_id}'
    )

    divider = '-' * 60

    # Assemble the full formatted message
    message = '\n'.join([
        'GUARDDUTY SECURITY ALERT',
        '=' * 60,
        urgency,
        '',
        'FINDING SUMMARY',
        divider,
        f'Type:           {finding_type}',
        f'Severity:       {severity_label} ({severity})',
        f'Account:        {account_id}',
        f'Region:         {region}',
        f'Detected:       {updated_at}',
        f'Finding ID:     {finding_id}',
        '',
        'DESCRIPTION',
        divider,
        description,
        '',
        'AFFECTED RESOURCE',
        divider,
        *resource_lines,
        '',
        'RECOMMENDED ACTIONS',
        divider,
        *actions,
        '',
        'VIEW IN CONSOLE',
        divider,
        console_link,
        '',
        '=' * 60,
        'This alert was generated automatically by your GuardDuty SOAR',
        'pipeline. Do not reply to this email.',
    ])

    # SNS subjects are capped at 100 characters
    subject = f'[GuardDuty {severity_label}] {finding_type}'[:100]

    sns.publish(
        TopicArn=os.environ['SNS_TOPIC_ARN'],
        Subject=subject,
        Message=message,
    )

    return {'statusCode': 200, 'body': 'Alert sent.'}
```

**Step 3c:** **Save** the file (**Ctrl+S** / **Cmd+S**). You should see `format_finding.py` in the file tree.

> **What does this code do?**
>
> | Section | What It Does |
> |---------|-------------|
> | Extracts fields | Pulls finding type, severity score, description, account, region, and resource info from the raw EventBridge event |
> | Assigns severity label | Maps the numeric score (1–10) to HIGH / MEDIUM / LOW with a plain-English urgency line |
> | Builds resource block | Adds resource-specific details (instance ID, access key, S3 bucket name, etc.) based on the type of resource involved |
> | Recommends actions | Provides a pre-written action checklist matched to the severity level |
> | Publishes to SNS | Sends the formatted message to your SNS topic (ARN stored in an environment variable, not hardcoded) |

---

### Step 4: Package the Lambda Function

Lambda requires your code to be uploaded as a ZIP file.

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
Compress-Archive -Path .\format_finding.py -DestinationPath .\format_finding.zip
```

**macOS / Linux:**

📋 Copy and paste:

```bash
zip format_finding.zip format_finding.py
```

Verify the ZIP was created:

**Windows (PowerShell):**

```powershell
Get-Item .\format_finding.zip
```

**macOS / Linux:**

```bash
ls -lh format_finding.zip
```

**✅ You should see** `format_finding.zip` listed in the output.

---

### Step 5: Create the IAM Role for Lambda

Lambda functions need an IAM role to authenticate with other AWS services. You will create the role in two parts: a **trust policy** (who can use this role) and a **permissions policy** (what the role can do).

**Step 5a: Write the trust policy**

In the VS Code file tree, create a **New File** named `lambda-trust-policy.json`. 📋 Copy and paste this block into it (no placeholders to replace):

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

> **What does this do?** It tells AWS: "the Lambda service is allowed to assume this role." Without this, Lambda cannot use the role to call other services on your behalf.

**Step 5b: Create the IAM role**

📋 Copy and paste:

```
aws iam create-role --role-name workshop-lambda-guardduty-role --assume-role-policy-document file://lambda-trust-policy.json
```

**✅ You should see** JSON output containing the role ARN:

```json
{
    "Role": {
        "RoleName": "workshop-lambda-guardduty-role",
        "Arn": "arn:aws:iam::123456789012:role/workshop-lambda-guardduty-role",
        ...
    }
}
```

> **📝 Write down your Role ARN** (the value of `"Arn"`): ______________________________

**Step 5c: Write the permissions policy**

In the VS Code file tree, create a **New File** named `lambda-permissions.json`. 📋 Copy and paste this block into it, **replacing `<TOPIC_ARN>`** with your SNS topic ARN from Lab 4C:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowSNSPublish",
            "Effect": "Allow",
            "Action": "sns:Publish",
            "Resource": "<TOPIC_ARN>"
        }
    ]
}
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

> **What does this do?** It grants Lambda permission to publish to your specific SNS topic and nothing else. Scoping the `Resource` to your exact topic ARN (instead of `*`) follows the principle of least privilege — the function can only publish to the one topic it needs.

**Step 5d: Create the permissions policy in IAM**

📋 Copy and paste:

```
aws iam create-policy --policy-name workshop-lambda-guardduty-policy --policy-document file://lambda-permissions.json
```

**✅ You should see** JSON output containing the policy ARN:

```json
{
    "Policy": {
        "PolicyName": "workshop-lambda-guardduty-policy",
        "Arn": "arn:aws:iam::123456789012:policy/workshop-lambda-guardduty-policy",
        ...
    }
}
```

**Step 5e: Attach the permissions policy to the role**

📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>`**:

```
aws iam attach-role-policy --role-name workshop-lambda-guardduty-role --policy-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:policy/workshop-lambda-guardduty-policy
```

**✅ No output means success.**

**Step 5f: Attach the AWS managed policy for Lambda logging**

📋 Copy and paste:

```
aws iam attach-role-policy --role-name workshop-lambda-guardduty-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

**✅ No output means success.**

> **What does this do?** `AWSLambdaBasicExecutionRole` is an AWS-maintained policy that grants the CloudWatch Logs permissions every Lambda function needs to record its execution output. Attaching it here means Lambda can write logs automatically — you do not need to write or maintain those permissions yourself. Notice the ARN uses `::aws:` instead of your account ID — this is how all AWS managed policies are identified.

---

### Step 6: Wait for the IAM Role to Propagate

IAM roles take a few seconds to become available after creation. If you deploy Lambda immediately, you may get an error saying the role does not exist yet.

📋 Wait **15 seconds**, then continue to Step 7.

> **Why does this happen?** IAM is a global service and changes replicate across AWS infrastructure before the Lambda service can use them. A short pause is enough for the role to become available.

---

### Step 7: Deploy the Lambda Function

📋 Copy and paste, **replacing `<ROLE_ARN>` and `<TOPIC_ARN>`**:

```
aws lambda create-function --function-name workshop-guardduty-formatter --runtime python3.12 --role <ROLE_ARN> --handler format_finding.lambda_handler --zip-file fileb://format_finding.zip --environment "Variables={SNS_TOPIC_ARN=<TOPIC_ARN>}" --timeout 15 --region us-east-1
```

> **🔄 Example:**
> ```
> aws lambda create-function --function-name workshop-guardduty-formatter --runtime python3.12 --role arn:aws:iam::123456789012:role/workshop-lambda-guardduty-role --handler format_finding.lambda_handler --zip-file fileb://format_finding.zip --environment "Variables={SNS_TOPIC_ARN=arn:aws:sns:us-east-1:123456789012:workshop-security-alerts}" --timeout 15 --region us-east-1
> ```

**✅ You should see** JSON output containing the function ARN:

```json
{
    "FunctionName": "workshop-guardduty-formatter",
    "FunctionArn": "arn:aws:lambda:us-east-1:123456789012:function:workshop-guardduty-formatter",
    "Runtime": "python3.12",
    ...
}
```

> **📝 Write down your Lambda Function ARN** (the value of `"FunctionArn"`): ______________________________

> **What do these flags mean?**
>
> | Flag | What It Does |
> |------|-------------|
> | `--runtime python3.12` | Tells Lambda which language runtime to use |
> | `--handler format_finding.lambda_handler` | Points Lambda to the function to run: `filename.function_name` |
> | `--zip-file fileb://format_finding.zip` | Uploads your code package (`fileb://` means read as binary) |
> | `--environment` | Sets the `SNS_TOPIC_ARN` environment variable your code reads with `os.environ` |
> | `--timeout 15` | Gives the function up to 15 seconds to run (the default of 3 seconds is too short for network calls to SNS) |

---

### Step 8: Grant EventBridge Permission to Invoke Lambda

EventBridge needs explicit permission to call your Lambda function. This is a **resource-based policy** added directly to the Lambda function — separate from the IAM role permissions you created in Step 5.

📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>`**:

```
aws lambda add-permission --function-name workshop-guardduty-formatter --statement-id AllowEventBridgeInvoke --action lambda:InvokeFunction --principal events.amazonaws.com --source-arn arn:aws:events:us-east-1:<YOUR_ACCOUNT_ID>:rule/workshop-guardduty-alert --region us-east-1
```

**✅ You should see** JSON output with a `Statement` field confirming the permission was added.

> **Why is this step needed?** Lambda has two separate access control layers:
> - **IAM role** (Steps 5–6) — controls what the Lambda function *can do* (publish to SNS, write logs)
> - **Resource-based policy** (this step) — controls who *can invoke* the Lambda function
>
> Without this step, EventBridge will be blocked from calling your function even though the rule and target are correctly configured.

---

### Step 9: Rewire EventBridge — Replace the SNS Target with Lambda

Right now the EventBridge rule sends findings directly to SNS (raw JSON). You will remove that target and replace it with your Lambda function.

**Step 9a: Remove the existing SNS target**

📋 Copy and paste:

```
aws events remove-targets --rule workshop-guardduty-alert --ids sns-target --region us-east-1
```

**✅ You should see** JSON output with `"FailedEntryCount": 0`.

**Step 9b: Add Lambda as the new target**

Passing the target inline is fiddly across shells (the quoting breaks easily), so put it in a small file — the reliable, cross-platform way.

In the VS Code file tree, create a **New File** named `lambda-target.json`. 📋 Copy and paste this into it, **replacing `<LAMBDA_ARN>`**:

```json
[
    {
        "Id": "lambda-target",
        "Arn": "<LAMBDA_ARN>"
    }
]
```

**Save** the file (**Ctrl+S** / **Cmd+S**), then add the target:

```
aws events put-targets --rule workshop-guardduty-alert --targets file://lambda-target.json --region us-east-1
```

**✅ You should see** JSON output with `"FailedEntryCount": 0`.

> **What changed?** EventBridge now sends matching GuardDuty findings to Lambda instead of directly to SNS. Lambda formats the finding into a readable message and publishes it to SNS. Your SNS topic and email subscription are unchanged — only the message content is different.

---

### Step 10: Test the Formatted Alert Pipeline

Generate a sample GuardDuty finding to trigger the full pipeline end-to-end.

📋 Copy and paste, **replacing `<DETECTOR_ID>`**:

```
aws guardduty create-sample-findings --detector-id <DETECTOR_ID> --finding-types "UnauthorizedAccess:IAMUser/MaliciousIPCaller.Custom" --region us-east-1
```

> **Why a different finding type than Lab 4C?** GuardDuty deduplicates findings of the same type within a rolling time window — repeated findings of the same type are aggregated as updates rather than new events, and EventBridge only fires on new findings. Using a different finding type guarantees a fresh event flows through the pipeline. For a full explanation of this behavior, see the note at the end of Lab 4C.

**✅ No output means success.**

> **⏱️ Wait 2–5 minutes**, then check your email inbox (and spam folder).

---

### Step 11: Verify the Formatted Email Alert

Open the email you receive. Instead of raw JSON, you should now see a structured alert like this:

```
GUARDDUTY SECURITY ALERT
============================================================
*** URGENT — Immediate action required ***

FINDING SUMMARY
------------------------------------------------------------
Type:           UnauthorizedAccess:IAMUser/MaliciousIPCaller.Custom
Severity:       HIGH (8.0)
Account:        123456789012
Region:         us-east-1
Detected:       2025-01-15T14:23:11Z
Finding ID:     abc123def456...

DESCRIPTION
------------------------------------------------------------
An API was invoked from an IP address on a custom threat list.

AFFECTED RESOURCE
------------------------------------------------------------
Resource Type:  AccessKey
Access Key ID:  AKIAIOSFODNN7EXAMPLE
User Name:      my-iam-user
User Type:      IAMUser

RECOMMENDED ACTIONS
------------------------------------------------------------
1. Immediately isolate the affected resource (stop instance / disable access key).
2. Revoke any credentials associated with this finding.
3. Review CloudTrail logs for unauthorized API calls in the past 24 hours.
4. Escalate to your security team now.
5. Open an incident response ticket and begin containment.

VIEW IN CONSOLE
------------------------------------------------------------
https://console.aws.amazon.com/guardduty/home?region=us-east-1#/findings?...

============================================================
This alert was generated automatically by your GuardDuty SOAR
pipeline. Do not reply to this email.
```

**✅ If you see a formatted alert like this:** Your pipeline is working correctly. Move on to the Console Checkpoint.

**💡 If you receive raw JSON instead:** The EventBridge rule may still be pointing to the old SNS target. Re-run both commands in Step 9 and verify `"FailedEntryCount": 0` on each.

**💡 If you receive no email at all:** Check the Lambda execution logs in Step 12 to diagnose the issue.

---

### Step 12: Check Lambda Logs (If Needed)

If no email arrived, Lambda's execution logs will show you exactly what happened — including any errors.

First, get the name of the most recent log stream:

📋 Copy and paste:

```
aws logs describe-log-streams --log-group-name /aws/lambda/workshop-guardduty-formatter --order-by LastEventTime --descending --max-items 1 --region us-east-1 --query "logStreams[0].logStreamName" --output text
```

This returns a log stream name. 📋 Copy that name, then run (📋 **replacing `<LOG_STREAM_NAME>`**):

```
aws logs get-log-events --log-group-name /aws/lambda/workshop-guardduty-formatter --log-stream-name "<LOG_STREAM_NAME>" --region us-east-1 --query "events[].message" --output text
```

**✅ A successful run shows** `"Alert sent."` and a `200` status code near the end of the output.

> **Common errors in the logs:**
>
> | Error | Cause | Fix |
> |-------|-------|-----|
> | `AuthorizationError` calling `sns:Publish` | Lambda role is missing SNS publish permission | Verify Step 5c — confirm `<TOPIC_ARN>` was replaced correctly, then re-run Steps 5d and 5e |
> | `ResourceNotFoundException` for SNS topic | The `SNS_TOPIC_ARN` environment variable is wrong | Run the update command in the Troubleshooting table below |
> | `Task timed out after 3.00 seconds` | Function timeout too short | Verify `--timeout 15` was included in Step 7; if not, see the fix in Troubleshooting |
> | No log group visible in CloudWatch | Lambda was never invoked | EventBridge cannot reach Lambda — re-run Steps 8 and 9b, then generate another sample finding |

---

### Step 13: Console Checkpoint

Verify all components in the AWS Console.

**Lambda Function:**
1. Go to [https://console.aws.amazon.com/](https://console.aws.amazon.com/) and search for **Lambda**
2. Click **Functions** — you should see **workshop-guardduty-formatter**
3. Click on the function
4. Under **Configuration → Environment variables** — verify `SNS_TOPIC_ARN` is set to your topic ARN
5. Under **Configuration → Permissions** — you should see the IAM role and, under **Resource-based policy**, a statement granting `events.amazonaws.com` permission to invoke the function

**EventBridge Rule:**
1. Search for **EventBridge** and click **Rules**
2. Click **workshop-guardduty-alert** — under **Targets**, you should now see your Lambda function (not the SNS topic)

**CloudWatch Logs:**
1. Search for **CloudWatch** and click **Log groups** in the left sidebar
2. You should see `/aws/lambda/workshop-guardduty-formatter`
3. Click into it to browse individual invocation logs and confirm the function ran without errors

**✅ Checkpoint:** You can trace the complete pipeline — GuardDuty (detection) → EventBridge (routing) → Lambda (formatting) → SNS (notification). This is a production-grade SOAR pattern used in real security operations centers.

---

## What You Just Did

You upgraded your automated alert pipeline from raw JSON dumps to actionable security notifications:

| Before (Lab 4C) | After (Lab 4D) |
|----------------|----------------|
| Raw JSON — hundreds of lines | Structured sections with clear labels |
| Numeric severity score only (`8.0`) | Plain-English label + score (`HIGH (8.0)`) |
| No guidance on what to do | Severity-matched action checklist |
| No direct link to the finding | One-click console link |
| Resource details buried in JSON | Instance ID, access key, or bucket called out clearly |

This pattern — inserting a Lambda function between the event source and the notification channel — is called a **message enrichment** or **alert normalization** step. In a real SOC, this same Lambda function would also:

- Look up the resource owner in a CMDB to auto-assign the ticket
- Attach relevant CloudTrail events from the past hour as supporting evidence
- Create a ticket in Jira or ServiceNow automatically
- Route HIGH severity findings to PagerDuty (wakes someone up at 3 AM) and MEDIUM to a Slack channel

---

## Cert Prep Callout

**Target Certification:** AWS Security Specialty (SCS-C02)

| Exam Topic | What This Lab Covers |
|-----------|---------------------|
| Lambda resource-based policies | Why EventBridge needs `lambda:InvokeFunction` permission separate from the IAM execution role |
| IAM execution roles | Trust policies vs. permissions policies — two documents, one role, two different questions answered |
| IAM least privilege | The permissions policy grants `sns:Publish` on *one specific topic ARN*, not `*` |
| EventBridge targets | How to chain services (Lambda as a target instead of SNS directly) |
| AWS managed policies | `AWSLambdaBasicExecutionRole` is the standard way to grant Lambda logging permissions without writing custom CloudWatch JSON |
| SOAR architecture | Lambda as the "Automation" layer between detection (GuardDuty) and response (SNS) |

**Sample question type:** "A Lambda function invoked by EventBridge returns an authorization error when calling SNS. The Lambda execution role has `sns:Publish` on the correct topic. What else must be verified?"  
**Answer:** A resource-based policy on the Lambda function must grant `events.amazonaws.com` permission to call `lambda:InvokeFunction`. The IAM role controls what Lambda *can do*; the resource-based policy controls who *can invoke* Lambda. Both must be correct.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `put-targets` returns `FailedEntryCount: 1` | EventBridge cannot access Lambda | Verify Step 8 — check that `<YOUR_ACCOUNT_ID>` was replaced correctly in the `--source-arn` |
| Lambda logs show `AccessDeniedException` on `sns:Publish` | Permissions policy not attached or wrong topic ARN | Re-run Steps 5c–5e; verify `<TOPIC_ARN>` was replaced correctly in `lambda-permissions.json` |
| Lambda logs show `ResourceNotFoundException` for SNS | `SNS_TOPIC_ARN` environment variable is incorrect | Run: `aws lambda update-function-configuration --function-name workshop-guardduty-formatter --environment "Variables={SNS_TOPIC_ARN=<CORRECT_TOPIC_ARN>}" --region us-east-1` |
| Email still contains raw JSON | Old SNS target was not removed | Re-run Step 9a (confirm `FailedEntryCount: 0`), then re-run Step 9b |
| No Lambda log group in CloudWatch | Function has never been invoked | Generate another sample finding (Step 10), wait 3 minutes, then check again |
| `InvalidParameterValueException: role cannot be assumed` when creating Lambda | IAM role not yet propagated | Wait 15 seconds and retry the `create-function` command in Step 7 |
| No email and logs show function ran successfully | SNS subscription is not confirmed | Run: `aws sns list-subscriptions-by-topic --topic-arn <TOPIC_ARN> --region us-east-1` and confirm the status is not `PendingConfirmation` |

---

## Cleanup

Clean up all resources created in this lab. Resources from Lab 4C (GuardDuty, SNS topic, EventBridge rule) are cleaned up separately in Lab 4C's cleanup section — run the steps below first, then run Lab 4C cleanup.

### Step 1: Remove the Lambda EventBridge Target

📋 Copy and paste:

```
aws events remove-targets --rule workshop-guardduty-alert --ids lambda-target --region us-east-1
```

**✅ No output or `"FailedEntryCount": 0` means success.**

### Step 2: Delete the Lambda Function

📋 Copy and paste:

```
aws lambda delete-function --function-name workshop-guardduty-formatter --region us-east-1
```

**✅ No output means success.**

### Step 3: Detach Both IAM Policies from the Role

📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>`**:

```
aws iam detach-role-policy --role-name workshop-lambda-guardduty-role --policy-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:policy/workshop-lambda-guardduty-policy
```

```
aws iam detach-role-policy --role-name workshop-lambda-guardduty-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

**✅ No output means success for each command.**

### Step 4: Delete the Custom IAM Policy

📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>`**:

```
aws iam delete-policy --policy-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:policy/workshop-lambda-guardduty-policy
```

**✅ No output means success.**

> **Why only delete one policy?** `AWSLambdaBasicExecutionRole` is an AWS managed policy — AWS owns it and it exists across all accounts. You can only delete policies you created yourself.

### Step 5: Delete the IAM Role

📋 Copy and paste:

```
aws iam delete-role --role-name workshop-lambda-guardduty-role
```

**✅ No output means success.**

### Step 6: Delete the CloudWatch Log Group

📋 Copy and paste:

```
aws logs delete-log-group --log-group-name /aws/lambda/workshop-guardduty-formatter --region us-east-1
```

**✅ No output means success.**

### Step 7: Run Lab 4C Cleanup

Return to **Lab 4C — Cleanup** and run all steps there to remove the EventBridge rule, SNS topic, and GuardDuty detector.

### Step 8: Delete Local Files

> **⚠️ Close VS Code first.** If VS Code still has the `workshop-lab-4d` folder open, the delete will fail — especially on Windows. Choose **File → Close Folder** or quit VS Code before running the commands below.

**Windows (PowerShell):**

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-4d
```

**macOS / Linux:**

```bash
cd ~/Desktop
rm -rf ~/Desktop/workshop-lab-4d
```

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** you received (copy and paste it)
4. Your **operating system** (Windows, Mac, or Linux)
