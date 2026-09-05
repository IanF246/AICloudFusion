# Lab 1A: Create Your AWS Account & Set Up Secure CLI Access

**Session:** 1 — Cloud Concepts & AWS Global Infrastructure  
**Track:** Cloud Basics  
**Difficulty:** Beginner  
**Estimated Time:** 40–50 minutes

---

## Overview

In this lab, you will:

1. **Create your own AWS account** — your personal cloud environment
2. **Enable IAM Identity Center** — the modern, secure way to manage access
3. **Create a user for yourself** — so you never log in as the root (owner) account
4. **Install the AWS CLI** — the command-line tool cloud engineers use every day
5. **Configure SSO** — so the CLI connects to your account securely

By the end of this lab, you will have a fully configured AWS account with secure access from both the web console and the command line. This is the foundation for every lab in this program.

---

## Prerequisites

- An **email address** you have access to (for account verification)
- A **credit or debit card** (required by AWS to verify your identity — see the **Cost Notice** below for what this program actually costs; most labs stay at or near $0.00)
- A **phone number** (for identity verification during signup)
- A computer running **Windows, macOS, or Linux**

---

## Cost Notice

| Service | What It Is | Cost | 
|---------|-----------|------|
| AWS Account | Your cloud environment | Free to create |
| IAM Identity Center | Secure login management | Always Free |
| AWS Organizations | Groups AWS accounts together | Always Free |
| AWS CLI | A program on your computer | Always Free |

**Estimated cost for _this_ lab (1A): $0.00**

---

>[!IMPORTANT]
>### 💰 Read this before you start

**New AWS accounts come with free credits — but this program's setup gives them up. That is expected, and here is why.**

When you sign up in 2026, AWS offers new customers a Free Tier worth **up to $200 in credits**: an initial **$100**, plus another **$100** you can earn by completing AWS's onboarding activities. There is a catch that matters for us:

- This program uses **IAM Identity Center** for secure sign-ins (you will set it up in Part 2 of this lab), and Identity Center requires **AWS Organizations**.
- **The moment you enable AWS Organizations, AWS forfeits those free credits.** For a brand-new account, this happens as soon as you complete Part 2 — so plan as if the credits are not there.

There *is* a workaround that keeps the credits — using an old-style personal IAM user with long-lived access keys instead of Identity Center — but it is rarely used in real companies, is less secure, and works against the whole point of this program: giving you practical, hands-on experience with how cloud teams actually operate. So we don't use it.

**What this means for your wallet:**

- At signup you will choose the **Paid plan** (Step 7). This does **not** charge you anything up front — you only pay for what you actually use, and most of what we use falls under AWS's **Always Free** limits.
- The costs that do come up are small. For example, Session 2 costs about **$3.00** for the month *if* you forget to clean up your EC2 instance and leave it running the entire month. Follow each lab's cleanup steps and you will pay far less.
- Every lab from here on lists its **estimated monthly cost** up front, so there are no surprises.
- In **Lab 1B**, you will set up a **billing budget alert** — a safety net that emails you if your usage ever climbs toward a threshold you set.

---

## ⚠️ How to Read These Labs

Throughout these labs, you will see commands inside gray code boxes like this:

```
aws --version
```

**📋 Copy and paste** these commands into your terminal. You do not need to type them by hand — highlight the text, copy it (Ctrl+C on Windows, Cmd+C on Mac), then paste it into your terminal (right-click in PowerShell, or Cmd+V in Mac Terminal) and press **Enter**.

You will also see **placeholders** — these are values you need to replace with your own information. They always look like this:

```
<YOUR_PROFILE_NAME>
```

Placeholders are wrapped in `< >` angle brackets and written in UPPER_CASE. **You must replace the entire placeholder, including the `<` and `>` characters**, with your actual value.

**Example:**
- The lab says: `aws sso login --profile <YOUR_PROFILE_NAME>`
- Your profile name is `AdministratorAccess-123456789012`
- You would type: `aws sso login --profile AdministratorAccess-123456789012`

---

## Part 1: Create Your AWS Account

### Step 1: Start the Signup Process

1. Open your web browser and go to: [https://aws.amazon.com/](https://aws.amazon.com/)
2. Click the **Create an AWS Account** button (top-right corner)
3. You will be taken to the signup page

---

### Step 2: Enter Your Email and Account Name

1. In the **Root user email address** field, enter your email address
2. In the **AWS account name** field, enter a name for your account (e.g., `My Cloud Workshop Account`)
3. Click **Verify email address**
4. Check your email inbox for a **verification code** from AWS
5. Enter the verification code on the signup page and click **Verify**

>[!NOTE]
> **💡 What is a "root user"?** The root user is the owner of the AWS account. It has full, unrestricted access to everything. Think of it like the master key to a building — you should rarely use it. Later in this lab, you will create a separate user for your day-to-day work, which is a security best practice.

---

### Step 3: Create a Password

1. Enter a **strong password** for your root user account
2. Confirm the password
3. Click **Continue**

> **📝 Write down or save this password securely.** You will need it if you ever need to access the root account (which should be rare after this lab).

**Password requirements:**
- At least 8 characters
- Mix of uppercase, lowercase, numbers, and symbols (e.g., `! @ # $ %`)

---

### Step 4: Enter Your Contact Information

1. Choose **Personal** (unless this is for a business)
2. Fill in your name, phone number, and address
3. Read and accept the **AWS Customer Agreement**
4. Click **Continue**

---

### Step 5: Enter Payment Information

1. Enter your **credit or debit card** information
2. Click **Verify and Continue**

> **⚠️ Will I be charged now?** No — AWS requires a payment method to verify your identity and prevent fraud; adding a card does not trigger a charge. You only pay for what you actually use, and if you follow each lab's cleanup steps your costs stay at or near $0.00. See the **Cost Notice** above for the details.

---

### Step 6: Verify Your Identity

1. Choose how you want to receive a verification code: **Text message (SMS)** or **Voice call**
2. Enter your phone number
3. Complete the CAPTCHA
4. Enter the PIN you receive and click **Continue**

---

### Step 7: Select Your Plan

You will see two options: **Free** and **Paid**.

**Select the Paid plan.**

> **💡 Why Paid and not Free?** The Free plan restricts which services you can use — including the secure login system (IAM Identity Center) this program relies on — and it automatically closes after six months or once its credits run out, which we don't want happening mid-program. The Paid plan removes those limits and only charges you for what you actually use, which we keep to a minimum (see the **Cost Notice** near the top of this lab).

---

### Step 8: Select a Support Plan

1. Choose **Basic support - Free**
2. Click **Complete sign up**

>[!WARNING]
> **⚠️ Choose Basic support — nothing else.** Basic support is free. Every other support plan carries a **fixed monthly charge starting at $29 USD**, billed whether or not you use it. For these labs you only ever need **Basic support - Free**, so make sure it is the option selected before you continue.

---

### Step 9: Wait for Account Activation

1. You will see a confirmation page saying your account is being activated
2. Check your email for an **activation confirmation** from AWS
3. Activation usually takes a few minutes but can take up to 24 hours

> **⏳ If your account is not activated after a few minutes**, you can still proceed to the next step — most accounts activate within 1–2 minutes.

**✅ Checkpoint:** Once activated, sign in to the [AWS Management Console](https://console.aws.amazon.com/) using your **root user email** and **password**. You should see the AWS Console homepage.


---

## Part 2: Enable IAM Identity Center

Now that you have an AWS account, you will set up **IAM Identity Center** — the modern, secure way to manage who can access your account. This replaces the old method of creating IAM users with long-lived access keys.

> **💡 Why are we doing this?** In the real world, companies never let employees log in as the root user. Instead, they use Identity Center to create individual users with specific permissions. We are setting this up the same way a real company would.

### Step 10: Open IAM Identity Center

1. Make sure you are signed in to the [AWS Console](https://console.aws.amazon.com/) as the **root user**
2. In the **search bar at the top** of the page, type **IAM Identity Center**
3. Click **IAM Identity Center** from the search results
4. You will see a page that says **Enable IAM Identity Center**
5. At the top right section of the page next to your username, select **N. Virginia us-east-1** as your region.

---

### Step 11: Enable IAM Identity Center

1. Click the **Enable** button
2. On the next page, you will see a message about **AWS Organizations**. This is normal — Identity Center needs Organizations to work.

> **💡 What is AWS Organizations?** AWS Organizations is a service that lets you group and manage multiple AWS accounts. When you enable Identity Center, it automatically creates an Organization for your account. This is free and does not add any charges.

3. Click **Enable** (or **Create AWS organization** if prompted)
4. Wait a few seconds for the setup to complete

**✅ Checkpoint:** You should now see the **IAM Identity Center dashboard**. It will show your **AWS access portal URL** — this is the web address where you (and any future users) will log in. It looks like:

```
https://d-xxxxxxxxxx.awsapps.com/start
```

> **📝 Write down your AWS access portal URL:** ______________________________
>
> You will need this later when configuring the CLI.

---

## Part 3: Create a User, Group, and Permission Set

Now you will create a user for yourself (so you stop using the root account), a group to organize users, and a permission set that defines what the user can do.

### Step 12: Create a User

1. In the IAM Identity Center console, click **Users** in the left sidebar
2. Click **Add user**
3. Fill in the following:

| Field | What to Enter |
|-------|--------------|
| **Username** | A username for yourself (e.g., `your-first-name` or your email) |
| **Email address** | Your email address (can be the same one you used for the AWS account) |
| **First name** | Your first name |
| **Last name** | Your last name |
| **Display name** | This auto-fills from your first and last name |

4. Click **Next**
5. On the next page (Add user to groups), **skip this for now** — click **Next** again
6. Review the details and click **Add user**

> **📧 Check your email!** You will receive an email from AWS with the subject **"Invitation to join AWS IAM Identity Center"**. This email contains a link to set your password. **Click the link and create a password for this new user.** This is a different password from your root account password.

> **📝 Write down your Identity Center username:** ______________________________
>
> **📝 Write down your Identity Center password:** ______________________________

---

### Step 13: Create a Group

Groups make it easier to manage permissions. Instead of assigning permissions to each user individually, you assign them to a group, and anyone in the group gets those permissions.

1. In the left sidebar, click **Groups**
2. Click **Create group**
3. For **Group name**, enter: `Administrators`
4. Under **Add users to group**, check the box next to the user you just created
5. Click **Create group**

**✅ Checkpoint:** You should see the `Administrators` group in the list with 1 member.

---

### Step 14: Create a Permission Set

A permission set defines what actions a user is allowed to perform. We will create one that gives full administrator access.

1. In the left sidebar, click **Permission sets** (under **Multi-account permissions**)
2. Click **Create permission set**
3. On the first page:
   - Select **Predefined permission set**
   - From the dropdown, choose **AdministratorAccess**
   - Click **Next**
4. On the next page:
   - Leave the **Session duration** as the default (1 hour) or increase it to **4 hours** for convenience
   - Click **Next**
5. Review and click **Create**

> **💡 What is AdministratorAccess?** This is a pre-built AWS policy that grants full access to all AWS services. In a real company, you would create more restrictive permission sets (e.g., "read-only" or "S3-only"). For this workshop, we use AdministratorAccess so you can complete all labs without permission issues.

**✅ Checkpoint:** You should see `AdministratorAccess` in the permission sets list.

---

### Step 15: Assign the Group to Your AWS Account

Now you need to connect the group (with its permission set) to your AWS account.

1. In the left sidebar, click **AWS accounts** (under **Multi-account permissions**)
2. You should see your AWS account listed. **Check the box** next to it
3. Click **Assign users or groups**
4. Click the **Groups** tab
5. Check the box next to **Administrators**
6. Click **Next**
7. Check the box next to **AdministratorAccess** (the permission set you created)
8. Click **Next**
9. Review and click **Submit**

**✅ Checkpoint:** Your account should now show the `Administrators` group with the `AdministratorAccess` permission set assigned.

---

### Step 16: Test Your New User Login

Let's verify that your new user can log in through the AWS access portal.

1. **Open a new browser window** (or an incognito/private window)
2. Go to your **AWS access portal URL** (the one you wrote down in Step 11, e.g., `https://d-xxxxxxxxxx.awsapps.com/start`)
3. Enter the **username** you created in Step 12
4. Enter the **password** you set when you clicked the email invitation link
5. You should see your AWS account listed with the **AdministratorAccess** role
6. Click on the account, then click **Management console** to open the AWS Console

**✅ Checkpoint:** You are now logged in to the AWS Console as your Identity Center user (not the root user). You should see the console homepage. Check the top-right corner — it should show your username and the role `AdministratorAccess`.

>[!TIP]
> **🎉 From now on, always use this login method** (the access portal URL) instead of signing in as the root user. The root user should only be used for account-level tasks like changing your payment method or closing the account.

---

## Part 4: Install and Configure the AWS CLI

### Step 17: Open Your Terminal

**Windows:**
1. Click the **Start** button (Windows icon in the bottom-left corner)
2. Type **PowerShell**
3. Click **Windows PowerShell** to open it

**macOS:**
1. Press **Cmd + Space** to open Spotlight Search
2. Type **Terminal** and press **Enter**

**Linux:**
1. Press **Ctrl + Alt + T** to open a terminal

> **💡 Keep this terminal window open** for the rest of the lab.

---

### Step 18: Install the AWS CLI

**Windows:**

1. Open your browser and go to: [https://awscli.amazonaws.com/AWSCLIV2.msi](https://awscli.amazonaws.com/AWSCLIV2.msi)
2. A file will download. Find it in your Downloads folder and **double-click** it
3. Click **Next** through each screen, then click **Install**
4. When it finishes, click **Finish**
5. **Close your PowerShell window completely** (click the X) and **open a new one** — the installer needs a fresh terminal

**macOS:**

📋 Copy and paste these commands into your terminal, one at a time, pressing Enter after each:

```bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
```

> **What does this do?** Downloads the AWS CLI installer to your computer.

```bash
sudo installer -pkg AWSCLIV2.pkg -target /
```

> **What does this do?** Installs the AWS CLI. You may be asked for your computer password — type it and press Enter (the password won't appear on screen, which is normal).

**Linux:**

📋 Copy and paste these commands one at a time:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```

```bash
unzip awscliv2.zip
```

```bash
sudo ./aws/install
```

---

### Step 19: Verify the Installation

📋 Copy and paste this command and press Enter:

```
aws --version
```

**What does this do?** Asks the AWS CLI to report its version. If it responds, the installation worked.

**✅ You should see something like:**

```
aws-cli/2.x.x Python/3.x.x <your-os>
```

> **🔧 If you see `'aws' is not recognized` or `command not found`:** Close your terminal completely and open a brand new one. If it still doesn't work, restart your computer.

---

### Step 20: Configure SSO in the CLI

Now you will connect the CLI to your AWS account using the Identity Center SSO you set up earlier.

**Before you start, make sure you have:**
- ✅ Your **AWS access portal URL** from Step 11 (e.g., `https://d-xxxxxxxxxx.awsapps.com/start`)

📋 Copy and paste this command and press Enter:

```
aws configure sso
```

**What does this do?** Starts an interactive setup wizard that connects your CLI to your AWS account via Identity Center.

**The terminal will ask you questions one at a time. Here is exactly what to type:**

| The terminal shows this prompt | What you type and press Enter |
|-------------------------------|-------------------------------|
| `SSO session name (Recommended):` | `my-sso` |
| `SSO start URL [None]:` | **Paste your AWS access portal URL** from Step 11 |
| `SSO region [None]:` | `us-east-1` |
| `SSO registration scopes [sso:account:access]:` | **Just press Enter** (accept the default) |

**What happens next:**
1. Your web browser will open to a login page
2. Log in with the **Identity Center username and password** you created in Step 12 (NOT the root user credentials)
3. Click **Allow** or **Approve** when asked to authorize the request
4. Go back to your terminal — it will continue automatically

**The terminal will show your AWS account.** Press Enter to select it.

**Then it will ask you to choose a role.** Select **AdministratorAccess** and press Enter.

**Finally, it will ask for settings:**

| The terminal shows this prompt | What you type and press Enter |
|-------------------------------|-------------------------------|
| `CLI default client Region [None]:` | `us-east-1` |
| `CLI default output format [None]:` | `json` |
| `CLI profile name [AdministratorAccess-XXXX]:` | **Press Enter** to accept the suggested name, or type a name you will remember |

> **📝 Write down your profile name:** ______________________________
>
> You will need this for every future lab. It looks something like `AdministratorAccess-123456789012`.


> **💡 This is a one-time setup.** `aws configure sso` writes an SSO session block, a profile pointing to it, and your default region and output format into `~/.aws/config`. It persists on disk — it does **not** disappear when you close the terminal, restart your machine, or open a new window. You only run this once per computer.

---

### Step 21: Log In via SSO

📋 Copy and paste this command, **replacing `<YOUR_PROFILE_NAME>`** with the profile name from Step 20:

```
aws sso login --profile <YOUR_PROFILE_NAME>
```

> **🔄 Example:** If your profile name is `AdministratorAccess-123456789012`:
> ```
> aws sso login --profile AdministratorAccess-123456789012
> ```

Your browser will open. Approve the login, then return to your terminal.

---

### Step 22: Verify Your Identity

📋 Copy and paste this command, **replacing `<YOUR_PROFILE_NAME>`**:

```
aws sts get-caller-identity --profile <YOUR_PROFILE_NAME>
```

**What does this do?** Asks AWS "who am I?" — it confirms which account and role you are connected to.

**✅ You should see something like:**

```json
{
    "UserId": "AROAEXAMPLEID:your-username",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/AWSReservedSSO_AdministratorAccess_.../your-username"
}
```

**How to read this:**
- `"Account"` — Your 12-digit AWS account number
- `"Arn"` — Shows `AdministratorAccess` confirming you have the right permissions
- `"UserId"` — Shows your Identity Center username

> **📝 Write down your Account ID:** ______________________________
>
> You will need this 12-digit number in future labs.

---

### Step 23: Set Your Default Profile

To avoid typing `--profile <YOUR_PROFILE_NAME>` on every command, set a default for your terminal session.

**Windows (PowerShell):**

📋 Copy and paste, **replacing `<YOUR_PROFILE_NAME>`**:

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**macOS / Linux:**

```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**Test it.** 📋 Copy and paste:

```
aws sts get-caller-identity
```

**✅ You should see the same output as Step 22** — but without needing `--profile`.

>[!IMPORTANT]
> **⚠️ This default only lasts for the current terminal session.** If you close the terminal, you will need to:
> 1. Run `aws sso login --profile <YOUR_PROFILE_NAME>` to log in again
> 2. Run the profile command above again

---

## What You Just Did

You completed the full setup that a cloud engineer does when starting at a new company:

1. **Created an AWS account** — your own cloud environment
2. **Enabled IAM Identity Center** — the modern, secure identity management system
3. **Created a user, group, and permission set** — following the security best practice of never using the root account for daily work
4. **Installed the AWS CLI** — the command-line tool used by cloud engineers worldwide
5. **Configured SSO** — connecting the CLI to your account securely, with temporary credentials that expire (no passwords stored on your machine)

This is the foundation for every lab in this program. From now on, you will log in through the access portal (for the console) or via `aws sso login` (for the CLI).

---

## Key Information to Save

Keep these values somewhere safe — you will need them for every future lab:

| Value | Where You Got It | Your Value |
|-------|-----------------|------------|
| AWS access portal URL | Step 11 | __________________________ |
| Identity Center username | Step 12 | __________________________ |
| Identity Center password | Step 12 (email link) | __________________________ |
| AWS CLI profile name | Step 20 | __________________________ |
| AWS Account ID (12 digits) | Step 22 | __________________________ |

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| Account activation takes more than a few minutes | AWS is verifying your information | Wait up to 24 hours. Check your email for updates. You can try signing in periodically. |
| Cannot find IAM Identity Center in the console | You may be in the wrong region | Make sure the region (top-right of console) is set to **US East (N. Virginia)** or your preferred region. |
| "You need to create an AWS Organization" message | This is expected | Click the button to create the Organization. It is free. |
| Email invitation for Identity Center user not received | Check spam/junk folder | Wait 2–3 minutes, then check spam. If still nothing, go to Users in Identity Center, select the user, and click "Send email verification." |
| `'aws' is not recognized` or `command not found` | Terminal doesn't know where the CLI is | Close your terminal and open a brand new one. If still failing, restart your computer. |
| Browser does not open during `aws configure sso` | Browser launch failed | Look in your terminal for a URL — copy it and paste it into your browser manually. |
| `Error when retrieving token from sso: Token has expired` | Your SSO session expired (normal) | Run `aws sso login --profile <YOUR_PROFILE_NAME>` again. |
| Wrong account number in `get-caller-identity` output | You may have selected the wrong account during SSO setup | Run `aws configure sso` again and select the correct account. |
| No new-customer credits offered at signup | You used a card (or identity) AWS has seen before, so the account isn't eligible for the $100–$200 new-customer credits | This usually doesn't block you — this program forfeits those credits anyway when you enable AWS Organizations in Part 2. If you specifically want the credits for other learning, sign up with a different card, or log back into your original account (if it was closed, contact AWS Support to reopen it, which can take up to 24 hrs). |

---

## Cleanup

No AWS resources were created in this lab that incur charges. The account, Identity Center, Organization, user, group, and permission set are all free.

>[!CAUTION]
> **⚠️ Do NOT delete your account, Identity Center setup, or user** — you need all of these for every future lab.

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (if applicable — copy and paste it)
3. The **full error message** you received (copy and paste it)
4. Your **operating system** (Windows, Mac, or Linux)
