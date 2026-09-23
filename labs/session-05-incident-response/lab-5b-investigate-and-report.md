# Lab 5B: Investigate an Incident & Write a Report

**Session:** 5 — Incident Response on AWS  
**Track:** Cloud Security & IR  
**Difficulty:** Intermediate  
**Estimated Time:** 45–55 minutes  
**Target Cert:** AWS Security Specialty

---

## Overview

In this lab, you will:

1. **Configure a CloudTrail trail with data event logging and CloudWatch Logs** — so every action in your account, including file downloads, is recorded and queryable
2. **Simulate a security incident** — create evidence of unauthorized access to an S3 bucket
3. **Investigate with CloudWatch Log Analytics** — query the full attacker timeline, including data exfiltration events, using the compromised account's identity
4. **Contain and eradicate** — revoke the attacker's credentials and remove the user
5. **Write an incident report** — document the incident using a professional template

By the end of this lab, you will understand how to configure CloudTrail to stream events into CloudWatch Logs, how to write Log Analytics queries to reconstruct a forensic timeline, and how to produce incident documentation that meets industry standards.

---

## Prerequisites

- ✅ Completed **Lab 1A** (AWS CLI installed and configured)
- ✅ AWS CLI authenticated — run `aws sts get-caller-identity` and confirm it returns your account info
- ✅ **VS Code** installed (from Lab 1B) — any text editor works, but these labs assume VS Code

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| IAM | Identity and Access Management | Always Free |
| Amazon S3 | Cloud storage for files | ~$0.00 (tiny files, short duration) |
| CloudTrail | First trail — management events | Always Free |
| CloudTrail | S3 data events via trail | ~$0.00 ($0.10 per 100,000 events — this lab generates fewer than 20) |
| CloudWatch Logs | Log ingestion from CloudTrail trail | ~$0.00 ($0.50 per GB — this lab generates kilobytes) |
| CloudWatch Log Analytics | Queries | ~$0.00 ($0.005 per GB scanned — kilobytes of data) |

**Estimated cost for this lab: $0.00**

---

## Why This Setup Matters

**CloudTrail Event History** (always free, always on) records management events only — things like creating users and buckets. It does NOT record data events (file downloads, uploads). Even more importantly, `aws cloudtrail lookup-events` in the CLI queries Event History, not trail log files — so it also returns management events only, regardless of whether a trail is configured.

**The problem for incident response:** The most damaging evidence in a breach — which files were stolen — lives in data events. If no trail is configured, that evidence never exists.

**The solution this lab uses:** A CloudTrail trail configured to:
1. Capture S3 data events on the evidence bucket (so `GetObject` is recorded)
2. Stream all events to CloudWatch Logs (so you can query everything using Log Analytics)

This combination gives you a complete, queryable forensic record — including the file download — that you can filter by the compromised account's identity.

> **This is what a mature AWS security program looks like.** GuardDuty detects the threat. CloudTrail + CloudWatch Logs provides the evidence. Log Analytics is the investigation tool.

---

## Concepts

**NIST Incident Response Lifecycle** is the industry-standard framework for handling security incidents. It has six phases:
1. **Preparation** — Have tools, processes, and training ready before an incident occurs
2. **Detection & Analysis** — Identify that an incident is happening and understand its scope
3. **Containment** — Stop the incident from spreading (revoke credentials, isolate systems)
4. **Eradication** — Remove the threat completely (delete malicious users, keys, resources)
5. **Recovery** — Return to normal operations safely
6. **Lessons Learned** — Document what happened and improve defenses

**Management Events vs. Data Events:**
- **Management events** — Operations that modify AWS infrastructure: creating users, creating buckets, attaching policies. Captured by Event History for free. Returned by `aws cloudtrail lookup-events`.
- **Data events** — Operations on data inside resources: downloading a file (`GetObject`), uploading a file (`PutObject`). Require a trail with data event logging. NOT returned by `lookup-events` — must be queried from CloudWatch Logs or S3 trail log files.

**CloudTrail Trail** is a configuration you create that tells CloudTrail to deliver a persistent, structured log of all events (management + data) to an S3 bucket and/or a CloudWatch Log Group.

**CloudWatch Logs** is AWS's log aggregation service. When CloudTrail streams events into it, every API call in your account becomes a queryable log entry in near-real time.

**CloudWatch Log Analytics** is a query engine built into CloudWatch. It lets you write SQL-like queries against your log groups to filter, sort, and aggregate events. In incident response, you use it to filter all events by the compromised account's username or access key ID.

**Forensic Timeline** is a chronological reconstruction of events during an incident — what happened, when, in what order, and who did it. CloudWatch Log Analytics is the tool that builds it.

**Incident Report** is a formal document recording what happened, how it was detected, what actions were taken, and what should improve. It is a legal record, a learning tool, and evidence for compliance audits.

---

## ⚠️ Reminder: How to Read Commands in This Lab

Commands are inside gray code boxes. **📋 Copy and paste** them into your terminal.

**Placeholders** look like `<THIS>` — replace them (including the `<` and `>`) with your actual values.

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name from Lab 1A | `AdministratorAccess-123456789012` |
| `<TRAIL_BUCKET_NAME>` | Your Lab 3C CloudTrail log bucket (or a new one if you skipped 3C) | `jdoe-security-trail-logs` |
| `<EVIDENCE_BUCKET_NAME>` | A globally unique S3 bucket name for the "evidence" data | `jdoe-lab5b-evidence` |
| `<ACCOUNT_ID>` | Your 12-digit AWS account ID (Step 3a) | `123456789012` |
| `<KEY_ID>` | The attacker's Access Key ID (Step 5d) | `AKIAIOSFODNN7EXAMPLE` |
| `<SECRET_KEY>` | The attacker's Secret Access Key (Step 5d) | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` |

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

**Verify it works:**

```
aws sts get-caller-identity
```

**✅ You should see** your account ID and role. If you get an expired token error, run `aws sso login --profile <YOUR_PROFILE_NAME>` first.

---

### Step 2: Create Your Project Folder

**Windows (PowerShell):**

```powershell
mkdir ~\Desktop\workshop-lab-5b
cd ~\Desktop\workshop-lab-5b
```

**macOS / Linux:**

```bash
mkdir ~/Desktop/workshop-lab-5b
cd ~/Desktop/workshop-lab-5b
```

Verify:

```
pwd
```

**✅ You should see** a path ending in `workshop-lab-5b`.

> **💡 Save ALL files you create in this lab to this folder.**

**Open the folder in VS Code.** 📋 Copy and paste:

```
code .
```

> This opens VS Code with `workshop-lab-5b` as its **file tree**, so the several JSON files and the incident report you create land in the right place. (You set up the `code` command in Lab 1B — if you see `'code' is not recognized`, close and reopen your terminal, or revisit Lab 1B, Step 6.)

---

### Step 3: Prepare the Trail for Data-Event Forensics

This entire step is the **Preparation** phase of the NIST IR Lifecycle — logging must be in place *before* the attack, or the evidence is already gone.

You'll build on the security-track CloudTrail trail (`workshop-security-trail`) you created in **Lab 3C** and **upgrade** it to capture the one thing management events can't show you: the actual file download (an S3 **data event**). You'll add CloudWatch Logs streaming (for Log Analytics queries) and a data-event selector on the evidence bucket.

**Step 3a: Get your AWS Account ID**

📋 Copy and paste:

```
aws sts get-caller-identity --query Account --output text
```

**✅ You should see** a 12-digit number.

> **📝 Write this down:**
>
> - **Account ID:** ______________________________

**Step 3b: Create the evidence bucket**

The bucket that will hold "sensitive" data the attacker targets.

📋 Copy and paste, **replacing `<EVIDENCE_BUCKET_NAME>`**:

```
aws s3 mb s3://<EVIDENCE_BUCKET_NAME> --region us-east-1
```

**✅ You should see:** `make_bucket: <EVIDENCE_BUCKET_NAME>`

**Step 3c: Verify your security-track trail is running**

📋 Copy and paste:

```
aws cloudtrail get-trail-status --name workshop-security-trail --query "IsLogging"
```

**✅ You should see** `true` — the trail you created in Lab 3C is active. Continue to Step 3d to upgrade it.

> **🔗 Don't have the trail?** (You skipped Lab 3C, or already tore it down.) Expand and run this one-time basic setup, then continue:
> 1. Create the log bucket — **replace `<TRAIL_BUCKET_NAME>`**:
>    `aws s3 mb s3://<TRAIL_BUCKET_NAME> --region us-east-1`
> 2. In VS Code, create `trail-policy.json` using the same two-statement CloudTrail bucket policy from **Lab 3C, Step 4** — set the `Resource` ARNs to your `<TRAIL_BUCKET_NAME>` and your 12-digit account ID. Then apply it:
>    `aws s3api put-bucket-policy --bucket <TRAIL_BUCKET_NAME> --policy file://trail-policy.json`
> 3. Create the trail:
>    `aws cloudtrail create-trail --name workshop-security-trail --s3-bucket-name <TRAIL_BUCKET_NAME> --is-multi-region-trail`
> 4. Start it:
>    `aws cloudtrail start-logging --name workshop-security-trail`

**Step 3d: Create a CloudWatch Log Group for the trail**

This is where CloudTrail will stream every event so you can query them with Log Analytics.

📋 Copy and paste:

```
aws logs create-log-group --log-group-name security-track-cloudtrail-logs --region us-east-1
```

**✅ No output means success.**

Set a 30-day retention policy so logs are automatically cleaned up:

```
aws logs put-retention-policy --log-group-name security-track-cloudtrail-logs --retention-in-days 30 --region us-east-1
```

**✅ No output means success.**

**Step 3e: Create an IAM role so CloudTrail can write to CloudWatch Logs**

CloudTrail needs explicit permission to send events to your log group. You grant this with a dedicated IAM role.

In the VS Code file tree, create a **New File** named `cloudtrail-trust-policy.json`. 📋 Copy and paste this entire block into it:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudtrail.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

Create the role:

```
aws iam create-role --role-name security-track-cloudtrail-role --assume-role-policy-document file://cloudtrail-trust-policy.json
```

**✅ You should see** JSON output with `"RoleName": "security-track-cloudtrail-role"`.

Now create another **New File** named `cloudtrail-logs-policy.json`. 📋 Copy and paste this entire block into it, **replacing `<ACCOUNT_ID>`**:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:us-east-1:<ACCOUNT_ID>:log-group:security-track-cloudtrail-logs:*"
        }
    ]
}
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

Attach the permissions to the role:

```
aws iam put-role-policy --role-name security-track-cloudtrail-role --policy-name CloudWatchLogsWrite --policy-document file://cloudtrail-logs-policy.json
```

**✅ No output means success.**

**Step 3f: Upgrade the trail to stream to CloudWatch Logs**

Attach the log group and role to your existing trail with `update-trail`. This adds CloudWatch Logs streaming to the trail you already have — no new trail is created.

📋 Copy and paste, **replacing `<ACCOUNT_ID>`**:

```
aws cloudtrail update-trail --name workshop-security-trail --cloud-watch-logs-log-group-arn arn:aws:logs:us-east-1:<ACCOUNT_ID>:log-group:security-track-cloudtrail-logs:* --cloud-watch-logs-role-arn arn:aws:iam::<ACCOUNT_ID>:role/security-track-cloudtrail-role --region us-east-1
```

**✅ You should see** JSON output for `workshop-security-trail` now listing the CloudWatch log group and role ARNs.

**Step 3g: Enable data event logging for the evidence bucket**

This is the key step — it tells the trail to record every object-level operation (downloads, uploads) on your evidence bucket, not just bucket-level management events.

In the VS Code file tree, create a **New File** named `event-selectors.json`. 📋 Copy and paste this entire block into it, **replacing `<EVIDENCE_BUCKET_NAME>`**:

```json
[
    {
        "ReadWriteType": "All",
        "IncludeManagementEvents": true,
        "DataResources": [
            {
                "Type": "AWS::S3::Object",
                "Values": ["arn:aws:s3:::<EVIDENCE_BUCKET_NAME>/"]
            }
        ]
    }
]
```

**Save** the file (**Ctrl+S** / **Cmd+S**), then apply:

```
aws cloudtrail put-event-selectors --trail-name workshop-security-trail --event-selectors file://event-selectors.json
```

**✅ You should see** JSON confirming `"ReadWriteType": "All"` and your bucket ARN under `DataResources`.

**Step 3h: Confirm the trail is logging**

Your trail has been logging since Lab 3C, but confirm it's active:

```
aws cloudtrail get-trail-status --name workshop-security-trail --query "{IsLogging:IsLogging,LatestDeliveryError:LatestDeliveryError}"
```

**✅ You should see:** `{ "IsLogging": true }` with no delivery error. If `IsLogging` is `false`, run `aws cloudtrail start-logging --name workshop-security-trail`.

> **What did you just do?** You took the security-track trail from Lab 3C and upgraded it into a full forensic recorder:
> - It still delivers structured logs to your S3 bucket (long-term archival)
> - It now **also streams every event to CloudWatch Logs** in near-real time (for live investigation with Log Analytics)
> - It now captures **S3 data events** on your evidence bucket (so file downloads are recorded)
>
> Every API call against your evidence bucket — including who downloaded what and when — will now appear in CloudWatch Logs within minutes.

---

### Step 4: Set Up the Evidence

**Step 4a: Upload a "sensitive" file**

**Windows (PowerShell):**

📋 Copy and paste, **replacing `<EVIDENCE_BUCKET_NAME>`**:

```powershell
"CONFIDENTIAL: Employee salary data - Q4 2024 report. SSN: XXX-XX-XXXX" | Out-File -Encoding utf8 sensitive-data.txt
aws s3 cp sensitive-data.txt s3://<EVIDENCE_BUCKET_NAME>/sensitive-data.txt
```

**macOS / Linux:**

📋 Copy and paste, **replacing `<EVIDENCE_BUCKET_NAME>`**:

```bash
echo "CONFIDENTIAL: Employee salary data - Q4 2024 report. SSN: XXX-XX-XXXX" > sensitive-data.txt
aws s3 cp sensitive-data.txt s3://<EVIDENCE_BUCKET_NAME>/sensitive-data.txt
```

**✅ You should see:** `upload: ./sensitive-data.txt to s3://<EVIDENCE_BUCKET_NAME>/sensitive-data.txt`

**Step 4b: Create the attacker user**

```
aws iam create-user --user-name attacker-simulation
```

**✅ You should see** JSON with `"UserName": "attacker-simulation"`.

**Step 4c: Give the attacker S3 read access**

In the VS Code file tree, create a **New File** named `attacker-policy.json`. 📋 Copy and paste this into it:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowS3Read",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetObject",
                "s3:ListAllMyBuckets"
            ],
            "Resource": "*"
        }
    ]
}
```

**Save** the file (**Ctrl+S** / **Cmd+S**), then apply:

```
aws iam put-user-policy --user-name attacker-simulation --policy-name S3ReadAccess --policy-document file://attacker-policy.json
```

**✅ No output means success.**

**Step 4d: Create access keys for the attacker**

```
aws iam create-access-key --user-name attacker-simulation
```

**✅ You should see** JSON with `AccessKeyId` and `SecretAccessKey`.

> **📝 Write down BOTH values immediately:**
>
> - **AccessKeyId:** ______________________________
> - **SecretAccessKey:** ______________________________

---

### Step 5: Simulate the Attack

Switch to the attacker's credentials and perform the unauthorized actions. Because the trail is live, every API call is being recorded.

**Step 5a: Set the attacker's credentials**

**Windows (PowerShell):**

📋 Copy and paste, **replacing `<KEY_ID>` and `<SECRET_KEY>`**:

```powershell
$env:AWS_ACCESS_KEY_ID="<KEY_ID>"
$env:AWS_SECRET_ACCESS_KEY="<SECRET_KEY>"
$env:AWS_DEFAULT_REGION="us-east-1"
Remove-Item Env:\AWS_PROFILE
```

**macOS / Linux:**

```bash
export AWS_ACCESS_KEY_ID="<KEY_ID>"
export AWS_SECRET_ACCESS_KEY="<SECRET_KEY>"
export AWS_DEFAULT_REGION="us-east-1"
unset AWS_PROFILE
```

**Step 5b: Verify you are the attacker**

```
aws sts get-caller-identity
```

**✅ You should see** `attacker-simulation` — NOT your admin role.

**Step 5c: Reconnaissance — list all S3 buckets**

```
aws s3 ls
```

**✅ You should see** a list of buckets including `<EVIDENCE_BUCKET_NAME>`.

**Step 5d: List the contents of the sensitive bucket**

📋 Copy and paste, **replacing `<EVIDENCE_BUCKET_NAME>`**:

```
aws s3 ls s3://<EVIDENCE_BUCKET_NAME>/
```

**✅ You should see** `sensitive-data.txt` listed.

**Step 5e: Download the sensitive file (data exfiltration)**

📋 Copy and paste, **replacing `<EVIDENCE_BUCKET_NAME>`**:

```
aws s3 cp s3://<EVIDENCE_BUCKET_NAME>/sensitive-data.txt stolen-data.txt
```

**✅ You should see:** `download: s3://<EVIDENCE_BUCKET_NAME>/sensitive-data.txt to ./stolen-data.txt`

> **The attacker now has your confidential data.** This `GetObject` call was recorded by the trail as a data event and is streaming to CloudWatch Logs right now. You will see it in your investigation.

**Step 5f: Switch back to admin credentials**

**Windows (PowerShell):**

📋 Copy and paste, **replacing `<YOUR_PROFILE_NAME>`**:

```powershell
Remove-Item Env:\AWS_ACCESS_KEY_ID
Remove-Item Env:\AWS_SECRET_ACCESS_KEY
Remove-Item Env:\AWS_DEFAULT_REGION
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**macOS / Linux:**

```bash
unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
unset AWS_DEFAULT_REGION
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

Verify:

```
aws sts get-caller-identity
```

**✅ You should see** your admin role.

---

### Step 6: Wait for Events to Arrive in CloudWatch Logs

> **⏱️ CloudTrail delivers events to CloudWatch Logs within 5–15 minutes.** This is normal.
>
> **While you wait:** Re-read the Concepts section and think about which events from Step 5 fall under management events versus data events. Consider — if you only had Event History and the CLI `lookup-events` command, which part of the attack would be invisible?
>
> **After 10–15 minutes**, continue to Step 7.

---

### Step 7: Investigate — CLI lookup-events (Management Events Only)

Start the investigation with the AWS CLI. This is the quickest first step in any incident — it returns immediately and shows the IAM activity around the breach.

📋 Copy and paste:

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=Username,AttributeValue=attacker-simulation --query "Events[].{Time:EventTime,Name:EventName,Source:EventSource}" --output table
```

**✅ You should see** a table of management events:

```
----------------------------------------------------------------------
|                           LookupEvents                             |
+-----------------------------+--------------------+-----------------+
|            Time             |        Name        |     Source      |
+-----------------------------+--------------------+-----------------+
|  2024-XX-XXTXX:XX:XXZ      |  GetCallerIdentity |  sts...         |
|  2024-XX-XXTXX:XX:XXZ      |  ListBuckets       |  s3...          |
+-----------------------------+--------------------+-----------------+
```

> **⚠️ Important limitation:** `aws cloudtrail lookup-events` queries CloudTrail Event History, not the trail's log files. It returns **management events only** — regardless of whether a trail with data events is configured. The `GetObject` event (the file download) will NOT appear here. This is why CloudWatch Log Analytics is required for a complete investigation.
>
> Use this CLI output as a quick first pass to confirm the attacker's identity and timestamp. Use Log Analytics in Step 8 for the full forensic picture.

---

### Step 8: Investigate — CloudWatch Log Analytics (Full Forensic Timeline)

This is the primary investigation step. CloudWatch Log Analytics queries the trail's log stream directly, which includes both management events and data events. You will filter by the compromised account's identity to reconstruct everything the attacker did.

**Step 8a: Open CloudWatch Logs in the console**

1. Go to [https://console.aws.amazon.com/](https://console.aws.amazon.com/)
2. Search for **CloudWatch** in the top search bar and click it
3. In the left sidebar, under **Logs**, click **Log Analytics**

**Step 8b: Select the log group**

1. Where it says **Log group** dropdown, select **All** to open search log group
2. Type `security-track-cloudtrail-logs` and select it, or scroll and select it
3. Set the time range to **Last 1 hour** (or **Custom** if you started the lab more than an hour ago)

**Step 8c: Run the investigation query**

Clear the default query and 📋 copy and paste this query into the editor:

```
fields @timestamp, eventName, userIdentity.userName, userIdentity.accessKeyId,
       requestParameters.bucketName, requestParameters.key,
       sourceIPAddress, errorCode, errorMessage
| filter userIdentity.userName = "attacker-simulation"
| sort @timestamp asc
```

Click **Run query**.

**✅ You should see** every action the attacker took, in chronological order, including the file download:

| @timestamp | eventName | userIdentity.userName | requestParameters.bucketName | requestParameters.key |
|-----------|-----------|----------------------|-----------------------------|-----------------------|
| 2024-XX-XX... | GetCallerIdentity | attacker-simulation | — | — |
| 2024-XX-XX... | ListBuckets | attacker-simulation | — | — |
| 2024-XX-XX... | ListObjects | attacker-simulation | \<EVIDENCE_BUCKET_NAME> | — |
| 2024-XX-XX... | GetObject | attacker-simulation | \<EVIDENCE_BUCKET_NAME> | sensitive-data.txt |

> **This is your forensic timeline.** Every row is a tamper-evident, timestamped record of what the attacker did. The `GetObject` row on `sensitive-data.txt` is your evidence of data exfiltration — the specific file that was stolen.

**Step 8d: Query by Access Key ID instead of username**

In a real incident, you may only have the access key ID (from an alert), not the username. Try querying by key ID to practice this:

Clear the query and 📋 copy and paste, **replacing `<KEY_ID>`** with the attacker's key from Step 4d:

```
fields @timestamp, eventName, userIdentity.userName, userIdentity.accessKeyId,
       requestParameters.bucketName, requestParameters.key,
       sourceIPAddress
| filter userIdentity.accessKeyId = "<KEY_ID>"
| sort @timestamp asc
```

Click **Run query**.

**✅ You should see** the same timeline as before — same events, just queried by key ID instead of username. Both query methods identify the same attacker.

> **Why does this matter?** GuardDuty and other detection tools typically alert on access key IDs, not usernames. Knowing how to pivot from a key ID to a full activity timeline is a core IR skill.

**Step 8e: Isolate the exfiltration event**

Narrow the query to focus specifically on the data exfiltration:

```
fields @timestamp, userIdentity.userName, userIdentity.accessKeyId,
       requestParameters.bucketName, requestParameters.key,
       sourceIPAddress, responseElements
| filter eventName = "GetObject" and userIdentity.userName = "attacker-simulation"
| sort @timestamp asc
```

Click **Run query**.

**✅ You should see** a single row confirming: who downloaded the file, which bucket, which file, at what time, and from which IP address. This one record is the core evidence of data exfiltration for your incident report.

> **💡 In a real incident**, you would copy the full JSON of this event and attach it as an exhibit to the incident report. Click on any row in Log Analytics to expand the full raw event.

---

### Step 9: Contain — Deactivate the Attacker's Key

📋 Copy and paste, **replacing `<KEY_ID>`**:

```
aws iam update-access-key --user-name attacker-simulation --access-key-id <KEY_ID> --status Inactive
```

**✅ No output means success.** The attacker's credentials are revoked — they cannot authenticate again.

---

### Step 10: Eradicate — Remove the Attacker

**Step 10a: Delete the access key**

📋 Copy and paste, **replacing `<KEY_ID>`**:

```
aws iam delete-access-key --user-name attacker-simulation --access-key-id <KEY_ID>
```

**Step 10b: Delete the user policy**

```
aws iam delete-user-policy --user-name attacker-simulation --policy-name S3ReadAccess
```

**Step 10c: Delete the attacker user**

```
aws iam delete-user --user-name attacker-simulation
```

**✅ No output for each command means success.**

---

### Step 11: Document — Write the Incident Report

**Step 11a:** In the VS Code file tree, create a **New File** named `incident-report.md`.

**Step 11b:** 📋 Copy and paste this entire template into it:

```markdown
# Incident Report

## Incident Summary

| Field | Value |
|-------|-------|
| **Incident ID** | IR-2024-001 |
| **Date Detected** | [TODAY'S DATE] |
| **Severity** | High |
| **Status** | Resolved |
| **Reported By** | [YOUR NAME] |
| **Compromised Identity** | attacker-simulation / [ACCESS KEY ID] |

## Detection

**How was the incident discovered?**

[DESCRIBE HOW THE INCIDENT WAS DETECTED — e.g., "Automated GuardDuty alert for unusual API activity from IAM user attacker-simulation"]

## Timeline of Events

| Time (UTC) | Event | Details | Evidence Source |
|------------|-------|---------|-----------------|
| [TIME] | Attacker credentials created | Access key created for attacker-simulation | CloudTrail — management event (CLI lookup-events) |
| [TIME] | Attacker authenticated | GetCallerIdentity called | CloudTrail — management event (CLI lookup-events + Log Analytics) |
| [TIME] | Reconnaissance | ListBuckets — attacker surveyed all S3 buckets | CloudTrail — management event (CLI lookup-events + Log Analytics) |
| [TIME] | Data discovery | ListObjects on [BUCKET NAME] — attacker enumerated bucket contents | CloudTrail — data event (CloudWatch Log Analytics only) |
| [TIME] | Data exfiltration | GetObject on sensitive-data.txt — file downloaded from [BUCKET NAME] | CloudTrail — data event (CloudWatch Log Analytics only) |
| [TIME] | Containment | Access key [KEY ID] deactivated | Responder action |
| [TIME] | Eradication | attacker-simulation user and all credentials deleted | Responder action |

## Affected Resources

| Resource | Type | Impact |
|----------|------|--------|
| [YOUR BUCKET NAME] | S3 Bucket | Unauthorized read access |
| sensitive-data.txt | S3 Object | Confirmed download via GetObject (CloudWatch Log Analytics) |
| attacker-simulation | IAM User | Unauthorized user with S3 read access; now deleted |

## Key Forensic Evidence

**GetObject event (data exfiltration — from CloudWatch Log Analytics):**

| Field | Value |
|-------|-------|
| Event Time | [TIMESTAMP FROM LOG ANALYTICS] |
| Event Name | GetObject |
| Username | attacker-simulation |
| Access Key ID | [KEY ID] |
| Source IP | [IP FROM LOG ANALYTICS] |
| Bucket | [EVIDENCE BUCKET NAME] |
| Object Key | sensitive-data.txt |

## Containment Actions

1. Deactivated the compromised access key immediately upon detection
2. Verified the key could no longer authenticate

## Eradication Actions

1. Deleted the compromised access key permanently
2. Removed the attacker's IAM inline policy
3. Deleted the attacker IAM user

## Root Cause

[DESCRIBE THE ROOT CAUSE — e.g., "Access keys were committed to a public repository due to lack of .gitignore rules and pre-commit secret scanning"]

## Lessons Learned & Recommendations

1. [e.g., "Implement automated secret scanning in CI/CD pipeline (e.g., git-secrets, truffleHog)"]
2. [e.g., "Enable GuardDuty for continuous threat detection and automated alerting"]
3. [e.g., "Use IAM roles and SSO instead of long-lived access keys wherever possible"]
4. [e.g., "Apply least-privilege S3 bucket policies — restrict access to known principals and source VPCs"]
5. [e.g., "Ensure CloudTrail trails with S3 data event logging and CloudWatch Logs integration are configured in all accounts before an incident occurs — evidence that does not exist cannot be recovered"]
```

**Step 11c:** **Save** the file (**Ctrl+S** / **Cmd+S**). You should see `incident-report.md` in the file tree.

**Step 11d:** Fill in the template using the timestamps and data from Steps 7 and 8. Pay attention to the **Evidence Source** column — note which events were only visible in CloudWatch Log Analytics (data events) versus which also appeared in the CLI `lookup-events` output (management events). This distinction goes directly into the Key Forensic Evidence section.

---

### Step 12: Console Checkpoint

**Part A — CloudTrail Event History (management events only):**

1. Go to [https://console.aws.amazon.com/](https://console.aws.amazon.com/)
2. Search for **CloudTrail** and click it
3. Click **Event history** in the left sidebar
4. Filter by **User name** → `attacker-simulation`
5. You will see `GetCallerIdentity` and `ListBuckets` — management events only

**Part B — CloudTrail Trail:**

1. In the left sidebar, click **Trails**
2. Click **workshop-security-trail**
3. Confirm the trail shows: S3 bucket, CloudWatch log group, and data events configured for your evidence bucket

**Part C — CloudWatch Log Analytics:**

1. Navigate to CloudWatch → Logs → Log Analytics
2. Re-run the query from Step 8c
3. Confirm `GetObject` on `sensitive-data.txt` appears in the results

**✅ Checkpoint:** You can see that Event History shows a partial picture (management events only), while Log Analytics shows the complete forensic record including data exfiltration. This is why CloudTrail + CloudWatch Logs is the production-grade investigation setup.

---

## What You Just Did

You executed a complete incident response following the NIST framework:

1. **Preparation** — Built a CloudTrail trail with data events and CloudWatch Logs before the attack
2. **Detection & Analysis** — Identified the breach and reconstructed the full timeline using Log Analytics
3. **Containment** — Deactivated the attacker's credentials
4. **Eradication** — Removed the attacker completely
5. **Recovery** — Verified the threat is eliminated
6. **Lessons Learned** — Documented the incident with a professional report

You practiced four critical IR skills:
- **Proactive logging** — Configuring a trail before an incident so all evidence is preserved
- **Management vs. data events** — Understanding the exact limitation of `lookup-events` and why CloudWatch is needed
- **Forensic querying** — Writing Log Analytics queries filtered by compromised identity (username and access key ID)
- **Incident documentation** — Writing a professional report that cites its evidence sources

---

## Cert Prep Callout

**Target Certification:** AWS Security Specialty (SCS-C02)

| Topic | What You Practiced |
|-------|--------------------|
| CloudTrail trail creation & update | Lab 3C (create) + Step 3f (`update-trail`) |
| S3 data event logging | Step 3g |
| CloudTrail → CloudWatch Logs integration | Steps 3d–3f |
| Management events vs. data events | Steps 7 vs. 8 |
| CloudWatch Log Analytics queries | Step 8c–8e |
| Querying by access key ID | Step 8d |
| NIST IR Lifecycle | All steps |

**Sample exam question:** "A security engineer reviews CloudTrail Event History after an S3 breach but finds no `GetObject` events. What is the most likely cause, and what should have been configured in advance?"  
**Answer:** `lookup-events` and Event History return management events only. A CloudTrail trail with S3 data event logging must be configured before the incident to capture object-level access.

**Sample exam question:** "A SOC analyst receives a GuardDuty alert containing a compromised IAM access key ID. Which service and query approach gives the analyst a complete timeline of all API calls made with that key, including S3 object downloads?"  
**Answer:** CloudWatch Log Analytics, querying the CloudTrail log group filtered by `userIdentity.accessKeyId`.

**Sample exam question:** "A company needs to ensure that any future S3 data access is queryable within minutes of occurring. Which combination of services satisfies this requirement?"  
**Answer:** CloudTrail trail with S3 data event logging enabled, integrated with CloudWatch Logs.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `InsufficientS3BucketPolicyException` (only if you used the Step 3c fallback to create the trail) | CloudTrail cannot write to the log bucket | Re-check `trail-policy.json` — ensure the bucket name and your account ID were filled in correctly, then re-apply the policy |
| `update-trail` returns an error about the CloudWatch Logs role | IAM role permissions are not yet propagated | Wait 10 seconds and retry — IAM changes take a moment to propagate |
| `get-trail-status` shows `LatestDeliveryError` | CloudTrail cannot deliver to S3 | Verify the bucket policy was applied and the bucket name in the trail matches exactly |
| Log Analytics query returns no results | Events have not arrived yet, or wrong time range | Set the time range to **Last 1 hour** and wait 10 more minutes. CloudTrail delivers to CloudWatch within 5–15 minutes |
| `GetObject` not appearing in Log Analytics | The trail was not active when the attack ran, OR data events were not configured | Verify the trail is logging (`get-trail-status`), re-run the attack steps (5c–5e), wait 15 minutes, and query again |
| `lookup-events` does not show `GetObject` | This is expected — `lookup-events` only returns management events | Use CloudWatch Log Analytics (Step 8) for data events |
| `get-caller-identity` still shows attacker after switching back | Env vars were not cleared | Run the switch-back commands in Step 5f again. Close and reopen terminal if needed |
| Cannot delete attacker user | User still has policies or access keys | Delete in order: access key → user policy → user (Steps 10a, 10b, 10c) |

---

## Cleanup

**⚠️ Run these steps in order.**

> **🔗 Keep the shared trail running.** The `workshop-security-trail` trail (now upgraded with CloudWatch Logs + data events), its log group `security-track-cloudtrail-logs`, the `security-track-cloudtrail-role` role, and the `<TRAIL_BUCKET_NAME>` log bucket are the **shared security-track backbone** — **Lab 5C reuses the trail**. Leave them running and remove only this lab's own resources below. You'll tear the whole backbone down with the **Final Track Teardown** at the end of Lab 5C. (If you're stopping the security track here instead, go run that Final Track Teardown to remove them.)

### Step 1: Delete the Evidence Bucket

📋 Copy and paste, **replacing `<EVIDENCE_BUCKET_NAME>`**:

```
aws s3 rm s3://<EVIDENCE_BUCKET_NAME> --recursive
```

```
aws s3 rb s3://<EVIDENCE_BUCKET_NAME>
```

**✅ You should see:** `remove_bucket: <EVIDENCE_BUCKET_NAME>`

### Step 2: Verify Attacker User Is Deleted

```
aws iam get-user --user-name attacker-simulation
```

**✅ Expected:** `NoSuchEntity` error — user is gone.

> **If the user still exists:**
> ```
> aws iam list-access-keys --user-name attacker-simulation
> aws iam delete-access-key --user-name attacker-simulation --access-key-id <KEY_ID>
> aws iam delete-user-policy --user-name attacker-simulation --policy-name S3ReadAccess
> aws iam delete-user --user-name attacker-simulation
> ```

### Step 3: Delete Local Files

> **⚠️ Close VS Code first.** If VS Code still has the `workshop-lab-5b` folder open, the delete will fail — especially on Windows. Choose **File → Close Folder** or quit VS Code before running the commands below.

**Windows (PowerShell):**

```powershell
cd ~
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-5b
```

**macOS / Linux:**

```bash
cd ~
rm -rf ~/Desktop/workshop-lab-5b
```

---

## Help

If you get stuck, post in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** (copy and paste it)
4. Your **operating system** (Windows, Mac, or Linux)
