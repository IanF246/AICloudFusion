# Lab 1B: Create a Cost Budget Alert with Email Notification

**Session:** 1 — Cloud Concepts & AWS Global Infrastructure  
**Track:** Cloud Basics  
**Difficulty:** Intermediate  
**Estimated Time:** 25–30 minutes

---

## Overview

In this lab, you will create a **monthly cost budget** in AWS and connect it to an **email notification** so you get alerted if your account starts spending money. This is one of the very first things a cloud engineer sets up on any new AWS account — cost visibility is critical because cloud services can incur charges if left running.

By the end of this lab, you will have:
- An SNS topic (a notification channel) that can send emails
- A budget that monitors your AWS spending
- An automatic email alert that fires if spending approaches your limit

---

## Prerequisites

- ✅ Completed **Lab 1A** (AWS CLI installed and configured)
- ✅ AWS CLI authenticated — run `aws sts get-caller-identity` and confirm it returns your account info
- ✅ Your **AWS account ID** (the 12-digit number you saved in Lab 1A, Step 22)
- ✅ An **email address** you can check during this lab (you will need to click a confirmation link)

---

## Cost Notice

| Service | What It Is | Free Tier Limit |
|---------|-----------|----------------|
| AWS Budgets | Tracks your spending and sends alerts | 2 budgets free per account |
| Amazon SNS | Sends notifications (email, SMS, etc.) | 1,000 email notifications/month free |

**Estimated cost for this lab: $0.00**

---

## Concepts

Before we start, here is what each service does and why we are using it:

**AWS Budgets** is a service that lets you set a spending limit on your AWS account. You tell it "alert me if I spend more than X dollars this month," and it watches your spending for you. Think of it like setting a spending alert on your bank account — you set a threshold, and the bank texts you when you get close.

**Amazon SNS (Simple Notification Service)** is a messaging service. Here is how it works:
1. You create a **"topic"** — think of this as a mailing list
2. You **subscribe** an email address to the topic — this is like signing up for the mailing list
3. Other AWS services (like Budgets) can **send messages** to the topic
4. Everyone subscribed to the topic receives the message

In this lab, we will create an SNS topic, subscribe your email to it, and then tell AWS Budgets to send alerts to that topic whenever your spending gets close to your budget limit.

---

## ⚠️ Reminder: How to Read Commands in This Lab

Commands are inside gray code boxes. **📋 Copy and paste** them into your terminal — do not type them by hand.

**Placeholders** look like `<THIS>` — you must replace them (including the `<` and `>`) with your actual values.

Here are the placeholders you will use in this lab:

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name from Lab 1A | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account number | `123456789012` |
| `<YOUR_EMAIL_ADDRESS>` | An email address you can check right now | `jane.doe@gmail.com` |
| `<YOUR_TOPIC_ARN>` | The TopicArn value returned in Step 3 (you will get this during the lab) | `arn:aws:sns:us-east-1:123456789012:workshop-budget-alert` |
| `<YOUR_SUBSCRIPTION_ARN>` | The SubscriptionArn from the Cleanup step (the topic ARN followed by a long random ID) | `arn:aws:sns:us-east-1:123456789012:workshop-budget-alert:1a2b3c4d-5e6f-7890-abcd-ef1234567890` |

---

## Lab Steps

### Step 1: Set Your AWS Profile

Before running any AWS commands, set your default profile so you do not have to type it on every command.

**Windows (PowerShell):**

📋 Copy and paste this command, **replacing `<YOUR_PROFILE_NAME>`** with your actual profile name from Lab 1A:

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**macOS / Linux:**

```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**Verify it works.** 📋 Copy and paste this and press Enter:

```
aws sts get-caller-identity
```

**✅ You should see** your account ID and role. If you get an error about an expired token, run `aws sso login --profile <YOUR_PROFILE_NAME>` first, then set the profile again.

---

### Step 2: Create Your Project Folder

Before creating any files, let's set up a dedicated folder for this lab. This keeps your files organized and ensures your terminal can find them.

**Step 2a: Create the folder**

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
mkdir ~\Desktop\workshop-lab-1b
cd ~\Desktop\workshop-lab-1b
```

**macOS / Linux:**

📋 Copy and paste:

```bash
mkdir ~/Desktop/workshop-lab-1b
cd ~/Desktop/workshop-lab-1b
```

> **What does this do?** Creates a new folder on your Desktop called `workshop-lab-1b` and moves your terminal into that folder. All commands you run and files you create will be in this folder.

**Step 2b: Verify you're in the right folder**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
pwd
```

**macOS / Linux:**
```bash
pwd
```

**✅ You should see** a path ending in `workshop-lab-1b` (e.g., `C:\Users\YourName\Desktop\workshop-lab-1b` on Windows or `/Users/YourName/Desktop/workshop-lab-1b` on Mac).

> **💡 From now on, save ALL files you create in this lab to this folder.** When the lab says "save the file," save it here. This is where your terminal is looking for files.

---

### Step 3: Create an SNS Topic (Your Notification Channel)

First, we will create an SNS topic. This is the "mailing list" that will receive budget alerts and forward them to your email.

📋 Copy and paste this command exactly as-is (no placeholders to replace here):

```
aws sns create-topic --name workshop-budget-alert --region us-east-1
```

**What does this do?**
- `aws sns create-topic` — tells AWS to create a new SNS topic
- `--name workshop-budget-alert` — gives the topic a name so we can find it later
- `--region us-east-1` — creates it in the US East (N. Virginia) region

**✅ You should see output like this:**

```json
{
    "TopicArn": "arn:aws:sns:us-east-1:123456789012:workshop-budget-alert"
}
```

> **📝 Important: Copy the `TopicArn` value and save it somewhere** (a notepad, sticky note, etc.). You will need it in the next steps. It looks like:
> ```
> arn:aws:sns:us-east-1:123456789012:workshop-budget-alert
> ```
> This is the unique address of your topic — other AWS services use this address to send messages to it.

**✅ Checkpoint — Verify in the AWS Console:**
1. Open your browser and go to [https://console.aws.amazon.com/](https://console.aws.amazon.com/)
2. In the **search bar at the top**, type **SNS** and click **Simple Notification Service**
3. Make sure the **region** in the top-right corner says **US East (N. Virginia)** — if it says something else, click the region dropdown and select **US East (N. Virginia)**
4. In the left sidebar, click **Topics**
5. You should see **workshop-budget-alert** in the list

---

### Step 4: Subscribe Your Email to the Topic

Now we will add your email address to the topic so you receive notifications.

📋 Copy and paste this command, **replacing the two placeholders**:

```
aws sns subscribe --topic-arn <YOUR_TOPIC_ARN> --protocol email --notification-endpoint <YOUR_EMAIL_ADDRESS> --region us-east-1
```

> **🔄 Example:** If your TopicArn is `arn:aws:sns:us-east-1:123456789012:workshop-budget-alert` and your email is `jane.doe@gmail.com`, the command would be:
> ```
> aws sns subscribe --topic-arn arn:aws:sns:us-east-1:123456789012:workshop-budget-alert --protocol email --notification-endpoint jane.doe@gmail.com --region us-east-1
> ```

**What does this do?**
- `aws sns subscribe` — adds a subscriber to the topic
- `--topic-arn <YOUR_TOPIC_ARN>` — which topic to subscribe to
- `--protocol email` — send notifications via email
- `--notification-endpoint <YOUR_EMAIL_ADDRESS>` — the email address to send to

**✅ You should see:**

```json
{
    "SubscriptionArn": "pending confirmation"
}
```

> The status says "pending confirmation" because AWS needs to verify that you actually own this email address. You will confirm it in the next step.

---

### Step 5: Confirm Your Email Subscription

AWS just sent a confirmation email to the address you provided. You need to click the link in that email to activate the subscription.

1. **Open your email inbox** (check the address you used in Step 4)
2. **Look for an email** from **AWS Notifications** with the subject "AWS Notification - Subscription Confirmation"
3. **Check your spam/junk folder** if you do not see it in your inbox
4. **Click the "Confirm subscription" link** in the email
5. A web page will open confirming your subscription is active

> **⏳ If the email has not arrived after 2–3 minutes**, check your spam folder. If it is still not there, go back to Step 4 and run the subscribe command again.

**✅ Checkpoint — Verify in the AWS Console:**
1. Go back to the AWS Console → **SNS** → **Subscriptions** (in the left sidebar)
2. Find the subscription for your email address
3. The **Status** column should say **Confirmed** (not "PendingConfirmation")

---

### Step 6: Create the Budget Definition File

AWS Budgets needs a configuration file that describes your budget. You will create a small JSON file (a structured text file that AWS can read).

> **📝 A note on text editors — we use VS Code.** Any plain-text editor can create these files (Notepad, TextEdit, nano, etc.), so you are free to use whatever you like. **From here on, though, these labs assume you are using [Visual Studio Code (VS Code)](https://code.visualstudio.com/)** — a free editor from Microsoft. We rely on it much more heavily in later sessions, especially the **Infrastructure as Code (IaC)** labs, where a real editor with a file tree, syntax highlighting, and a built-in terminal makes a big difference. Getting comfortable with it now will pay off later.
>
> **Install it (one time only):**
> 1. Go to [https://code.visualstudio.com/](https://code.visualstudio.com/) and click **Download** for your operating system.
> 2. Run the installer and accept the defaults.
> 3. **Windows:** on the "Select Additional Tasks" screen, leave **"Add to PATH"** checked (it already is by default) — this is what lets you launch VS Code from the terminal.
> 4. **macOS:** open VS Code once, press **Cmd+Shift+P**, type **Shell Command: Install 'code' command in PATH**, and press Enter — this enables the `code` command in Terminal.

**Step 6a: Open your project folder in VS Code**

Your terminal is already inside the `workshop-lab-1b` folder from Step 2. 📋 Copy and paste:

```
code .
```

> **What does this do?** `code` launches VS Code, and the `.` (a single dot, meaning "the folder I'm currently in") tells it which folder to open. VS Code starts up with `workshop-lab-1b` as its **file tree** on the left. Every file you create and save here appears in that panel — and, just as importantly, stays in the exact folder your AWS commands read from.
>
> **🔧 If you see `'code' is not recognized` or `command not found`:** the `code` command isn't on your PATH yet. Close your terminal and open a fresh one, then try again. If it still fails, reinstall VS Code with the "Add to PATH" option checked (Windows) or run the "Install 'code' command in PATH" step above (macOS).

**Step 6b: Create the budget file**

1. In the VS Code **file tree** (the left panel), click the **New File** icon and name the file `budget.json`. It opens as an empty tab in the editor.

2. 📋 Copy and paste this entire block into the file:

```json
{
    "BudgetName": "workshop-monthly-budget",
    "BudgetLimit": {
        "Amount": "1",
        "Unit": "USD"
    },
    "BudgetType": "COST",
    "TimeUnit": "MONTHLY"
}
```

3. **Save** the file with **Ctrl+S** (Windows) or **Cmd+S** (Mac). Because you created it inside the file tree, it saves straight into `workshop-lab-1b` — no need to choose a location.

> **⚠️ Common mistake:** Make sure the name is exactly `budget.json`, not `budget.json.txt`. Naming the file in the file tree with a `.json` ending sets the file type automatically — you should see a JSON icon next to it.

> **What does this file do?** It defines your budget configuration — the name, dollar limit, type, and time period that AWS Budgets will use.

**What does each line mean?**

| Field | Value | What It Means |
|-------|-------|---------------|
| `BudgetName` | `workshop-monthly-budget` | A name for your budget — you will use this to find and delete it later |
| `BudgetLimit.Amount` | `1` | The budget limit in dollars — we set $1 so even tiny spending triggers the alert |
| `BudgetLimit.Unit` | `USD` | The currency (US Dollars) |
| `BudgetType` | `COST` | We are tracking actual dollar spending (not usage counts) |
| `TimeUnit` | `MONTHLY` | The budget resets at the start of each month |

---

### Step 7: Create the Notification Configuration File

Now create a second file that tells AWS Budgets **when** to send an alert and **where** to send it.

1. In the VS Code **file tree**, click the **New File** icon and name this file `notifications.json`. It opens as an empty tab.

2. 📋 Copy and paste this entire block into the file, **replacing `<YOUR_TOPIC_ARN>`** with the TopicArn you saved in Step 3:

```json
[
    {
        "Notification": {
            "NotificationType": "ACTUAL",
            "ComparisonOperator": "GREATER_THAN",
            "Threshold": 80,
            "ThresholdType": "PERCENTAGE"
        },
        "Subscribers": [
            {
                "SubscriptionType": "SNS",
                "Address": "<YOUR_TOPIC_ARN>"
            }
        ]
    }
]
```

3. **Save** the file with **Ctrl+S** (Windows) or **Cmd+S** (Mac). It saves into `workshop-lab-1b` alongside `budget.json`, and you should now see both files in the file tree.

> **⚠️ Common mistake:** Make sure the name is exactly `notifications.json`, not `notifications.json.txt`. Naming it in the file tree with a `.json` ending sets the file type automatically.

> **What does this file do?** It tells AWS Budgets when to send an alert (at 80% of your budget) and where to send it (your SNS topic, which forwards to your email).

**What does each part mean?**

| Field | Value | What It Means |
|-------|-------|---------------|
| `NotificationType` | `ACTUAL` | Check real spending (not forecasted/predicted spending) |
| `ComparisonOperator` | `GREATER_THAN` | Alert when spending goes **above** the threshold |
| `Threshold` | `80` | Alert at 80% of the budget limit |
| `ThresholdType` | `PERCENTAGE` | The 80 is a percentage, not a dollar amount |
| `SubscriptionType` | `SNS` | Send the alert to an SNS topic |
| `Address` | Your TopicArn | Which SNS topic to send to (the one with your email subscribed) |

> **In plain English:** "When my actual spending exceeds 80% of my $1 budget ($0.80), send an alert to my SNS topic, which will forward it to my email."

---

### Step 8: Create the Budget

Now you will create the budget using both JSON files.

📋 Copy and paste this command, **replacing `<YOUR_ACCOUNT_ID>`** with your 12-digit AWS account number:

**macOS / Linux:**

```bash
aws budgets create-budget \
    --account-id <YOUR_ACCOUNT_ID> \
    --budget file://budget.json \
    --notifications-with-subscribers file://notifications.json
```

**Windows (PowerShell):**

```powershell
aws budgets create-budget --account-id <YOUR_ACCOUNT_ID> --budget file://budget.json --notifications-with-subscribers file://notifications.json
```

> **🔄 Example:** If your account ID is `123456789012`:
> ```powershell
> aws budgets create-budget --account-id 123456789012 --budget file://budget.json --notifications-with-subscribers file://notifications.json
> ```

**What does this do?**
- `aws budgets create-budget` — tells AWS to create a new budget
- `--account-id` — which AWS account to create the budget in
- `--budget file://budget.json` — reads the budget settings from your file
- `--notifications-with-subscribers file://notifications.json` — reads the alert settings from your file

**✅ If the command succeeds, you will see no output** — the terminal will just show a new prompt. No output means success for this command.

> **🔧 If you see an error:** `An error occurred (DuplicateBudgetException)` — this means a budget with this name already exists. Delete it first with:
> ```
> aws budgets delete-budget --account-id <YOUR_ACCOUNT_ID> --budget-name workshop-monthly-budget
> ```
> Then try the create command again.

---

### Step 9: Verify the Budget Was Created

📋 Copy and paste this command, **replacing `<YOUR_ACCOUNT_ID>`**:

```
aws budgets describe-budgets --account-id <YOUR_ACCOUNT_ID> --output table
```

**What does this do?** This lists all budgets on your account in a readable table format.

**✅ You should see a table** that includes:
- `BudgetName`: `workshop-monthly-budget`
- `BudgetLimit.Amount`: `1.0`
- `TimeUnit`: `MONTHLY`

**✅ Checkpoint — Verify in the AWS Console:**
1. In the AWS Console, search for **Budgets** in the top search bar
2. Click **AWS Budgets**
3. You should see **workshop-monthly-budget** in the list
4. Click on it to see the details — you should see the $1.00 limit and the notification configuration

---

## What You Just Did

You set up **cost monitoring** on an AWS account — one of the first things any cloud engineer does when managing cloud infrastructure. Here is what you built:

1. **An SNS topic** — a notification channel that can send emails
2. **An email subscription** — your email is connected to that channel
3. **A cost budget** — AWS watches your spending and alerts you when it approaches $1

In a real production environment, these alerts prevent surprise bills and are part of the **AWS Well-Architected Framework's Cost Optimization pillar**. Companies set budgets on every AWS account to maintain cost visibility.

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

This lab covers concepts from the **Cost Optimization** pillar of the AWS Well-Architected Framework. The SAA exam tests your ability to design cost-efficient architectures, including setting up billing alarms and budgets.

**Sample question type:** "A company wants to be notified when their monthly AWS spending exceeds a threshold. Which AWS service should they use?"

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `An error occurred (DuplicateBudgetException)` | A budget with this name already exists | Delete it first: `aws budgets delete-budget --account-id <YOUR_ACCOUNT_ID> --budget-name workshop-monthly-budget` then try again |
| Confirmation email not received | AWS sent it but it may be in spam | Check your spam/junk folder. Wait 2–3 minutes. If still nothing, run the subscribe command from Step 4 again. |
| `An error occurred (NotFoundException)` when creating budget | The TopicArn in your notifications.json is wrong | Open `notifications.json` and make sure the `Address` value exactly matches the TopicArn from Step 3. Check for typos or extra spaces. |
| `An error occurred (InvalidParameterException)` | Something in your JSON file is formatted incorrectly | Make sure you saved the files as `.json` (not `.json.txt`). Open them and verify the content matches the examples exactly. |

---

## Cleanup

>[!IMPORTANT]
>**⚠️** Always clean up resources after completing a lab to avoid unexpected charges. Follow these steps in order.

### Step 1: Delete the Budget

📋 Copy and paste this command, **replacing `<YOUR_ACCOUNT_ID>`**:

```
aws budgets delete-budget --account-id <YOUR_ACCOUNT_ID> --budget-name "workshop-monthly-budget"
```

**✅ No output means success.**

### Step 2: Delete the SNS Subscription, Then the Topic

An email subscription is **not** reliably removed when you delete its topic — it can linger and must be deleted explicitly. So you will remove the subscription first, then the topic.

**Step 2a: Find the subscription's ARN**

📋 Copy and paste this command, **replacing `<YOUR_TOPIC_ARN>`**:

```
aws sns list-subscriptions-by-topic --topic-arn <YOUR_TOPIC_ARN> --region us-east-1
```

**✅ You should see output like this:**

```json
{
    "Subscriptions": [
        {
            "SubscriptionArn": "arn:aws:sns:us-east-1:123456789012:workshop-budget-alert:1a2b3c4d-5e6f-7890-abcd-ef1234567890",
            "Owner": "123456789012",
            "Protocol": "email",
            "Endpoint": "jane.doe@gmail.com",
            "TopicArn": "arn:aws:sns:us-east-1:123456789012:workshop-budget-alert"
        }
    ]
}
```

> **📝 Copy the full `SubscriptionArn` value** — you will use it in the next command. Notice it is the topic ARN followed by a long random ID; that trailing ID is what makes it different from your `TopicArn`.
>
> **If `SubscriptionArn` shows `PendingConfirmation`** instead of a full ARN, you never confirmed the email. It cannot be unsubscribed and expires on its own after 3 days — skip to **Step 2c** and just delete the topic.

**Step 2b: Delete the subscription**

📋 Copy and paste this command, **replacing `<YOUR_SUBSCRIPTION_ARN>`** with the value you just copied:

```
aws sns unsubscribe --subscription-arn <YOUR_SUBSCRIPTION_ARN> --region us-east-1
```

**✅ No output means success.**

**Step 2c: Delete the topic**

📋 Copy and paste this command, **replacing `<YOUR_TOPIC_ARN>`**:

```
aws sns delete-topic --topic-arn <YOUR_TOPIC_ARN> --region us-east-1
```

**✅ No output means success.**

### Step 3: Verify Everything Is Gone

📋 Run these two commands to confirm cleanup:

```
aws budgets describe-budgets --account-id <YOUR_ACCOUNT_ID> --output json
```

**✅ You should see** an empty `Budgets` list, or `workshop-monthly-budget` should not appear.

```
aws sns list-topics --region us-east-1
```

**✅ You should see** that `workshop-budget-alert` is no longer in the list.

```
aws sns list-subscriptions --region us-east-1
```

**✅ You should see** no subscription pointing to the `workshop-budget-alert` topic with your email address. (A leftover entry whose `SubscriptionArn` reads `Deleted` is harmless and disappears on its own.)

**✅ Checkpoint — Verify in the AWS Console:**
1. Go to **AWS Budgets** — confirm `workshop-monthly-budget` is gone
2. Go to **SNS → Topics** — confirm `workshop-budget-alert` is gone
3. Go to **SNS → Subscriptions** — confirm no subscription for your email remains

### Step 4: Delete Local Files

> **⚠️ Close VS Code first.** While VS Code has the `workshop-lab-1b` folder open, your operating system treats the folder as "in use" and the delete command below will fail — especially on Windows. Before you delete it, either choose **File → Close Folder** in VS Code or close VS Code entirely. Also note the terminal you run the command in must **not** be sitting inside that folder, which is why the commands below move you to your home directory (`cd ~`) first.

Remove the project folder you created for this lab:

**macOS / Linux:**

```bash
cd ~
rm -rf ~/Desktop/workshop-lab-1b
```

**Windows (PowerShell):**

```powershell
cd ~
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-1b
```

> **🔧 If you still get "cannot remove ... it is being used by another process" (Windows):** VS Code or one of its terminals is still holding the folder open. Close VS Code completely, open a brand-new PowerShell window, and run the two commands above again.

>[!IMPORTANT]
>### Step 5: Time to Redo!
>Now that you’ve successfully cleaned up this lab, it’s strongly recommended to **set up a budget again** for the duration of the project. Associating a budget with any project is considered best practice, and in this case the cost of keeping this budget, SNS topic and subscription active over three months should remain at zero or very close to it. Whereas, the **safeguard it provides in alerting you** in advance if any resources consuming credits are left behind, is invaluable.
>
>To set it back up, simply **repeat Steps 3 through 8** of this lab — create the SNS topic, confirm your email subscription, and recreate the budget. This time, **leave it in place** rather than deleting it at the end, so it keeps watching your account for the rest of the program.

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **command** you ran (copy and paste it)
2. The **full error message** you received (copy and paste it)
3. Which **step number** you are on
