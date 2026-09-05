# Lab 1C: Host a Static Website on Amazon S3

**Session:** 1 — Cloud Concepts & AWS Global Infrastructure  
**Track:** Cloud Basics  
**Difficulty:** Advanced  
**Estimated Time:** 30–40 minutes

---

## Overview

In this lab, you will deploy a **real, live website on the internet** using Amazon S3. By the end, you will have a public URL that anyone in the world can visit to see your web page.

This is one of the most common tasks for cloud engineers — hosting static content (marketing pages, documentation sites, company landing pages) on S3 is fast, inexpensive, and scales automatically to handle any amount of traffic.

**What you will build:**
- An S3 bucket (cloud storage) configured as a website
- A custom homepage (`index.html`) and error page (`error.html`)
- A public URL where your website is live on the internet

---

## Prerequisites

- ✅ Completed **Lab 1A** (AWS CLI installed and configured)
- ✅ AWS CLI authenticated — run `aws sts get-caller-identity` and confirm it returns your account info
- ✅ **VS Code** installed and the `code` command working (you set this up in Lab 1B) — any text editor will do, but these labs assume VS Code

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|----------------|
| Amazon S3 | Cloud storage for files | $0.023 per GB/month, $0.0005 per 1,000 PUT/GET requests |

**Estimated cost for this lab: ~$0.02** (effectively $0.00 if you complete the cleanup steps promptly)

---

## Concepts

Before we start, here is what each service and concept does:

**Amazon S3 (Simple Storage Service)** is cloud storage. Think of it as a hard drive in the cloud — you create **"buckets"** (like folders) and upload files to them. S3 is one of the oldest and most widely used AWS services. Companies use it to store everything from website files to database backups to machine learning datasets.

**Static website hosting** is a feature built into S3. When you turn it on, S3 serves your HTML, CSS, and JavaScript files as a website — just like a web server would. The difference is that you do not need to set up or manage any servers. S3 handles everything for you.

**Bucket policy** is a set of rules (written in JSON format) that controls who can access the files in your bucket. For a public website, you need a policy that says "anyone on the internet can read these files." You will learn much more about policies in Session 3 (Cloud Security).

---

## ⚠️ Reminder: How to Read Commands in This Lab

Commands are inside gray code boxes. **📋 Copy and paste** them into your terminal.

**Placeholders** look like `<THIS>` — replace them (including the `<` and `>`) with your actual values.

Here are the placeholders you will use in this lab:

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name from Lab 1A | `AdministratorAccess-123456789012` |
| `<YOUR_UNIQUE_BUCKET_NAME>` | A globally unique name for your S3 bucket (see Step 3 for naming rules) | `jane-doe-cloud-workshop-site` |

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

Before creating any files, let's set up a dedicated folder for this lab. This keeps your files organized and ensures your terminal can find them.

**Step 2a: Create the folder**

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
mkdir ~\Desktop\workshop-lab-1c
cd ~\Desktop\workshop-lab-1c
```

**macOS / Linux:**

📋 Copy and paste:

```bash
mkdir ~/Desktop/workshop-lab-1c
cd ~/Desktop/workshop-lab-1c
```

> **What does this do?** Creates a new folder on your Desktop called `workshop-lab-1c` and moves your terminal into that folder. All commands you run and files you create will be in this folder.

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

**✅ You should see** a path ending in `workshop-lab-1c` (e.g., `C:\Users\YourName\Desktop\workshop-lab-1c` on Windows or `/Users/YourName/Desktop/workshop-lab-1c` on Mac).

> **💡 From now on, save ALL files you create in this lab to this folder.** When the lab says "save the file," save it here. This is where your terminal is looking for files.

**Step 2c: Open the folder in VS Code**

📋 Copy and paste:

```
code .
```

> **What does this do?** This opens VS Code with `workshop-lab-1c` as its **file tree** on the left, so every file you create in the next steps lands in the right place. (You set up the `code` command in Lab 1B — if you see `'code' is not recognized`, close and reopen your terminal, or revisit Lab 1B, Step 6.)

---

### Step 3: Create an S3 Bucket

An S3 bucket is where your website files will live. Bucket names must be **globally unique** — no two AWS accounts in the world can have the same bucket name. Use your name or initials to make it unique.

**Naming rules:**
- 3–63 characters long
- Only lowercase letters, numbers, and hyphens (`-`)
- No spaces, no uppercase letters, no underscores
- Must start with a letter or number

**Good examples:** `jane-doe-cloud-workshop-site`, `jd-workshop-2026`, `my-first-aws-website-42`

📋 Copy and paste this command, **replacing `<YOUR_UNIQUE_BUCKET_NAME>`** with a name you choose:

```
aws s3 mb s3://<YOUR_UNIQUE_BUCKET_NAME> --region us-east-1
```

> **🔄 Example:**
> ```
> aws s3 mb s3://jane-doe-cloud-workshop-site --region us-east-1
> ```

**What does this do?**
- `aws s3 mb` — "mb" stands for "make bucket" — this creates a new S3 bucket
- `s3://` — this prefix tells the CLI you are referring to an S3 location
- `--region us-east-1` — creates the bucket in the US East (N. Virginia) region

**✅ You should see:**

```
make_bucket: <YOUR_UNIQUE_BUCKET_NAME>
```

> **🔧 If you see `BucketAlreadyExists`:** Someone else in the world already has a bucket with that name. Choose a different name — try adding numbers or your initials.

**✅ Checkpoint — Verify in the AWS Console:**
1. Go to [https://console.aws.amazon.com/](https://console.aws.amazon.com/)
2. Search for **S3** in the top search bar and click it
3. You should see your bucket in the list
4. Notice the **Region** column shows **US East (N. Virginia)**

> **📝 Write down your bucket name:** ______________________________ *(you will need it for every step below)*

---

### Step 4: Create Your Website Files

Now you will create the HTML files for your website. You need two files:
- `index.html` — your homepage (what visitors see when they go to your site)
- `error.html` — an error page (what visitors see if they go to a page that does not exist)

**First, create a subfolder to hold your website files.**

📋 Copy and paste this command:

```
mkdir my-website
```

**What does this do?** This creates a new folder called `my-website` inside your `workshop-lab-1c` project folder where you will save your HTML files.

---

**Now create the homepage.** In the VS Code file tree, open the `my-website` folder, click the **New File** icon, and name the file `index.html`. 📋 Copy and paste this entire block into it:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My First Cloud Website</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            padding: 50px;
            background: #f0f4f8;
        }
        h1 { color: #232f3e; }
        p { color: #545b64; font-size: 18px; }
        .badge {
            background: #ff9900;
            color: white;
            padding: 10px 20px;
            border-radius: 5px;
            display: inline-block;
            margin-top: 20px;
        }
    </style>
</head>
<body>
    <h1>Hello from AWS S3!</h1>
    <p>You just deployed your first static website on the cloud.</p>
    <div class="badge">Cloud Engineer in Training</div>
</body>
</html>
```

**Save** the file with **Ctrl+S** (Windows) or **Cmd+S** (Mac). Because you created it inside `my-website`, it saves there automatically — you should see `index.html` under `my-website` in the file tree.

> **💡 Feel free to customize!** Change the text to say your name, change the colors, add more content. This is your website — make it yours.

> **💡 What is HTML?** HTML (HyperText Markup Language) is the language used to create web pages. The `<h1>` tag creates a heading, `<p>` creates a paragraph, and `<style>` controls colors and layout. You do not need to understand HTML to complete this lab — just copy and paste.

---

**Now create the error page.** In the file tree, create another **New File** inside `my-website` and name it `error.html`. 📋 Copy and paste this:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Page Not Found</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            padding: 50px;
            background: #f0f4f8;
        }
        h1 { color: #d13212; }
        p { color: #545b64; font-size: 18px; }
    </style>
</head>
<body>
    <h1>404 - Page Not Found</h1>
    <p>The page you are looking for does not exist.</p>
</body>
</html>
```

**Save** the file (**Ctrl+S** / **Cmd+S**). You should see `error.html` appear next to `index.html` in the file tree.

**Your folder should now look like this:**

```
my-website/
├── index.html
└── error.html
```

---

### Step 5: Upload Your Files to S3

Now you will upload both HTML files from your computer to your S3 bucket in the cloud.

📋 Copy and paste these two commands **one at a time**, **replacing `<YOUR_UNIQUE_BUCKET_NAME>`** in each:

```
aws s3 cp my-website/index.html s3://<YOUR_UNIQUE_BUCKET_NAME>/index.html --content-type "text/html"
```

```
aws s3 cp my-website/error.html s3://<YOUR_UNIQUE_BUCKET_NAME>/error.html --content-type "text/html"
```

> **🔄 Example:**
> ```
> aws s3 cp my-website/index.html s3://jane-doe-cloud-workshop-site/index.html --content-type "text/html"
> ```

**What does this do?**
- `aws s3 cp` — "cp" stands for "copy" — this copies a file from your computer to S3
- `my-website/index.html` — the file on your computer (the source)
- `s3://<YOUR_UNIQUE_BUCKET_NAME>/index.html` — where to put it in S3 (the destination)
- `--content-type "text/html"` — tells S3 this is an HTML file, so browsers will display it as a web page instead of trying to download it

**✅ You should see** (for each file):

```
upload: my-website/index.html to s3://<YOUR_UNIQUE_BUCKET_NAME>/index.html
```

**✅ Checkpoint — Verify in the AWS Console:**
1. Go to **S3** in the AWS Console
2. Click on your bucket name
3. You should see both **index.html** and **error.html** listed as objects (files)

---

### Step 6: Enable Static Website Hosting

Right now, your bucket is just storing files. You need to tell S3 to serve those files as a website.

📋 Copy and paste this command, **replacing `<YOUR_UNIQUE_BUCKET_NAME>`**:

```
aws s3 website s3://<YOUR_UNIQUE_BUCKET_NAME> --index-document index.html --error-document error.html
```

**What does this do?**
- `aws s3 website` — enables the static website hosting feature on your bucket
- `--index-document index.html` — when someone visits the root URL of your site (like `yoursite.com`), S3 will serve `index.html`
- `--error-document error.html` — when someone visits a page that does not exist (like `yoursite.com/blah`), S3 will serve `error.html` instead of a generic error

**✅ No output means success.** The terminal will just show a new prompt.

**✅ Checkpoint — Verify in the AWS Console:**
1. Go to **S3** → click your bucket → click the **Properties** tab
2. Scroll all the way down to the **Static website hosting** section
3. It should say **Enabled**
4. You should see your index document (`index.html`) and error document (`error.html`) listed
5. **Important:** You will also see a **Bucket website endpoint** URL — this is your website's address! It looks like:
   ```
   http://<YOUR_UNIQUE_BUCKET_NAME>.s3-website-us-east-1.amazonaws.com
   ```
   📝 **Copy this URL and save it** — you will use it in Step 9.

> **⚠️ If you try to visit the URL now, you will get a "403 Forbidden" error.** That is expected — the bucket is still private. We need to make it public in the next two steps.

---

### Step 7: Disable Block Public Access

By default, S3 **blocks all public access** to protect your data. This is a safety feature — you would not want someone accidentally making sensitive files public. But since we are intentionally creating a public website, we need to turn off these protections.

📋 Copy and paste this command, **replacing `<YOUR_UNIQUE_BUCKET_NAME>`**:

**macOS / Linux:**

```bash
aws s3api put-public-access-block \
    --bucket <YOUR_UNIQUE_BUCKET_NAME> \
    --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
```

**Windows (PowerShell):**

```powershell
aws s3api put-public-access-block --bucket <YOUR_UNIQUE_BUCKET_NAME> --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
```

**What does this do?** This disables the four "Block Public Access" settings on your bucket, allowing you to add a public access policy in the next step.

**✅ No output means success.**

> **⚠️ Security note:** In a real production environment, you would typically use **CloudFront** (a content delivery network) in front of S3 instead of making the bucket directly public. This is more secure and faster. We are using direct public access here because it is simpler for learning. You will learn about more secure patterns in later sessions.

---

### Step 8: Add a Bucket Policy for Public Read Access

Now you need to tell S3 **who** is allowed to access your files. You will create a **bucket policy** — a JSON file that defines the access rules.

1. In the VS Code file tree, at the **top level** of `workshop-lab-1c` (not inside `my-website`), click the **New File** icon and name it `bucket-policy.json`.

2. 📋 Copy and paste this entire block into the file, **replacing `<YOUR_UNIQUE_BUCKET_NAME>`** on the `Resource` line:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::<YOUR_UNIQUE_BUCKET_NAME>/*"
        }
    ]
}
```

> **🔄 Example:** If your bucket name is `jane-doe-cloud-workshop-site`, the Resource line would be:
> ```
> "Resource": "arn:aws:s3:::jane-doe-cloud-workshop-site/*"
> ```

3. **Save** the file (**Ctrl+S** / **Cmd+S**). It should sit at the top level of `workshop-lab-1c`, alongside the `my-website` folder — **not** inside `my-website`.

> **⚠️ Common mistake:** Make sure the name is exactly `bucket-policy.json`, not `bucket-policy.json.txt`. Naming it in the file tree with a `.json` ending sets the file type automatically.

> **What does this file do?** It defines a bucket policy that allows anyone on the internet to view the files in your S3 bucket — which is what you want for a public website.

**What does each line mean?**

| Field | Value | What It Means (in plain English) |
|-------|-------|----------------------------------|
| `Version` | `2012-10-17` | The version of the policy language (always use this date — it is not today's date, it is the policy format version) |
| `Sid` | `PublicReadGetObject` | A label for this rule (can be anything descriptive) |
| `Effect` | `Allow` | This rule **allows** access (as opposed to denying it) |
| `Principal` | `*` | **Who** is allowed — the `*` means "everyone" (the entire internet) |
| `Action` | `s3:GetObject` | **What** they can do — `GetObject` means "download/view files" |
| `Resource` | `arn:aws:s3:::your-bucket/*` | **Which files** — the `/*` means "all files in this bucket" |

> **In plain English:** "Allow anyone on the internet to view any file in this bucket."

> **💡 This is an IAM policy** — you will learn much more about policies, permissions, and security in Session 3 (Cloud Security Fundamentals).

**Now apply the policy to your bucket.** 📋 Copy and paste this command, **replacing `<YOUR_UNIQUE_BUCKET_NAME>`**:

```
aws s3api put-bucket-policy --bucket <YOUR_UNIQUE_BUCKET_NAME> --policy file://bucket-policy.json
```

**What does this do?**
- `aws s3api put-bucket-policy` — applies a policy to your bucket
- `--policy file://bucket-policy.json` — reads the policy from the file you just created

**✅ No output means success.**

---

### Step 9: Visit Your Website! 🎉

Your website is now live on the internet. The URL follows this pattern:

```
http://<YOUR_UNIQUE_BUCKET_NAME>.s3-website-us-east-1.amazonaws.com
```

> **🔄 Example:**
> ```
> http://jane-doe-cloud-workshop-site.s3-website-us-east-1.amazonaws.com
> ```

📋 **Copy your URL, paste it into your browser's address bar, and press Enter.**

**✅ You should see** your "Hello from AWS S3!" page with the orange "Cloud Engineer in Training" badge.

**Now test the error page.** Add `/nonexistent` to the end of your URL:

```
http://<YOUR_UNIQUE_BUCKET_NAME>.s3-website-us-east-1.amazonaws.com/nonexistent
```

**✅ You should see** your custom "404 - Page Not Found" error page.

**✅ Checkpoint:** Take a screenshot of your live website in the browser. **You just deployed a website to the cloud!** 🎉

> **💡 Share it!** This URL is public — you can send it to friends, family, or classmates and they can see your website too.

---

## What You Just Did

You deployed a **static website on Amazon S3** — the same service that hosts static content for companies around the world. Here is what you built:

1. **An S3 bucket** — cloud storage for your website files
2. **HTML files** — a homepage and an error page
3. **Static website hosting** — turned your storage bucket into a web server
4. **A bucket policy** — set permissions so anyone can view your site
5. **A live website** — accessible to anyone on the internet via a public URL

This is a real-world task that cloud engineers, DevOps engineers, and web developers perform regularly. Many production websites (documentation sites, marketing pages, single-page applications) are hosted exactly this way on S3.

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

The SAA exam frequently tests S3 concepts: storage classes, bucket policies, static website hosting, and access control. Understanding how S3 bucket policies work and how to configure public access is directly relevant to exam questions.

**Sample question type:** "A company needs to host a static website with low latency and high availability. Which AWS service combination should they use?"

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `BucketAlreadyExists` when creating the bucket | Someone else in the world already has a bucket with that name | Choose a different name — add numbers or your initials |
| `403 Forbidden` when visiting the website URL | The bucket is still private — the policy was not applied correctly | Make sure you completed **both** Step 7 (disable Block Public Access) **and** Step 8 (add bucket policy). Open `bucket-policy.json` and verify the bucket name in the `Resource` line is correct. |
| Website shows XML instead of HTML | You are using the wrong URL format | Make sure you are using the **website endpoint** (contains `s3-website-us-east-1`), **not** the S3 API endpoint (contains `s3.amazonaws.com`). The correct URL is in Step 9. |
| `NoSuchBucket` error | The bucket name in your command does not match your actual bucket | Double-check your bucket name — it must match exactly (case-sensitive, no extra spaces). |
| Files uploaded but website shows old content | Your browser may be caching the old version | Press **Ctrl+Shift+R** (Windows) or **Cmd+Shift+R** (Mac) to hard-refresh the page. |

---

## Cleanup

**⚠️ Important:** Always clean up resources after completing a lab so they do not incur charges. Follow these steps in order.

### Step 1: Empty the Bucket

You must remove all files before you can delete a bucket. S3 will not let you delete a bucket that still has files in it.

📋 Copy and paste this command, **replacing `<YOUR_UNIQUE_BUCKET_NAME>`**:

```
aws s3 rm s3://<YOUR_UNIQUE_BUCKET_NAME> --recursive
```

**What does this do?**
- `aws s3 rm` — "rm" stands for "remove" — this deletes files from S3
- `--recursive` — delete all files in the bucket (not just one)

**✅ You should see:**

```
delete: s3://<YOUR_UNIQUE_BUCKET_NAME>/index.html
delete: s3://<YOUR_UNIQUE_BUCKET_NAME>/error.html
```

### Step 2: Delete the Bucket

📋 Copy and paste this command, **replacing `<YOUR_UNIQUE_BUCKET_NAME>`**:

```
aws s3 rb s3://<YOUR_UNIQUE_BUCKET_NAME>
```

**What does this do?**
- `aws s3 rb` — "rb" stands for "remove bucket" — this deletes the bucket itself

**✅ You should see:**

```
remove_bucket: <YOUR_UNIQUE_BUCKET_NAME>
```

### Step 3: Verify the Bucket Is Gone

📋 Copy and paste this command, **replacing `<YOUR_UNIQUE_BUCKET_NAME>`**:

```
aws s3 ls s3://<YOUR_UNIQUE_BUCKET_NAME>
```

**✅ You should see an error:** `The specified bucket does not exist` — this confirms the bucket has been deleted.

**✅ Checkpoint — Verify in the AWS Console:**
1. Go to **S3** in the AWS Console
2. Confirm your bucket is **no longer in the list**

### Step 4: Try Visiting Your Website URL Again

Go back to your browser and refresh the website URL from Step 9.

**✅ You should see an error page** (like "404 Not Found" or a generic S3 error) — this confirms your website is no longer live.

### Step 5: Delete Local Files

> **⚠️ Close VS Code first.** While VS Code has the `workshop-lab-1c` folder open, your operating system treats the folder as "in use" and the delete command below will fail — especially on Windows. Before you delete it, either choose **File → Close Folder** in VS Code or close VS Code entirely. The commands below also move you to your home directory (`cd ~`) first so the terminal isn't sitting inside the folder it's deleting.

Remove the project folder you created for this lab:

**macOS / Linux:**

```bash
cd ~
rm -rf ~/Desktop/workshop-lab-1c
```

**Windows (PowerShell):**

```powershell
cd ~
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-1c
```

> **🔧 If you still get "cannot remove ... it is being used by another process" (Windows):** VS Code or one of its terminals is still holding the folder open. Close VS Code completely, open a brand-new PowerShell window, and run the two commands above again.

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **command** you ran (copy and paste it)
2. The **full error message** you received (copy and paste it)
3. Which **step number** you are on
