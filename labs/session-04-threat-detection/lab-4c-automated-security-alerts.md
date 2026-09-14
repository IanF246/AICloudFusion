# Lab 4C: Build Automated Security Alerts with EventBridge

**Session:** 4 — Threat Detection & AWS Security Services  
**Track:** Cloud Security & IR  
**Difficulty:** Advanced  
**Estimated Time:** 40–50 minutes  
**Target Cert:** AWS Security Specialty

---

## Overview

In this lab, you will:

1. **Enable GuardDuty** — activate threat detection (if not already enabled from Lab 4A)
2. **Create an SNS topic** — set up a notification channel for security alerts
3. **Subscribe your email** — receive alerts directly in your inbox
4. **Write an EventBridge rule** — define which GuardDuty findings trigger alerts (medium severity and above)
5. **Connect EventBridge to SNS** — route matching events to your notification topic
6. **Test the pipeline** — generate a sample finding and verify you receive an email alert

By the end of this lab, you will have built an automated security alerting pipeline — the foundation of SOAR (Security Orchestration, Automation, and Response). When GuardDuty detects a medium or high severity threat, you get an email automatically.

---

## Prerequisites

- ✅ Completed **Lab 1A** (AWS CLI installed and configured)
- ✅ AWS CLI authenticated — run `aws sts get-caller-identity` and confirm it returns your account info
- ✅ **VS Code** installed (from Lab 1B) — any text editor works, but these labs assume VS Code
- ✅ Access to an email inbox (you will need to confirm a subscription)

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| Amazon GuardDuty | Continuous threat detection | **30-day free trial**, then usage-based (see note) |
| Amazon EventBridge | Event routing service | Free for AWS service events |
| Amazon SNS | Notification service | Free within 1,000 email notifications/month |

**Estimated cost for this lab: $0.00** (within the GuardDuty free trial; EventBridge and SNS stay within their free limits)

**⚠️ Important:** Past the 30-day GuardDuty trial, the detector charges for the data it analyzes — roughly **$4.00 per million CloudTrail management events** plus **$1.00/GB** of VPC Flow Log + DNS log analysis (us-east-1), which on an idle learning account is about **a few cents to ~$1–2/month**. Small, but ongoing — so you **MUST** delete the detector in the cleanup section.

---

## Concepts

Before we start, here is what each concept means:

**Amazon EventBridge** is a serverless event bus that routes events between AWS services. When something happens in AWS (like GuardDuty detecting a threat), EventBridge can automatically trigger an action (like sending a notification). Think of it as a smart mail sorter — it reads every event, checks if it matches your rules, and routes matching events to the right destination.

**Event Pattern** is a JSON filter that tells EventBridge which events to match. You define the source (which service), the detail-type (what kind of event), and conditions on the event details (like severity level). Only events matching your pattern trigger the rule.

**Amazon SNS (Simple Notification Service)** is a messaging service that sends notifications to subscribers. You create a "topic" (a channel), subscribe endpoints to it (email, SMS, Lambda, etc.), and then publish messages to the topic. All subscribers receive the message.

**SOAR (Security Orchestration, Automation, and Response)** is the practice of automating security workflows. Instead of a human manually checking GuardDuty every hour, you build automated pipelines that detect threats and respond immediately — sending alerts, isolating resources, or triggering remediation scripts.

**Event-Driven Architecture** means services react to events as they happen, rather than polling for changes. GuardDuty generates a finding → EventBridge detects it → SNS sends an alert. No human intervention required. This is how modern security operations scale.

**Topic Policy** is an access control document on an SNS topic that defines who can publish messages to it. EventBridge needs permission to publish to your SNS topic, so you must add a policy that allows the `events.amazonaws.com` service to call `sns:Publish`.

---

## ⚠️ Reminder: How to Read Commands in This Lab

Commands are inside gray code boxes. **📋 Copy and paste** them into your terminal.

**Placeholders** look like `<THIS>` — replace them (including the `<` and `>`) with your actual values.

Here are the placeholders you will use in this lab:

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name from Lab 1A | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account ID (from `aws sts get-caller-identity`) | `123456789012` |
| `<DETECTOR_ID>` | The GuardDuty detector ID (from Step 4) | `a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6` |
| `<TOPIC_ARN>` | The SNS topic ARN (from Step 5) | `arn:aws:sns:us-east-1:123456789012:workshop-security-alerts` |
| `<YOUR_EMAIL>` | Your email address for receiving alerts | `you@example.com` |

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
mkdir ~\Desktop\workshop-lab-4c
cd ~\Desktop\workshop-lab-4c
```

**macOS / Linux:**

📋 Copy and paste:

```bash
mkdir ~/Desktop/workshop-lab-4c
cd ~/Desktop/workshop-lab-4c
```

> **What does this do?** Creates a new folder on your Desktop called `workshop-lab-4c` and moves your terminal into that folder.

**Step 2b: Verify you're in the right folder**

📋 Copy and paste:

```
pwd
```

**✅ You should see** a path ending in `workshop-lab-4c`.

> **💡 From now on, save ALL files you create in this lab to this folder.**

**Step 2c: Open the folder in VS Code**

📋 Copy and paste:

```
code .
```

> **What does this do?** This opens VS Code with `workshop-lab-4c` as its **file tree** on the left, so the JSON files you create in Steps 8, 9, and 11 land in the right place. (You set up the `code` command in Lab 1B — if you see `'code' is not recognized`, close and reopen your terminal, or revisit Lab 1B, Step 6.)

---

### Step 3: Check If GuardDuty Is Already Enabled

📋 Copy and paste:

```
aws guardduty list-detectors --region us-east-1
```

**✅ If you see** `"DetectorIds": []` (empty list) — GuardDuty is NOT enabled. Continue to Step 4.

**✅ If you see** a detector ID in the list — GuardDuty is already enabled. Write down that detector ID and skip to Step 5.

---

### Step 4: Enable GuardDuty

📋 Copy and paste:

```
aws guardduty create-detector --enable --region us-east-1
```

**✅ You should see** JSON output containing a `DetectorId`:

```json
{
    "DetectorId": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6"
}
```

> **📝 Write down your Detector ID:** ______________________________

---

### Step 5: Create an SNS Topic for Security Alerts

📋 Copy and paste:

```
aws sns create-topic --name workshop-security-alerts --region us-east-1
```

**✅ You should see** JSON output with the topic ARN:

```json
{
    "TopicArn": "arn:aws:sns:us-east-1:123456789012:workshop-security-alerts"
}
```

> **📝 Write down your Topic ARN:** ______________________________
>
> You will need this in several steps below.

> **What does this do?** Creates a notification channel called `workshop-security-alerts`. Think of it as creating a mailing list — anyone subscribed to this topic will receive messages published to it.

---

### Step 6: Subscribe Your Email to the Topic

📋 Copy and paste, **replacing `<TOPIC_ARN>` and `<YOUR_EMAIL>`**:

```
aws sns subscribe --topic-arn <TOPIC_ARN> --protocol email --notification-endpoint <YOUR_EMAIL> --region us-east-1
```

> **🔄 Example:**
> ```
> aws sns subscribe --topic-arn arn:aws:sns:us-east-1:123456789012:workshop-security-alerts --protocol email --notification-endpoint jane@example.com --region us-east-1
> ```

**✅ You should see** JSON output with a `SubscriptionArn` of `"pending confirmation"`.

---

### Step 7: Confirm the Email Subscription

**⚠️ This step requires checking your email inbox.**

1. Open your email inbox (the address you used in Step 6)
2. Look for an email from **AWS Notifications** with subject "AWS Notification - Subscription Confirmation"
3. Click the **Confirm subscription** link in the email
4. You should see a confirmation page saying "Subscription confirmed!"

> **💡 If you do not see the email:** Check your spam/junk folder. The email usually arrives within 1–2 minutes. If it still does not arrive, run the subscribe command again.

**Verify the subscription is confirmed:**

📋 Copy and paste, **replacing `<TOPIC_ARN>`**:

```
aws sns list-subscriptions-by-topic --topic-arn <TOPIC_ARN> --region us-east-1 --query "Subscriptions[].{Endpoint:Endpoint,Protocol:Protocol,Status:SubscriptionArn}" --output table
```

**✅ You should see** your email listed with a full ARN (not "PendingConfirmation").

---

### Step 8: Write the SNS Topic Policy

EventBridge needs permission to publish messages to your SNS topic. You must add a policy that allows the EventBridge service to call `sns:Publish`.

**Step 8a:** In the VS Code file tree, click the **New File** icon and name the file `sns-policy.json`.

**Step 8b:** 📋 Copy and paste this entire block into it, **replacing `<TOPIC_ARN>`** (2 places) and **`<YOUR_ACCOUNT_ID>`** (1 place):

```json
{
    "Version": "2012-10-17",
    "Id": "AllowEventBridgePublish",
    "Statement": [
        {
            "Sid": "AllowAccountToManage",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<YOUR_ACCOUNT_ID>:root"
            },
             "Action": [
            "SNS:Publish",
            "SNS:Subscribe",
            "SNS:Receive",
            "SNS:GetTopicAttributes",
            "SNS:SetTopicAttributes",
            "SNS:AddPermission",
            "SNS:RemovePermission",
            "SNS:ListSubscriptionsByTopic"
            ],
            "Resource": "<TOPIC_ARN>"
        },
        {
            "Sid": "AllowEventBridgeToPublish",
            "Effect": "Allow",
            "Principal": {
                "Service": "events.amazonaws.com"
            },
            "Action": "sns:Publish",
            "Resource": "<TOPIC_ARN>"
        }
    ]
}
```

> **🔄 Example:** If your topic ARN is `arn:aws:sns:us-east-1:123456789012:workshop-security-alerts` and your account ID is `123456789012`:
> - `"AWS": "arn:aws:iam::123456789012:root"`
> - `"Resource": "arn:aws:sns:us-east-1:123456789012:workshop-security-alerts"` (in both places)

**Step 8c:** **Save** the file (**Ctrl+S** / **Cmd+S**). You should see `sns-policy.json` appear in the file tree.

> **⚠️ Common mistake:** Make sure you replaced `<TOPIC_ARN>` in BOTH places and `<YOUR_ACCOUNT_ID>` in 1 place, and that the file is named exactly `sns-policy.json`.

> **What does this file do?** It grants two permissions:
> - **AllowAccountToManage** — your account can manage the topic (needed for cleanup)
> - **AllowEventBridgeToPublish** — the EventBridge service can publish messages to this topic

**Step 8d: Apply the SNS topic policy**

📋 Copy and paste, **replacing `<TOPIC_ARN>`**:

```
aws sns set-topic-attributes --topic-arn <TOPIC_ARN> --attribute-name Policy --attribute-value file://sns-policy.json --region us-east-1
```

**✅ No output means success.**

---

### Step 9: Write the EventBridge Event Pattern

Now you will define which GuardDuty findings should trigger an alert. You will match findings with severity >= 4 (medium and above).

**Step 9a:** In the VS Code file tree, create a **New File** named `eventbridge-rule.json`.

**Step 9b:** 📋 Copy and paste this entire block into it (no placeholders to replace):

```json
{
    "source": ["aws.guardduty"],
    "detail-type": ["GuardDuty Finding"],
    "detail": {
        "severity": [{"numeric": [">=", 4]}]
    }
}
```

**Step 9c:** **Save** the file (**Ctrl+S** / **Cmd+S**).

> **What does this file do?** It tells EventBridge: "Match any event that comes from GuardDuty, is a finding, and has a severity of 4 or higher." This means:
> - **Low severity (1–3.9):** Ignored — no alert sent
> - **Medium severity (4–6.9):** Alert sent ✅
> - **High severity (7–8.9):** Alert sent ✅
>
> This prevents alert fatigue from low-priority findings while ensuring you are notified of real threats.

> **What does each field mean?**
>
> | Field | Value | What It Means |
> |-------|-------|---------------|
> | `source` | `aws.guardduty` | Only match events from GuardDuty |
> | `detail-type` | `GuardDuty Finding` | Only match finding events (not configuration changes) |
> | `detail.severity` | `>= 4` | Only match findings with severity 4 or higher |

---

### Step 10: Create the EventBridge Rule

📋 Copy and paste:

```
aws events put-rule --name workshop-guardduty-alert --event-pattern file://eventbridge-rule.json --state ENABLED --region us-east-1
```

**✅ You should see** JSON output with the rule ARN:

```json
{
    "RuleArn": "arn:aws:events:us-east-1:123456789012:rule/workshop-guardduty-alert"
}
```

> **What does this do?** Creates an EventBridge rule that continuously monitors for GuardDuty findings matching your pattern. When a match is found, it triggers the targets (which you will add next).

---

### Step 11: Add the SNS Topic as the Target

Now connect the rule to your SNS topic — when the rule matches a finding, it publishes to the topic, which sends you an email. Passing this configuration inline is fiddly (the quoting differs between PowerShell and Bash and breaks easily), so you'll put it in a small file — the reliable, cross-platform way.

**Step 11a: Create the targets file**

In the VS Code file tree, create a **New File** named `targets.json`. 📋 Copy and paste this into it, **replacing `<TOPIC_ARN>`**:

```json
[
    {
        "Id": "sns-target",
        "Arn": "<TOPIC_ARN>"
    }
]
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

**Step 11b: Add the target to the rule**

📋 Copy and paste:

```
aws events put-targets --rule workshop-guardduty-alert --targets file://targets.json --region us-east-1
```

**✅ You should see** JSON output with `"FailedEntryCount": 0` — meaning the target was added successfully.

---

### Step 12: Test the Automated Alert Pipeline

Now generate a high-severity sample finding in GuardDuty to trigger your alert pipeline.

📋 Copy and paste, **replacing `<DETECTOR_ID>`**:

```
aws guardduty create-sample-findings --detector-id <DETECTOR_ID> --finding-types "CryptoCurrency:EC2/BitcoinTool.B!DNS" --region us-east-1
```

**✅ No output means success.**

> **What happens now?**
> 1. GuardDuty generates a sample finding with high severity (8.0)
> 2. GuardDuty publishes the finding as an event to EventBridge
> 3. EventBridge checks the event against your rule (severity >= 4? YES)
> 4. EventBridge publishes the event to your SNS topic
> 5. SNS sends an email to your subscribed address
>
> **⏱️ This may take 2–5 minutes.** Check your email inbox (and spam folder) for a notification from AWS.

---

### Step 13: Verify the Email Alert

1. Check your email inbox for a message from **AWS Notifications**
2. The email will contain the GuardDuty finding details in JSON format, including:
   - The finding type (`CryptoCurrency:EC2/BitcoinTool.B!DNS`)
   - The severity (8.0 — High)
   - A description of the threat
   - The affected resource

> **💡 If you do not receive an email within 5 minutes:**
> - Check your spam/junk folder
> - Verify your subscription is confirmed (Step 7)
> - Try generating another sample finding (repeat Step 12)
> - Note: Sample findings may not always trigger EventBridge rules in all accounts. The important thing is that you built the pipeline correctly.

> **💡 What the email looks like:** The email contains raw JSON from the GuardDuty finding event. In a production environment, you would use a Lambda function between EventBridge and SNS to format the message into a human-readable alert with clear action items.

Guardduty has a set behaviour regarding deduplication: If you do a second run (within 24hrs after cleanup), even though you deleted the detector and rebuilt everything, GuardDuty's backend will still have a recent record of that finding type for your account. When you call create-sample-findings again with the same type, GuardDuty will treat it as an update to the existing finding (incrementing the count, updating the timestamp) rather than creating a new finding event. EventBridge only fires on new finding events, not on updates — so no email will be triggered.

TLDR: GuardDuty deduplicates findings by type within a time window. This is by design to prevent alert fatigue — a single compromised resource triggering the same behavior 100 times generates one finding (updated), not 100 emails. In a real environment, you'd see the UpdatedAt and Count fields on the finding change, but only the initial detection fires through EventBridge.

---

### Step 14: Console Checkpoint

Let's verify the pipeline in the AWS Console:

**EventBridge Rule:**
1. Go to [https://console.aws.amazon.com/](https://console.aws.amazon.com/)
2. Search for **EventBridge** in the top search bar and click it
3. Click **Rules** in the left sidebar
4. You should see **workshop-guardduty-alert** with status **Enabled**
5. Click on the rule to see the event pattern and target

**SNS Topic:**
1. Search for **SNS** in the top search bar and click it
2. Click **Topics** in the left sidebar
3. You should see **workshop-security-alerts**
4. Click on the topic to see your email subscription

**GuardDuty Findings:**
1. Search for **GuardDuty** in the top search bar and click it
2. Click **Findings** in the left sidebar
3. You should see the sample findings with severity indicators

**✅ Checkpoint:** You can see all three components of your automated alert pipeline — GuardDuty (detection), EventBridge (routing), and SNS (notification). This is a basic SOAR implementation.

---

## What You Just Did

You built an automated security alerting pipeline:

1. **GuardDuty** continuously monitors for threats (detection layer)
2. **EventBridge** filters findings by severity and routes them (orchestration layer)
3. **SNS** delivers alerts to your email (notification layer)

This is the foundation of **SOAR (Security Orchestration, Automation, and Response)**:

| SOAR Component | What You Built | Production Version |
|---------------|---------------|-------------------|
| **Orchestration** | EventBridge rule matching severity >= 4 | Complex rules matching specific finding types, resources, or accounts |
| **Automation** | Automatic event routing (no human needed) | Lambda functions that auto-remediate (isolate instances, revoke keys) |
| **Response** | Email notification to analyst | PagerDuty alerts, Slack messages, JIRA tickets, automated runbooks |

In a real SOC, this pipeline would:
- Send high-severity alerts to PagerDuty (wakes someone up at 3 AM)
- Send medium-severity alerts to a Slack channel (investigated during business hours)
- Trigger Lambda functions that automatically isolate compromised resources
- Create tickets in a case management system for tracking

---

## Cert Prep Callout

**Target Certification:** AWS Security Specialty (SCS-C02)

This lab covers several high-weight exam topics:
- **EventBridge event patterns:** How to filter events by source, type, and detail fields
- **GuardDuty integration:** How GuardDuty findings flow to EventBridge automatically
- **SNS for security notifications:** Topic policies, subscriptions, and cross-service permissions
- **Automated incident response:** Building pipelines that respond to threats without human intervention

**Sample question type:** "A security team wants to be notified immediately when GuardDuty detects a high-severity finding. Which combination of services should they use?"  
**Answer:** Amazon EventBridge (to filter findings by severity) + Amazon SNS (to send notifications) — or EventBridge + Lambda for automated remediation.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `An error occurred (InvalidParameterValue)` when creating the rule | The event pattern JSON has a syntax error | Open `eventbridge-rule.json` and verify the JSON is valid. Check for missing commas or brackets. |
| `put-targets` returns `FailedEntryCount: 1` | EventBridge cannot access the SNS topic | Verify the SNS topic policy was applied correctly (Step 8d). Check that the topic ARN is correct. |
| Email subscription confirmation not received | Email may be in spam, or the address was typed wrong | Check spam folder. If not there, unsubscribe and re-subscribe with the correct email. |
| No alert email after generating sample findings | Sample findings may not always trigger EventBridge | Verify: (1) subscription is confirmed, (2) rule is ENABLED, (3) topic policy allows EventBridge. Try waiting 5 more minutes. |
| `A detector already exists for the current account` | GuardDuty is already enabled | Run `aws guardduty list-detectors --region us-east-1` to get the existing detector ID |
| `An error occurred (ResourceNotFoundException)` for the rule | The rule name is wrong or in the wrong region | Verify you are in us-east-1 and the rule name is exactly `workshop-guardduty-alert` |

---

## Cleanup

>[!IMPORTANT]
>**⚠️** Always clean up resources after completing a lab. Follow these steps in order.

> **🔗 Continuing to the 4D sidequest?** 4D extends this exact pipeline — it inserts a Lambda function between EventBridge and SNS. If you're moving straight to 4D, **skip this cleanup for now and leave the GuardDuty detector, SNS topic, and EventBridge rule in place** — you'll reuse them there. Come back and run these steps after you finish 4D. (Reminder: leaving GuardDuty enabled past its trial carries the small ongoing cost noted at the top of this lab.)

### Step 1: Remove EventBridge Targets

📋 Copy and paste:

```
aws events remove-targets --rule workshop-guardduty-alert --ids sns-target --region us-east-1
```

**✅ No output means success.**

### Step 2: Delete the EventBridge Rule

📋 Copy and paste:

```
aws events delete-rule --name workshop-guardduty-alert --region us-east-1
```

**✅ No output means success.**

### Step 3: Delete SNS Subscriptions

First, get the subscription ARN:

📋 Copy and paste, **replacing `<TOPIC_ARN>`**:

```
aws sns list-subscriptions-by-topic --topic-arn <TOPIC_ARN> --region us-east-1 --query "Subscriptions[].SubscriptionArn" --output text
```

Then unsubscribe (📋 **replacing `<SUBSCRIPTION_ARN>`** with the ARN from above):

```
aws sns unsubscribe --subscription-arn <SUBSCRIPTION_ARN> --region us-east-1
```

**✅ No output means success.**

### Step 4: Delete the SNS Topic

📋 Copy and paste, **replacing `<TOPIC_ARN>`**:

```
aws sns delete-topic --topic-arn <TOPIC_ARN> --region us-east-1
```

**✅ No output means success.**

### Step 5: Delete the GuardDuty Detector

📋 Copy and paste, **replacing `<DETECTOR_ID>`**:

```
aws guardduty delete-detector --detector-id <DETECTOR_ID> --region us-east-1
```

**✅ No output means success.**

### Step 6: Verify GuardDuty Is Disabled

📋 Copy and paste:

```
aws guardduty list-detectors --region us-east-1
```

**✅ You should see** `"DetectorIds": []` (empty list).

### Step 7: Delete Local Files

> **⚠️ Close VS Code first.** If VS Code still has the `workshop-lab-4c` folder open, the delete will fail — especially on Windows. Choose **File → Close Folder** or quit VS Code before running the commands below.

Remove the project folder:

**Windows (PowerShell):**

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-4c
```

**macOS / Linux:**

```bash
cd ~/Desktop
rm -rf ~/Desktop/workshop-lab-4c
```

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** you received (copy and paste it)
4. Your **operating system** (Windows, Mac, or Linux)
