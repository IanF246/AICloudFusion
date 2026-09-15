# Lab 3A: IAM Least Privilege — Create a Read-Only User

**Session:** 3 — Cloud Security Fundamentals  
**Track:** Cloud Security & IR  
**Difficulty:** Beginner  
**Estimated Time:** 20–25 minutes  
**Target Cert:** AWS Security Specialty

---

## Overview

In this lab, you will:

1. **Create an S3 bucket** with a test file
2. **Create an IAM user** with no permissions
3. **Write a custom policy** that grants ONLY read access to one specific bucket
4. **Attach the policy** to the user
5. **Test the restricted credentials** — prove that reads work and writes are denied

By the end of this lab, you will understand the **principle of least privilege** — giving users only the minimum permissions they need to do their job, and nothing more. This is the single most important concept in cloud security.

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
| Amazon S3 | Cloud storage for files | $0.023 per GB/month |

**Estimated cost for this lab: $0.00** — the test bucket holds a single tiny file and is deleted in cleanup.

---

## Concepts

Before we start, here is what each concept means:

**IAM (Identity and Access Management)** is the AWS service that controls who can do what in your account. Every time someone (or something) interacts with AWS, IAM checks whether they have permission.

**IAM User** is an identity within your AWS account. Each user has their own credentials and permissions. In the real world, you would create IAM users for automated systems or legacy applications (humans should use Identity Center, which you set up in Lab 1A).

**IAM Policy** is a JSON document that defines permissions. It says "Allow (or Deny) these actions on these resources." Policies are the rules — users, groups, and roles are who the rules apply to.

**Least Privilege** means giving a user or system only the permissions it absolutely needs — nothing more. If a user only needs to read files, do not give them write access. If they only need access to one bucket, do not give them access to all buckets. This limits the damage if credentials are stolen.

**Inline Policy** is a policy that is embedded directly on a single user, group, or role. It is not reusable — it only applies to that one entity. For quick, one-off permissions, inline policies are simple and effective.

**Access Keys** are long-lived credentials (an Access Key ID + Secret Access Key) that allow programmatic access to AWS. They are like a username and password for the CLI. In production, you should avoid access keys and use roles or SSO instead — but for this lab, we use them to demonstrate how permissions work.

---

## ⚠️ Reminder: How to Read Commands in This Lab

Commands are inside gray code boxes. **📋 Copy and paste** them into your terminal.

**Placeholders** look like `<THIS>` — replace them (including the `<` and `>`) with your actual values.

Here are the placeholders you will use in this lab:

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name from Lab 1A | `AdministratorAccess-123456789012` |
| `<YOUR_BUCKET_NAME>` | A globally unique S3 bucket name you choose | `jane-doe-lab3a-readonly` |

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

---

### Step 2: Create Your Project Folder

**Step 2a: Create the folder**

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
mkdir ~\Desktop\workshop-lab-3a
cd ~\Desktop\workshop-lab-3a
```

**macOS / Linux:**

📋 Copy and paste:

```bash
mkdir ~/Desktop/workshop-lab-3a
cd ~/Desktop/workshop-lab-3a
```

> **What does this do?** Creates a new folder on your Desktop called `workshop-lab-3a` and moves your terminal into that folder.

**Step 2b: Verify you're in the right folder**

📋 Copy and paste:

```
pwd
```

**✅ You should see** a path ending in `workshop-lab-3a` (e.g., `C:\Users\YourName\Desktop\workshop-lab-3a` on Windows or `/Users/YourName/Desktop/workshop-lab-3a` on Mac).

> **💡 From now on, save ALL files you create in this lab to this folder.** When the lab says "save the file," save it here.

**Step 2c: Open the folder in VS Code**

📋 Copy and paste:

```
code .
```

> **What does this do?** This opens VS Code with `workshop-lab-3a` as its **file tree** on the left, so the policy file you create in Step 5 lands in the right place. (You set up the `code` command in Lab 1B — if you see `'code' is not recognized`, close and reopen your terminal, or revisit Lab 1B, Step 6.)

---

### Step 3: Create an S3 Bucket and Upload a Test File

First, create a bucket to use for testing permissions.

📋 Copy and paste this command, **replacing `<YOUR_BUCKET_NAME>`**:

```
aws s3 mb s3://<YOUR_BUCKET_NAME> --region us-east-1
```

> **🔄 Example:**
> ```
> aws s3 mb s3://jane-doe-lab3a-readonly --region us-east-1
> ```

**✅ You should see:**

```
make_bucket: <YOUR_BUCKET_NAME>
```

Now upload a test file to the bucket.

**Windows (PowerShell):**

📋 Copy and paste, **replacing `<YOUR_BUCKET_NAME>`**:

```powershell
"This is a secret document. Only authorized readers should see this." | Out-File -Encoding utf8 test-file.txt
aws s3 cp test-file.txt s3://<YOUR_BUCKET_NAME>/test-file.txt
```

**macOS / Linux:**

📋 Copy and paste, **replacing `<YOUR_BUCKET_NAME>`**:

```bash
echo "This is a secret document. Only authorized readers should see this." > test-file.txt
aws s3 cp test-file.txt s3://<YOUR_BUCKET_NAME>/test-file.txt
```

**✅ You should see:**

```
upload: ./test-file.txt to s3://<YOUR_BUCKET_NAME>/test-file.txt
```

---

### Step 4: Create an IAM User

Now create a user that will have restricted (read-only) access.

📋 Copy and paste:

```
aws iam create-user --user-name workshop-readonly-user
```

**What does this do?** Creates a new IAM user called `workshop-readonly-user`. Right now, this user has **zero permissions** — it cannot do anything in your account.

**✅ You should see** JSON output with the user details, including `"UserName": "workshop-readonly-user"`.

---

### Step 5: Write the Read-Only Policy

Now you will write a custom IAM policy that grants ONLY read access to your specific bucket. This is the **least privilege** principle in action — the user can read from this one bucket and nothing else.

**Step 5a:** In the VS Code file tree, click the **New File** icon and name the file `s3-readonly-policy.json`.

**Step 5b:** 📋 Copy and paste this entire block into it, **replacing `<YOUR_BUCKET_NAME>`** in BOTH places:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowListBucket",
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::<YOUR_BUCKET_NAME>"
        },
        {
            "Sid": "AllowGetObject",
            "Effect": "Allow",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::<YOUR_BUCKET_NAME>/*"
        }
    ]
}
```

> **🔄 Example:** If your bucket name is `jane-doe-lab3a-readonly`:
> ```json
> "Resource": "arn:aws:s3:::jane-doe-lab3a-readonly"
> ```
> and
> ```json
> "Resource": "arn:aws:s3:::jane-doe-lab3a-readonly/*"
> ```

**Step 5c:** **Save** the file (**Ctrl+S** / **Cmd+S**). You should see `s3-readonly-policy.json` appear in the file tree.

> **⚠️ Common mistake:** Make sure the name is exactly `s3-readonly-policy.json`, not `s3-readonly-policy.json.txt`. Naming it in the file tree with a `.json` ending sets the file type automatically.

> **What does this file do?** It defines a policy with two permissions:
> - **ListBucket** — allows the user to list the contents of the bucket (like `ls` in a folder)
> - **GetObject** — allows the user to download/read files from the bucket
>
> Notice what is **NOT** included: `PutObject` (upload), `DeleteObject` (delete), `DeleteBucket`. The user cannot write, modify, or delete anything.

**What does each field mean?**

| Field | Value | What It Means |
|-------|-------|---------------|
| `Version` | `2012-10-17` | The policy language version (always use this date) |
| `Sid` | `AllowListBucket` | A label for this statement (can be anything descriptive) |
| `Effect` | `Allow` | This statement grants permission |
| `Action` | `s3:ListBucket` | The specific AWS action being allowed |
| `Resource` | `arn:aws:s3:::bucket-name` | Which resource this applies to (the bucket itself for ListBucket, the objects inside for GetObject) |

> **💡 Why two Resource formats?** `arn:aws:s3:::bucket-name` (no `/*`) refers to the bucket itself — needed for listing. `arn:aws:s3:::bucket-name/*` (with `/*`) refers to the objects inside the bucket — needed for reading files.

---

### Step 6: Attach the Policy to the User

Now apply the policy to the user using an inline policy.

📋 Copy and paste:

```
aws iam put-user-policy --user-name workshop-readonly-user --policy-name S3ReadOnlyPolicy --policy-document file://s3-readonly-policy.json
```

**What does this do?**
- `put-user-policy` — attaches an inline policy directly to the user
- `--policy-name S3ReadOnlyPolicy` — a name for this policy (for your reference)
- `--policy-document file://s3-readonly-policy.json` — reads the policy from the file you created

**✅ No output means success.**

---

### Step 7: Create Access Keys for the User

To test the user's permissions, you need credentials for that user. Access keys are like a username/password pair for the CLI.

📋 Copy and paste:

```
aws iam create-access-key --user-name workshop-readonly-user
```

**✅ You should see** JSON output containing:

```json
{
    "AccessKey": {
        "UserName": "workshop-readonly-user",
        "AccessKeyId": "AKIA...",
        "SecretAccessKey": "wJalr...",
        "Status": "Active"
    }
}
```

> **📝 Write down BOTH values immediately:**
>
> - **AccessKeyId:** ______________________________
> - **SecretAccessKey:** ______________________________
>
> ⚠️ The SecretAccessKey is shown **only once**. If you lose it, you must delete the key and create a new one.

---

### Step 8: Test with Restricted Credentials

Now you will switch to the restricted user's credentials and prove that the policy works — reads succeed and writes are denied.

**Step 8a: Set the restricted user's credentials**

You need to temporarily override your admin credentials with the restricted user's credentials.

**Windows (PowerShell):**

📋 Copy and paste, **replacing the placeholders** with the values from Step 7:

```powershell
$env:AWS_ACCESS_KEY_ID="<ACCESS_KEY_ID_FROM_STEP_7>"
$env:AWS_SECRET_ACCESS_KEY="<SECRET_ACCESS_KEY_FROM_STEP_7>"
$env:AWS_DEFAULT_REGION="us-east-1"
Remove-Item Env:\AWS_PROFILE
```

**macOS / Linux:**

📋 Copy and paste, **replacing the placeholders** with the values from Step 7:

```bash
export AWS_ACCESS_KEY_ID="<ACCESS_KEY_ID_FROM_STEP_7>"
export AWS_SECRET_ACCESS_KEY="<SECRET_ACCESS_KEY_FROM_STEP_7>"
export AWS_DEFAULT_REGION="us-east-1"
unset AWS_PROFILE
```

> **What does this do?** Sets environment variables that the AWS CLI uses for authentication. Removing `AWS_PROFILE` ensures the CLI uses these access keys instead of your admin SSO session.

**Step 8b: Verify you are now the restricted user**

📋 Copy and paste:

```
aws sts get-caller-identity
```

**✅ You should see** output showing `workshop-readonly-user` — NOT your admin role.

---

**Step 8c: Test LIST — should WORK ✅**

📋 Copy and paste, **replacing `<YOUR_BUCKET_NAME>`**:

```
aws s3 ls s3://<YOUR_BUCKET_NAME>/
```

**✅ You should see** `test-file.txt` listed. The read-only user CAN list the bucket contents.

---

**Step 8d: Test READ — should WORK ✅**

📋 Copy and paste, **replacing `<YOUR_BUCKET_NAME>`**:

```
aws s3 cp s3://<YOUR_BUCKET_NAME>/test-file.txt -
```

**✅ You should see** the contents of the file printed: `This is a secret document. Only authorized readers should see this.`

> **💡 What does the `-` at the end mean?** It tells the CLI to print the file contents to the screen instead of saving it to a file on your computer.

---

**Step 8e: Test WRITE — should be DENIED ❌**

**Windows (PowerShell):**

📋 Copy and paste, **replacing `<YOUR_BUCKET_NAME>`**:

```powershell
"hack attempt" | Out-File test-write.txt
aws s3 cp test-write.txt s3://<YOUR_BUCKET_NAME>/unauthorized.txt 2>&1
```

**macOS / Linux:**

📋 Copy and paste, **replacing `<YOUR_BUCKET_NAME>`**:

```bash
echo "hack attempt" > test-write.txt
aws s3 cp test-write.txt s3://<YOUR_BUCKET_NAME>/unauthorized.txt 2>&1
```

**✅ You should see an error:** `upload failed` with `An error occurred (AccessDenied)`. The read-only user CANNOT write to the bucket. **Least privilege is working!**

> **💡 What does `2>&1` do?** It redirects error messages to the screen so you can see the "Access Denied" error clearly.

---

**Step 8f: Test DELETE — should be DENIED ❌**

📋 Copy and paste, **replacing `<YOUR_BUCKET_NAME>`**:

```
aws s3 rm s3://<YOUR_BUCKET_NAME>/test-file.txt 2>&1
```

**✅ You should see an error:** `delete failed` with `An error occurred (AccessDenied)`. The read-only user CANNOT delete files. **The data is protected.**

---

### Step 9: Switch Back to Admin Credentials

**⚠️ Important:** You must clear the restricted credentials and restore your admin profile before cleanup.

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
Remove-Item Env:\AWS_ACCESS_KEY_ID
Remove-Item Env:\AWS_SECRET_ACCESS_KEY
Remove-Item Env:\AWS_DEFAULT_REGION
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**macOS / Linux:**

📋 Copy and paste:

```bash
unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
unset AWS_DEFAULT_REGION
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**Verify you are back to admin:**

📋 Copy and paste:

```
aws sts get-caller-identity
```

**✅ You should see** your admin role (with `AdministratorAccess` in the ARN), NOT `workshop-readonly-user`.

---

### Step 10: Console Checkpoint

Let's verify what you built in the AWS Console:

1. Go to [https://console.aws.amazon.com/](https://console.aws.amazon.com/)
2. Search for **IAM** in the top search bar and click it
3. Click **Users** in the left sidebar
4. Click on **workshop-readonly-user**
5. Click the **Permissions** tab
6. You should see **S3ReadOnlyPolicy** listed as an inline policy
7. Click the policy name to expand it — you should see the JSON you wrote

**✅ Checkpoint:** You can see the user and its attached policy in the console. This is exactly how a security auditor would review permissions in a real AWS account.

---

## What You Just Did

You implemented the **principle of least privilege** — the foundation of cloud security:

1. **Created an IAM user** with zero permissions by default
2. **Wrote a custom policy** that grants only specific read actions on one specific bucket
3. **Proved the policy works** — reads succeed, writes and deletes are denied
4. **Understood the difference** between bucket-level permissions (`ListBucket`) and object-level permissions (`GetObject`)

In the real world, this is how security teams protect sensitive data. A data analyst might get read-only access to a reporting bucket. A backup system might get write-only access to an archive bucket. No one gets more access than they need.

---

## Cert Prep Callout

**Target Certification:** AWS Security Specialty (SCS-C02)

The Security Specialty exam heavily tests IAM policies. You need to understand:
- The difference between `Allow` and `Deny` effects
- How `Resource` ARNs work (bucket vs. object level)
- Inline policies vs. managed policies
- The principle of least privilege and how to implement it

**Sample question type:** "A company wants to allow a third-party application to read objects from a specific S3 bucket but prevent it from listing or deleting objects. Which policy statement achieves this?"

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `An error occurred (EntityAlreadyExists)` when creating the user | The user already exists from a previous attempt | Delete it first: `aws iam delete-user --user-name workshop-readonly-user` then try again |
| `An error occurred (MalformedPolicyDocument)` when attaching the policy | Your JSON file has a syntax error | Open `s3-readonly-policy.json` and check for missing commas, brackets, or quotes. Make sure you replaced `<YOUR_BUCKET_NAME>` in both Resource lines. |
| Write test shows "upload failed" but no "AccessDenied" message | The error went to stderr | Make sure you included `2>&1` at the end of the command |
| `get-caller-identity` still shows admin after setting env vars | `AWS_PROFILE` is overriding the access keys | Make sure you removed `AWS_PROFILE` (Step 8a) |
| `get-caller-identity` still shows readonly user after cleanup | Env vars were not cleared | Run the commands in Step 9 again. Close and reopen your terminal if needed. |
| `The specified bucket does not exist` during testing | Wrong bucket name in the command | Double-check your bucket name matches exactly |

---

## Cleanup

**⚠️ Important:** Always clean up resources after completing a lab. Follow these steps in order.

### Step 1: Delete the Access Key

First, you need the Access Key ID from Step 7.

📋 Copy and paste, **replacing `<ACCESS_KEY_ID_FROM_STEP_7>`**:

```
aws iam delete-access-key --user-name workshop-readonly-user --access-key-id <ACCESS_KEY_ID_FROM_STEP_7>
```

**✅ No output means success.**

### Step 2: Delete the User Policy

📋 Copy and paste:

```
aws iam delete-user-policy --user-name workshop-readonly-user --policy-name S3ReadOnlyPolicy
```

**✅ No output means success.**

### Step 3: Delete the User

📋 Copy and paste:

```
aws iam delete-user --user-name workshop-readonly-user
```

**✅ No output means success.**

### Step 4: Empty and Delete the S3 Bucket

📋 Copy and paste, **replacing `<YOUR_BUCKET_NAME>`**:

```
aws s3 rm s3://<YOUR_BUCKET_NAME> --recursive
```

```
aws s3 rb s3://<YOUR_BUCKET_NAME>
```

**✅ You should see** `remove_bucket: <YOUR_BUCKET_NAME>`.

### Step 5: Delete Local Files

> **⚠️ Close VS Code first.** If VS Code still has the `workshop-lab-3a` folder open, the delete will fail — especially on Windows. Choose **File → Close Folder** or quit VS Code before running the commands below.

Remove the project folder:

**Windows (PowerShell):**

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-3a
```

**macOS / Linux:**

```bash
cd ~/Desktop
rm -rf ~/Desktop/workshop-lab-3a
```

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** you received (copy and paste it)
4. Your **operating system** (Windows, Mac, or Linux)
