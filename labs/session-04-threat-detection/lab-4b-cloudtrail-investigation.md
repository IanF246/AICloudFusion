# Lab 4B: Investigate Activity with CloudTrail

**Session:** 4 — Threat Detection & AWS Security Services  
**Track:** Cloud Security & IR  
**Difficulty:** Intermediate  
**Estimated Time:** 25–30 minutes  
**Target Cert:** AWS Security Specialty

---

## Overview

In this lab, you will:

1. **Set up CloudTrail** — create a trail that records all API activity to an S3 bucket
2. **Generate suspicious activity** — simulate actions an attacker might take
3. **Investigate with CloudTrail** — query the audit log to find out what happened
4. **Prove immutability** — demonstrate that even deleted users leave a permanent audit trail

By the end of this lab, you will understand how CloudTrail provides an immutable record of everything that happens in your AWS account — and why attackers cannot cover their tracks in a properly configured cloud environment.

---

## Prerequisites

- ✅ Completed **Lab 1A** (AWS CLI installed and configured)
- ✅ AWS CLI authenticated — run `aws sts get-caller-identity` and confirm it returns your account info
- ✅ **VS Code** installed (from Lab 1B) — any text editor works, but these labs assume VS Code

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| CloudTrail | Audit logging for AWS actions | First active trail is Always Free |
| Amazon S3 | Storage for CloudTrail logs | $0.023 per GB/month |
| IAM | Identity and Access Management | Always Free |

**Estimated cost for this lab: $0.00** — the buckets hold only a few tiny files and are deleted in cleanup. Your **first** CloudTrail trail is free, so make sure the trail from Lab 3C has already been deleted (a second active trail would start incurring charges).

This command lists any existing trails:
```
aws cloudtrail describe-trails --query "trailList[].Name" --region us-east-1
```
If it returns `[]`, there are none active — you're good to go.

---

## Concepts

Before we start, here is what each concept means:

**AWS CloudTrail** is the audit log of everything that happens in your AWS account. Every API call — whether from the console, CLI, or SDK — is recorded: who made the call, what they did, when they did it, and from where. Think of it as a flight recorder (black box) for your cloud environment.

**Trail** is a CloudTrail configuration that delivers log files to an S3 bucket. Without a trail, you can only see the last 90 days of management events in the Event History. With a trail, logs are stored permanently in S3.

**Event History** is CloudTrail's built-in view of the last 90 days of management events. You can search and filter events without creating a trail. This is what we will use for investigation in this lab.

**Lookup Events** is the CloudTrail API that lets you search the event history. You can filter by event name (what happened), username (who did it), resource type, and more. This is the primary tool for forensic investigation.

**Immutable Audit Log** means the log cannot be altered or deleted by the person being audited. Even if an attacker deletes a user, the CloudTrail record of that user's creation and deletion remains. You cannot cover your tracks in AWS.

**Forensic Investigation** is the process of examining evidence after a security incident to determine what happened, who was responsible, and what was affected. CloudTrail is the primary data source for cloud forensics.

---

## ⚠️ Reminder: How to Read Commands in This Lab

Commands are inside gray code boxes. **📋 Copy and paste** them into your terminal.

**Placeholders** look like `<THIS>` — replace them (including the `<` and `>`) with your actual values.

Here are the placeholders you will use in this lab:

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name from Lab 1A | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account ID (from `aws sts get-caller-identity`) | `123456789012` |
| `<TRAIL_BUCKET_NAME>` | A globally unique S3 bucket name for CloudTrail logs | `jane-doe-lab4b-trail-logs` |
| `<EVIDENCE_BUCKET_NAME>` | A globally unique S3 bucket name for the "evidence" bucket | `jane-doe-lab4b-evidence` |
| `<ACCESS_KEY_ID>` | The access key ID of the suspicious user (from Step 6d) | `AKIA...` |
| `<SECRET_ACCESS_KEY>` | The secret access key of the suspicious user (from Step 6d) | `wJalr...` |

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

**Verify it works and get your Account ID.** 📋 Copy and paste:

```
aws sts get-caller-identity
```

**✅ You should see** your account ID and role.

> **📝 Write down your Account ID** (the 12-digit number in the `"Account"` field): ______________________________

---

### Step 2: Create Your Project Folder

**Step 2a: Create the folder**

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
mkdir ~\Desktop\workshop-lab-4b
cd ~\Desktop\workshop-lab-4b
```

**macOS / Linux:**

📋 Copy and paste:

```bash
mkdir ~/Desktop/workshop-lab-4b
cd ~/Desktop/workshop-lab-4b
```

> **What does this do?** Creates a new folder on your Desktop called `workshop-lab-4b` and moves your terminal into that folder.

**Step 2b: Verify you're in the right folder**

📋 Copy and paste:

```
pwd
```

**✅ You should see** a path ending in `workshop-lab-4b`.

> **💡 From now on, save ALL files you create in this lab to this folder.**

**Step 2c: Open the folder in VS Code**

📋 Copy and paste:

```
code .
```

> **What does this do?** This opens VS Code with `workshop-lab-4b` as its **file tree** on the left, so the policy file you create in Step 4 lands in the right place. (You set up the `code` command in Lab 1B — if you see `'code' is not recognized`, close and reopen your terminal, or revisit Lab 1B, Step 6.)

---

> **🔁 Heads-up: Steps 3–5 repeat the CloudTrail setup from Lab 3C.** Creating the log bucket, applying the bucket policy, and starting a multi-region trail are the same steps you did in 3C — included here so this lab stands on its own. If you're comfortable with them, move through them quickly; the real focus of this lab is the **investigation** in Steps 6–9.

### Step 3: Create an S3 Bucket for CloudTrail Logs

CloudTrail needs an S3 bucket to store its log files.

📋 Copy and paste this command, **replacing `<TRAIL_BUCKET_NAME>`**:

```
aws s3 mb s3://<TRAIL_BUCKET_NAME> --region us-east-1
```

> **🔄 Example:**
> ```
> aws s3 mb s3://jane-doe-lab4b-trail-logs --region us-east-1
> ```

**✅ You should see:**

```
make_bucket: <TRAIL_BUCKET_NAME>
```

---

### Step 4: Write and Apply the CloudTrail Bucket Policy

CloudTrail needs permission to write log files to your bucket. You must add a bucket policy that allows the CloudTrail service to put objects in the bucket.

**Step 4a:** In the VS Code file tree, click the **New File** icon and name the file `cloudtrail-bucket-policy.json`.

**Step 4b:** 📋 Copy and paste this entire block into it, **replacing `<TRAIL_BUCKET_NAME>`** (3 places) and **`<YOUR_ACCOUNT_ID>`** (1 place):

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
            "Resource": "arn:aws:s3:::<TRAIL_BUCKET_NAME>"
        },
        {
            "Sid": "AWSCloudTrailWrite",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudtrail.amazonaws.com"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::<TRAIL_BUCKET_NAME>/AWSLogs/<YOUR_ACCOUNT_ID>/*",
            "Condition": {
                "StringEquals": {
                    "s3:x-amz-acl": "bucket-owner-full-control"
                }
            }
        }
    ]
}
```

> **🔄 Example:** If your bucket is `jane-doe-lab4b-trail-logs` and your account ID is `123456789012`:
> - `"Resource": "arn:aws:s3:::jane-doe-lab4b-trail-logs"`
> - `"Resource": "arn:aws:s3:::jane-doe-lab4b-trail-logs/AWSLogs/123456789012/*"`

**Step 4c:** **Save** the file (**Ctrl+S** / **Cmd+S**). You should see `cloudtrail-bucket-policy.json` appear in the file tree.

> **⚠️ Common mistakes:** Make sure you replaced `<TRAIL_BUCKET_NAME>` in all 3 places and `<YOUR_ACCOUNT_ID>` in 1 place. The account ID must be exactly 12 digits with no dashes or spaces.

**Step 4d: Apply the bucket policy**

📋 Copy and paste, **replacing `<TRAIL_BUCKET_NAME>`**:

```
aws s3api put-bucket-policy --bucket <TRAIL_BUCKET_NAME> --policy file://cloudtrail-bucket-policy.json
```

**✅ No output means success.**

---

### Step 5: Create and Start the CloudTrail Trail

**Step 5a: Create the trail**

📋 Copy and paste, **replacing `<TRAIL_BUCKET_NAME>`**:

```
aws cloudtrail create-trail --name workshop-investigation-trail --s3-bucket-name <TRAIL_BUCKET_NAME> --is-multi-region-trail
```

> **Recap:** this creates a multi-region trail named `workshop-investigation-trail` that delivers logs to your bucket (same flags as 3C — see that lab for the per-flag breakdown).

**✅ You should see** JSON output with the trail details, including `"Name": "workshop-investigation-trail"`.

**Step 5b: Start logging**

📋 Copy and paste:

```
aws cloudtrail start-logging --name workshop-investigation-trail
```

**✅ No output means success.** CloudTrail is now recording every API call in your account.

---

### Step 6: Generate Suspicious Activity

Now you will simulate actions that a suspicious user might take. Later, you will use CloudTrail to investigate what happened — like a forensic analyst would after a security incident.

> **💡 Scenario:** Imagine you are a security analyst. You have been told that a "suspicious-user" was created in the account, did some things, and then was deleted. Your job is to figure out what happened using CloudTrail.

**Step 6a: Create an "evidence" S3 bucket**

📋 Copy and paste, **replacing `<EVIDENCE_BUCKET_NAME>`**:

```
aws s3 mb s3://<EVIDENCE_BUCKET_NAME> --region us-east-1
```

> **🔄 Example:**
> ```
> aws s3 mb s3://jane-doe-lab4b-evidence --region us-east-1
> ```

**✅ You should see:** `make_bucket: <EVIDENCE_BUCKET_NAME>`

**Step 6b: Upload a file to the evidence bucket**

**Windows (PowerShell):**

📋 Copy and paste, **replacing `<EVIDENCE_BUCKET_NAME>`**:

```powershell
"CONFIDENTIAL: Employee salary data Q4 2025" | Out-File -Encoding utf8 sensitive-data.txt
aws s3 cp sensitive-data.txt s3://<EVIDENCE_BUCKET_NAME>/sensitive-data.txt
```

**macOS / Linux:**

📋 Copy and paste, **replacing `<EVIDENCE_BUCKET_NAME>`**:

```bash
echo "CONFIDENTIAL: Employee salary data Q4 2025" > sensitive-data.txt
aws s3 cp sensitive-data.txt s3://<EVIDENCE_BUCKET_NAME>/sensitive-data.txt
```

**✅ You should see:** `upload: ./sensitive-data.txt to s3://<EVIDENCE_BUCKET_NAME>/sensitive-data.txt`

**Step 6c: Create a suspicious IAM user**

📋 Copy and paste:

```
aws iam create-user --user-name suspicious-user
```

**✅ You should see** JSON output with `"UserName": "suspicious-user"`.

**Step 6d: Create access keys for the suspicious user**

📋 Copy and paste:

```
aws iam create-access-key --user-name suspicious-user
```

**✅ You should see** JSON output with an AccessKeyId and SecretAccessKey.

> **📝 Write down BOTH values** — you need them to act as the attacker in the next step, and the AccessKeyId again for cleanup:
>
> - **AccessKeyId:** ______________________________
> - **SecretAccessKey:** ______________________________
>
> ⚠️ The SecretAccessKey is shown **only once**.

**Step 6e: Become the attacker — switch to the suspicious user's credentials**

You will now operate as the backdoor user, to see what an attacker holding these stolen keys would try.

> **💡 Contrast with Lab 3C:** in 3C you switched identity with `assume-role`, which issues **temporary** credentials that expire in about an hour (the secure, modern pattern). Here the attacker uses **long-lived IAM user access keys** — static credentials that keep working until someone deletes them. These are exactly the kind of credentials attackers steal, which is why roles are preferred over user access keys in the real world.

**Windows (PowerShell):**

📋 Copy and paste, **replacing the placeholders** with the values from Step 6d:

```powershell
$env:AWS_ACCESS_KEY_ID="<ACCESS_KEY_ID>"
$env:AWS_SECRET_ACCESS_KEY="<SECRET_ACCESS_KEY>"
$env:AWS_DEFAULT_REGION="us-east-1"
Remove-Item Env:\AWS_PROFILE
```

**macOS / Linux:**

📋 Copy and paste, **replacing the placeholders** with the values from Step 6d:

```bash
export AWS_ACCESS_KEY_ID="<ACCESS_KEY_ID>"
export AWS_SECRET_ACCESS_KEY="<SECRET_ACCESS_KEY>"
export AWS_DEFAULT_REGION="us-east-1"
unset AWS_PROFILE
```

**⏳ Wait about 10 seconds** for the new keys to become active, then confirm who you are. 📋 Copy and paste:

```
aws sts get-caller-identity
```

**✅ You should see** `suspicious-user` in the output — you are now operating as the attacker.

**Step 6f: Probe the account (these attempts are DENIED — but logged)**

The suspicious user has no permissions, so every command below fails with `AccessDenied`. That is exactly the point: **CloudTrail records the failed attempts anyway**, complete with the attacker's identity and IP address. 📋 Copy and paste each one, **replacing `<EVIDENCE_BUCKET_NAME>`** in the second:

```
aws s3api list-buckets 2>&1
```

```
aws s3api get-bucket-acl --bucket <EVIDENCE_BUCKET_NAME> 2>&1
```

```
aws iam list-users 2>&1
```

**✅ You should see** `An error occurred (AccessDenied)` for each — the attacker tried to enumerate your buckets, probe the confidential bucket's permissions, and list other users, and was blocked every time.

**Step 6g: Switch back to your admin credentials**

**Windows (PowerShell):**

📋 Copy and paste, **replacing `<YOUR_PROFILE_NAME>`**:

```powershell
Remove-Item Env:\AWS_ACCESS_KEY_ID
Remove-Item Env:\AWS_SECRET_ACCESS_KEY
Remove-Item Env:\AWS_DEFAULT_REGION
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**macOS / Linux:**

📋 Copy and paste, **replacing `<YOUR_PROFILE_NAME>`**:

```bash
unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
unset AWS_DEFAULT_REGION
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**Confirm you are back to admin.** 📋 Copy and paste:

```
aws sts get-caller-identity
```

**✅ You should see** your admin role again (with `AdministratorAccess` in the ARN).

**Step 6h: Delete the access keys (simulating covering tracks)**

📋 Copy and paste, **replacing `<ACCESS_KEY_ID>`** with the AccessKeyId from Step 6d:

```
aws iam delete-access-key --user-name suspicious-user --access-key-id <ACCESS_KEY_ID>
```

**✅ No output means success.**

**Step 6i: Delete the suspicious user (simulating covering tracks)**

📋 Copy and paste:

```
aws iam delete-user --user-name suspicious-user
```

**✅ No output means success.**

> **💡 What just happened?** First you set the scene: as **admin**, you created a bucket holding confidential data — the target an attacker would be after (Steps 6a–6b). Then you played the **attacker**, who:
> 1. Created a backdoor user with access keys
> 2. **Used those keys to probe the account** — enumerating buckets, targeting the confidential bucket, and listing users (every attempt denied)
> 3. Deleted the user and keys to cover their tracks
>
> Notice the two identities at work: the bucket and data were staged under **your admin identity**, while the probing was done under **suspicious-user**. That is exactly the distinction the investigation in Step 8 will reveal — what was done *to* the account (under admin) versus what the attacker did *from inside* it (under suspicious-user).
>
> The user is gone. The access keys are gone. But did they really cover their tracks? Let's find out...

---

### Step 7: Wait for CloudTrail to Record Events

> **⏱️ CloudTrail events take 5–15 minutes to appear in the Event History.**
>
> While you wait, read the **Concepts** section above if you haven't already, or review the **Cert Prep Callout** at the bottom of this lab.
>
> After 5–10 minutes, continue to Step 8.

---

### Step 8: Investigate with CloudTrail

Now put on your forensic analyst hat. The suspicious user has been deleted, but CloudTrail remembers everything.

**Step 8a: List recent events**

📋 Copy and paste:

```
aws cloudtrail lookup-events --max-results 10 --query "Events[].{Time:EventTime,Name:EventName,User:Username,Source:EventSource}" --output table
```

**✅ You should see** a table showing recent API calls in your account, including events like `CreateUser`, `CreateAccessKey`, `DeleteAccessKey`, `DeleteUser`, `CreateBucket`, `PutObject`.

> **💡 If the table is empty or shows very few events:** CloudTrail events take 5–15 minutes to appear. Wait a few more minutes and try again. This is normal behavior.

---

**Step 8b: Filter by specific event — find user creation**

📋 Copy and paste:

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateUser --query "Events[].{Time:EventTime,User:Username,Affected:Resources[0].ResourceName}" --output table
```

**✅ You should see** the `CreateUser` event — proving that someone created a user, even though that user no longer exists.

---

**Step 8c: Filter by username — find everything the suspicious user touched**

📋 Copy and paste:

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=Username,AttributeValue=suspicious-user --query "Events[].{Time:EventTime,Name:EventName,Source:EventSource}" --output table
```

**✅ You should see** the suspicious user's **own** actions — the denied `ListBuckets`, `GetBucketAcl`, and `ListUsers` attempts from Step 6f. This is the attacker's reconnaissance, attributed directly to `suspicious-user`, even though that user has since been deleted.

> **💡 Note:** This searches for events where `suspicious-user` was the **actor** (the one making the API calls) — in contrast to Step 8b, where `suspicious-user` was the **target** (created by your admin identity). Together they show the full picture: what was done *to* the account, and what the attacker did *from inside* it.

---

**Step 8d: Find the deletion events — prove tracks cannot be covered**

📋 Copy and paste:

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteUser --query "Events[].{Time:EventTime,User:Username,Affected:Resources[0].ResourceName, Action:EventName}" --output table
```

**✅ You should see** the `DeleteUser` event — CloudTrail recorded that someone deleted `suspicious-user`, who did it, and when.

📋 Copy and paste:

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteAccessKey --query "Events[].{Time:EventTime,Actor:Username,Affected:Resources[0].ResourceName,Action:EventName}" --output table
```

**✅ You should see** the `DeleteAccessKey` event — even though the key is gone, the record of its creation and deletion remains.

---

**Step 8e: Pull the full detail of one attacker action (who, and from where)**

The tables above show *what* the suspicious user did. To see *who* and *from where*, open the full record of one of their denied attempts.

📋 Copy and paste:

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=Username,AttributeValue=suspicious-user --max-results 1 --query "Events[0].CloudTrailEvent" --output text
```

This prints the full CloudTrail record for the attacker's most recent action as one block of JSON. Read through it and look for:

```
"errorCode": "AccessDenied",
"sourceIPAddress": "203.0.113.47",
"userAgent": "aws-cli/2...",
"userIdentity": { ... "userName": "suspicious-user" ... }
```

**✅ That is a forensic goldmine.** You now know exactly which identity attempted the action, what they tried, that it was blocked, the **IP address** they came from, and the tool they used (`userAgent`) — all preserved even though the user has since been deleted.

> **💡 If the command returns nothing:** the attacker's events may not have propagated yet (CloudTrail lag of 5–15 minutes). Wait a few minutes and try again.

---

### Step 9: The Key Insight — You Cannot Cover Your Tracks

> **🔑 Critical lesson:** The suspicious user was deleted. The access keys were deleted. But CloudTrail still has a complete record of:
> - **Who** created the user (your admin identity)
> - **When** the user was created (exact timestamp)
> - **What** access keys were generated
> - **When** the keys and user were deleted
> - **From where** the actions were taken (source IP address)
>
> **In a properly configured AWS environment, you cannot cover your tracks.** This is why CloudTrail is the foundation of cloud forensics and compliance.

---

### Step 10: Console Checkpoint

Let's see the event history in the AWS Console:

1. Go to [https://console.aws.amazon.com/](https://console.aws.amazon.com/)
2. Search for **CloudTrail** in the top search bar and click it
3. Click **Event history** in the left sidebar
4. You should see a list of recent events in your account
5. Use the filter dropdown set to Event Name and search for:
   - **Event name:** `CreateUser` — find when the suspicious user was created
   - **Event name:** `DeleteUser` — find when it was deleted
6. Click on any event to see the full JSON details, including the source IP address and request parameters

**✅ Checkpoint:** You can see the complete audit trail in the console. Every action is recorded with who, what, when, and from where. This is exactly how security teams investigate incidents.

---

## What You Just Did

You performed a forensic investigation using CloudTrail:

1. **Set up audit logging** — created a CloudTrail trail that records all API activity
2. **Simulated an attacker** — created a backdoor user, used its keys to probe the account, then deleted the user to cover their tracks
3. **Investigated with CloudTrail** — used lookup-events to find exactly what happened
4. **Proved immutability** — demonstrated that deleted resources still leave a permanent audit trail

In the real world, this is how security teams investigate breaches:
- An alert fires (from GuardDuty or another tool)
- The analyst queries CloudTrail to understand the full scope of the incident
- They determine: what was accessed, what was changed, and who was responsible
- The evidence is preserved for legal proceedings or compliance reporting

---

## Cert Prep Callout

**Target Certification:** AWS Security Specialty (SCS-C02)

The Security Specialty exam tests CloudTrail extensively. You need to understand:
- How to create and configure trails (multi-region, organization trails)
- How to protect CloudTrail logs (S3 bucket policies, Object Lock, separate security account)
- How to use `lookup-events` for investigation
- The difference between management events and data events
- How CloudTrail integrates with other services (EventBridge, Athena, Security Hub)

**Sample question type:** "After a security incident, an analyst needs to determine which IAM user deleted an S3 bucket last Tuesday. Which AWS service provides this information?"  
**Answer:** AWS CloudTrail — use `lookup-events` with the `EventName=DeleteBucket` filter

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `An error occurred (InsufficientS3BucketPolicyException)` when creating the trail | The bucket policy does not grant CloudTrail permission to write | Open `cloudtrail-bucket-policy.json` and verify: (1) bucket name is correct in all 3 places, (2) account ID is correct, (3) the policy was applied with `put-bucket-policy` |
| `An error occurred (TrailAlreadyExistsException)` | The trail already exists from a previous attempt | Delete it first: `aws cloudtrail delete-trail --name workshop-investigation-trail` then try again |
| `lookup-events` returns empty results | Events take 5–15 minutes to appear | Wait 10–15 minutes and try again. This is normal CloudTrail behavior. |
| `An error occurred (EntityAlreadyExists)` when creating the user | The user already exists from a previous attempt | Delete it first: `aws iam delete-user --user-name suspicious-user` then try again |
| `An error occurred (MalformedPolicyDocument)` | JSON syntax error in the bucket policy file | Check for missing commas, brackets, or quotes. Verify all placeholders were replaced. |
| `An error occurred (NoSuchEntity)` when deleting the user | The user does not exist (already deleted or never created) | This is fine — continue to the next step |

---

## Cleanup

>[!IMPORTANT]
>**⚠️** Always clean up resources after completing a lab. Follow these steps in order.

### Step 1: Stop CloudTrail Logging

📋 Copy and paste:

```
aws cloudtrail stop-logging --name workshop-investigation-trail
```

**✅ No output means success.**

### Step 2: Delete the CloudTrail Trail

📋 Copy and paste:

```
aws cloudtrail delete-trail --name workshop-investigation-trail
```

**✅ No output means success.**

### Step 3: Empty and Delete the Trail Logs Bucket

📋 Copy and paste, **replacing `<TRAIL_BUCKET_NAME>`**:

```
aws s3 rm s3://<TRAIL_BUCKET_NAME> --recursive
```

```
aws s3 rb s3://<TRAIL_BUCKET_NAME>
```

**✅ You should see** `remove_bucket: <TRAIL_BUCKET_NAME>`.

### Step 4: Empty and Delete the Evidence Bucket

📋 Copy and paste, **replacing `<EVIDENCE_BUCKET_NAME>`**:

```
aws s3 rm s3://<EVIDENCE_BUCKET_NAME> --recursive
```

```
aws s3 rb s3://<EVIDENCE_BUCKET_NAME>
```

**✅ You should see** `remove_bucket: <EVIDENCE_BUCKET_NAME>`.

### Step 5: Delete Local Files

> **⚠️ Close VS Code first.** If VS Code still has the `workshop-lab-4b` folder open, the delete will fail — especially on Windows. Choose **File → Close Folder** or quit VS Code before running the commands below.

Remove the project folder:

**Windows (PowerShell):**

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-4b
```

**macOS / Linux:**

```bash
cd ~/Desktop
rm -rf ~/Desktop/workshop-lab-4b
```

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** you received (copy and paste it)
4. Your **operating system** (Windows, Mac, or Linux)
