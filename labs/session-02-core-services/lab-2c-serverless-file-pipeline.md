# Lab 2C: Build a Serverless File Processing App with a Web Frontend

**Session:** 2 — Core AWS Services  
**Track:** Cloud Basics  
**Difficulty:** Advanced  
**Estimated Time:** 50–60 minutes

---

## Overview

In this lab, you will build a **complete serverless application** — a web page where you can upload text, have it automatically processed by AWS Lambda, and see the result displayed on the page. This ties together everything from Sessions 1 and 2: S3 static website hosting, Lambda functions, IAM roles, S3 event triggers, and presigned URLs.

**What you will build:**
- A new **"File Processor" page added to your existing Lab 1C website** (with a file upload form), plus a nav link between the two pages
- A **presign Lambda function** that generates secure upload URLs for the browser
- A **Lambda Function URL** so the web page can call the presign function
- An **input S3 bucket** where uploaded files land
- A **processing Lambda function** that triggers automatically on upload, converts text to uppercase, and counts words
- An **output S3 bucket** where processed results are stored and publicly readable
- The web page **displays the processed result** and provides a download link

This is a real-world architecture pattern — serverless web applications that process user uploads without any servers running 24/7.

---

## Prerequisites

- ✅ Completed **Lab 1A** (AWS account, CLI configured)
- ✅ Completed **Lab 1C** and your website is **still live** (you did not run 1C's cleanup) — this lab adds a page to that existing site. Have your **Lab 1C website bucket name** handy.
- ✅ AWS CLI authenticated
- ✅ VS Code installed (from Lab 1B)

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|----------------|
| AWS Lambda | Serverless compute | 1 million requests/month always free |
| Amazon S3 | Cloud storage + website hosting | $0.023 per GB/month |
| Lambda Function URLs | Public HTTP endpoint for Lambda | No extra charge — billed as normal Lambda invocations |
| IAM | Access management | Always Free |

**Estimated cost for this lab: ~$0.02** (effectively $0.00 if you complete the cleanup steps promptly)

---

## Concepts

**Lambda Function URL** is a simple way to give your Lambda function a public HTTP endpoint (a URL that anyone can call). Instead of setting up API Gateway (which is more complex), you create a Function URL and the browser can call it directly. We use this so the web page can ask Lambda for a presigned upload URL.

**Presigned URL** is a temporary URL that grants permission to upload a file to S3. Normally, S3 buckets are private. A presigned URL says "anyone with this link can upload one file, but only for the next 5 minutes." This is how the web page uploads files without needing AWS credentials in the browser.

**CORS (Cross-Origin Resource Sharing)** is a browser security feature. When your web page (hosted on one domain) tries to call a Lambda URL or fetch from S3 (different domains), the browser blocks it unless those services explicitly allow it. We configure CORS on our buckets and Lambda to permit these cross-origin requests.

---

## ⚠️ Placeholders in This Lab

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account number | `123456789012` |
| `<INPUT_BUCKET>` | Unique name for input bucket | `jane-doe-pipeline-input` |
| `<OUTPUT_BUCKET>` | Unique name for output bucket | `jane-doe-pipeline-output` |
| `<YOUR_WEBSITE_BUCKET>` | Your **existing** website bucket from Lab 1C (reused here) | `jane-doe-cloud-workshop-site` |
| `<FUNCTION_URL>` | The Lambda Function URL (you'll get this in Step 7) | `https://abc123.lambda-url.us-east-1.on.aws/` |

---

## Lab Steps

### Step 1: Set Your AWS Profile

**Windows (PowerShell):**

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**macOS / Linux:**

```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

We set the AWS profile here so you don't have to add `--profile <YOUR_PROFILE_NAME>` to every command below. Without it, the commands would fail with a credentials error.

### Step 1b: Connect your CLI to AWS via SSO

Check if your session is still active:

> **Note:** This uses the profile name you created in Lab 1A.

```
aws sts get-caller-identity
```

If you are starting a new terminal instance or the session has expired you will get this error:

> aws: [ERROR]: Error when retrieving token from sso: Token has expired and refresh failed

This is ok. Follow the next step.

Connect to AWS via SSO:
**Windows (PowerShell) & Linux/macOS:**

```
aws sso login
```
A new browser window should open — either authorizing the access automatically (*if you are already logged into the console*), or asking you to log into the console first before authorizing the connection.

---

### Step 2: Create Your Project Folder

Before creating any files, let's set up a dedicated folder for this lab. This keeps your files organized and ensures your terminal can find them.

**Step 2a: Create the folder**

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
mkdir ~\Desktop\workshop-lab-2c
cd ~\Desktop\workshop-lab-2c
```

**macOS / Linux:**

📋 Copy and paste:

```bash
mkdir ~/Desktop/workshop-lab-2c
cd ~/Desktop/workshop-lab-2c
```

> **What does this do?** Creates a new folder on your Desktop called `workshop-lab-2c` and moves your terminal into that folder. All commands you run and files you create will be in this folder.

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

**✅ You should see** a path ending in `workshop-lab-2c` (e.g., `C:\Users\YourName\Desktop\workshop-lab-2c` on Windows or `/Users/YourName/Desktop/workshop-lab-2c` on Mac).

> **💡 From now on, save ALL files you create in this lab to this folder.** When the lab says "save the file," save it here. This is where your terminal is looking for files.

**Step 2c: Open the folder in VS Code**

📋 Copy and paste:

```
code .
```

> **What does this do?** This opens VS Code with `workshop-lab-2c` as its **file tree** on the left. This lab creates several files (JSON configs, two Python functions, and an HTML page) — opening the folder now means every file you create lands in the right place. (You set up the `code` command in Lab 1B — if you see `'code' is not recognized`, close and reopen your terminal, or revisit Lab 1B, Step 6.)

---

### Step 3: Create Two S3 Buckets

You need two **new** buckets: one for file uploads (input) and one for processed results (output). You will **reuse your existing website bucket from Lab 1C** for the web page, so you do not create a website bucket here.

📋 Copy and paste, **replacing the bucket names** with your own unique names:

```
aws s3 mb s3://<INPUT_BUCKET> --region us-east-1
aws s3 mb s3://<OUTPUT_BUCKET> --region us-east-1
```

---

### Step 4: Create the Lambda IAM Role

Both Lambda functions need a role that gives them permission to access S3 and write logs. You will create this role now.

**Step 4a: Create the trust policy file**

In the VS Code file tree, click the **New File** icon and name the file `lambda-trust-policy.json`. 📋 Copy and paste this into it:

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

> **What is this file?** It tells AWS "Lambda functions are allowed to use this role." Without it, Lambda can't assume the role's permissions.

---

**Step 4b: Create the role and attach permissions**

📋 Copy and paste these three commands into your terminal, **one at a time**, pressing Enter after each:

```
aws iam create-role --role-name workshop-pipeline-role --assume-role-policy-document file://lambda-trust-policy.json
```

> **What does this do?** Creates a new IAM role called `workshop-pipeline-role`.

```
aws iam attach-role-policy --role-name workshop-pipeline-role --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
```

> **What does this do?** Gives the role permission to read from and write to any S3 bucket.

```
aws iam attach-role-policy --role-name workshop-pipeline-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

> **What does this do?** Gives the role permission to write logs to CloudWatch (so you can see what happened when the function runs).

**⏳ Wait 10 seconds** before continuing to the next step. IAM changes take a moment to propagate across AWS.

---

### Step 5: Create the Presign Lambda Function

Now you will create the first Lambda function. This function's job is simple: when the web page asks for an upload link, this function generates a temporary URL that allows the upload.

**You do NOT need to know Python to complete this step.** You are just copying pre-written code into a file. The code is provided for you — just follow the steps exactly.

---

**Step 5a: Create a new file in VS Code**

In the VS Code file tree, click the **New File** icon and name the file `presign_function.py`.

> **💡 Important:** The file needs to be inside your `workshop-lab-2c` folder — it will be, since you opened that folder with `code .` in Step 2c. If unsure, check that the file tree header at the top-left shows `workshop-lab-2c`.

---

**Step 5b: Copy and paste the code below into the file**

📋 Copy this **entire block** of code and paste it into your new file. Do not change anything — copy it exactly as shown:

```python
import json
import boto3
import os

s3_client = boto3.client('s3')

def lambda_handler(event, context):
    params = event.get('queryStringParameters', {}) or {}
    filename = params.get('filename', 'upload.txt')
    
    input_bucket = os.environ['INPUT_BUCKET']
    
    presigned_url = s3_client.generate_presigned_url(
        'put_object',
        Params={
            'Bucket': input_bucket,
            'Key': filename,
            'ContentType': 'text/plain'
        },
        ExpiresIn=300
    )
    
    return {
        'statusCode': 200,
        'body': json.dumps({'uploadUrl': presigned_url, 'filename': filename})
    }
```

---

**Step 5c: Save the file**

Press **Ctrl+S** (Windows) or **Cmd+S** (Mac) to save. Because you named it in the file tree, it saves straight into `workshop-lab-2c`.

> **⚠️ Common mistakes:**
> - The name must be exactly `presign_function.py` — Lambda uses it to find the code. Naming it in the file tree with the `.py` ending sets the file type automatically (no stray `.txt`).
> - Make sure there are no extra spaces or blank lines at the very beginning of the file.

---

**Step 5d: What does this code do? (optional reading)**

You don't need to understand the code to complete this lab, but here's what each part does if you're curious:

| Line(s) | What It Does |
|---------|-------------|
| `import json, boto3, os` | Loads libraries that the code needs (JSON handling, AWS SDK, environment variables) |
| `s3_client = boto3.client('s3')` | Creates a connection to the S3 service |
| `def lambda_handler(event, context):` | This is the function that Lambda calls when triggered. Think of it as the "main" function. |
| `params = event.get(...)` | Gets the filename from the web page's request |
| `input_bucket = os.environ['INPUT_BUCKET']` | Reads the input bucket name from a configuration variable (you'll set this when deploying) |
| `s3_client.generate_presigned_url(...)` | Generates a temporary URL that allows uploading one file to S3 |
| `ExpiresIn=300` | The URL is valid for 300 seconds (5 minutes) |
| `return {...}` | Sends the URL back to the web page |

---

**Step 5e: Package the code into a zip file**

Lambda requires code to be uploaded as a `.zip` file. You will zip the file you just created.

**macOS / Linux:**

📋 Copy and paste this command into your terminal:

```bash
zip presign.zip presign_function.py
```

**Windows (PowerShell):**

📋 Copy and paste this command into your terminal:

```powershell
Compress-Archive -Path presign_function.py -DestinationPath presign.zip -Force
```

> **What does this do?** Creates a file called `presign.zip` that contains your `presign_function.py`. This is the package you will upload to Lambda.

**✅ You should now have two files in your folder:** `presign_function.py` and `presign.zip`

---

**Step 5f: Deploy the function to AWS Lambda**

Now you will upload the zip file to Lambda and create the function.

📋 Copy and paste this command, **replacing `<YOUR_ACCOUNT_ID>` and `<INPUT_BUCKET>`** with your actual values:

**Windows (PowerShell):**

```powershell
aws lambda create-function --function-name workshop-presign --runtime python3.12 --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-pipeline-role" --handler presign_function.lambda_handler --zip-file fileb://presign.zip --environment "Variables={INPUT_BUCKET=<INPUT_BUCKET>}" --region us-east-1
```

**macOS / Linux:**

```bash
aws lambda create-function \
    --function-name workshop-presign \
    --runtime python3.12 \
    --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-pipeline-role" \
    --handler presign_function.lambda_handler \
    --zip-file fileb://presign.zip \
    --environment "Variables={INPUT_BUCKET=<INPUT_BUCKET>}" \
    --region us-east-1
```

> **What does each part mean?**
> - `--function-name workshop-presign` — names the function so you can find it later
> - `--runtime python3.12` — tells Lambda to use Python to run the code
> - `--role "arn:aws:iam::..."` — which permissions the function has (the role you created in Step 4)
> - `--handler presign_function.lambda_handler` — tells Lambda "look in the file `presign_function.py` and call the function named `lambda_handler`"
> - `--zip-file fileb://presign.zip` — the code package to upload (`fileb://` means "binary file from my computer")
> - `--environment "Variables={INPUT_BUCKET=...}"` — passes your input bucket name to the function as a configuration variable (this is how the code knows which bucket to generate URLs for)

**✅ You should see** a JSON response with details about the function, including `"State": "Active"`.

**✅ Checkpoint — In the AWS Console:**
1. Search for **Lambda** → click it
2. Make sure you're in the **us-east-1** region
3. You should see **workshop-presign** in the functions list

---

### Step 6: Create the Processing Lambda Function

Now you will create the second Lambda function. This one triggers automatically when a file is uploaded to the input bucket. It reads the file, converts the text to uppercase, counts the words, and saves the result to the output bucket.

Again — **you do NOT need to know Python.** Just copy the code exactly as shown.

---

**Step 6a: Create a new file in VS Code**

In the VS Code file tree, create another **New File** and name it `process_function.py`. This is a separate file from the one you created in Step 5.

---

**Step 6b: Copy and paste the code below into the file**

📋 Copy this **entire block** of code and paste it into your new file:

```python
import json
import boto3
import os

s3_client = boto3.client('s3')

def lambda_handler(event, context):
    source_bucket = event['Records'][0]['s3']['bucket']['name']
    source_key = event['Records'][0]['s3']['object']['key']
    
    if not source_key.endswith('.txt'):
        return {'statusCode': 200, 'body': 'Skipped'}
    
    response = s3_client.get_object(Bucket=source_bucket, Key=source_key)
    original_text = response['Body'].read().decode('utf-8')
    
    processed_text = original_text.upper()
    word_count = len(original_text.split())
    
    output_content = f"=== PROCESSED FILE ===\n"
    output_content += f"Original file: {source_key}\n"
    output_content += f"Word count: {word_count}\n"
    output_content += f"=== CONTENT (UPPERCASE) ===\n"
    output_content += processed_text
    
    output_bucket = os.environ['OUTPUT_BUCKET']
    output_key = f"processed-{source_key}"
    
    s3_client.put_object(
        Bucket=output_bucket,
        Key=output_key,
        Body=output_content.encode('utf-8'),
        ContentType='text/plain'
    )
    
    return {'statusCode': 200, 'body': json.dumps(f'Processed {source_key}')}
```

---

**Step 6c: Save the file**

Press **Ctrl+S** / **Cmd+S** to save. It saves into `workshop-lab-2c` alongside your other files.

> **⚠️ Same warnings as before:** The name must be exactly `process_function.py`, and there should be no extra spaces or blank lines at the beginning.

---

**Step 6d: What does this code do? (optional reading)**

| Line(s) | What It Does |
|---------|-------------|
| `source_bucket = event['Records'][0]...` | When S3 triggers this function, it sends information about which file was uploaded. This line reads the bucket name and filename. |
| `if not source_key.endswith('.txt'):` | Only process `.txt` files — skip everything else |
| `s3_client.get_object(...)` | Downloads the uploaded file from S3 |
| `original_text.upper()` | Converts all text to UPPERCASE |
| `len(original_text.split())` | Counts the number of words |
| `output_content = ...` | Builds the processed output with a header showing the filename and word count |
| `output_bucket = os.environ['OUTPUT_BUCKET']` | Reads the output bucket name from configuration |
| `s3_client.put_object(...)` | Uploads the processed result to the output bucket |
| `output_key = f"processed-{source_key}"` | Names the output file `processed-` plus the original filename |

**In plain English:** "When a .txt file is uploaded to the input bucket, download it, convert it to uppercase, count the words, add a header, and save the result to the output bucket."

---

**Step 6e: Package the code into a zip file**

**macOS / Linux:**

📋 Copy and paste:

```bash
zip process.zip process_function.py
```

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
Compress-Archive -Path process_function.py -DestinationPath process.zip -Force
```

**✅ You should now have these files in your folder:** `presign_function.py`, `presign.zip`, `process_function.py`, `process.zip`

---

**Step 6f: Deploy the function to AWS Lambda**

📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>` and `<OUTPUT_BUCKET>`**:

**Windows (PowerShell):**

```powershell
aws lambda create-function --function-name workshop-processor --runtime python3.12 --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-pipeline-role" --handler process_function.lambda_handler --zip-file fileb://process.zip --environment "Variables={OUTPUT_BUCKET=<OUTPUT_BUCKET>}" --region us-east-1
```

**macOS / Linux:**

```bash
aws lambda create-function \
    --function-name workshop-processor \
    --runtime python3.12 \
    --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-pipeline-role" \
    --handler process_function.lambda_handler \
    --zip-file fileb://process.zip \
    --environment "Variables={OUTPUT_BUCKET=<OUTPUT_BUCKET>}" \
    --region us-east-1
```

> **What's different from the presign function?**
> - Different function name (`workshop-processor`)
> - Different handler (`process_function.lambda_handler` — points to the `process_function.py` file)
> - Different environment variable (`OUTPUT_BUCKET` instead of `INPUT_BUCKET`)

**✅ You should see** a JSON response with `"State": "Active"`.

**✅ Checkpoint — In the AWS Console:**
1. Go to **Lambda** → **Functions**
2. You should now see **two functions**: `workshop-presign` and `workshop-processor`

---

### Step 7: Create a Function URL for the Presign Lambda and InvokeFunction Permission

This gives the presign function a public URL that the web page can call.

📋 Copy and paste:

```
aws lambda create-function-url-config --function-name workshop-presign --auth-type NONE --cors "AllowOrigins=*,AllowMethods=GET,AllowHeaders=*" --region us-east-1
```

> **What does `--auth-type NONE` mean?** It makes the URL publicly accessible without authentication. This is fine for our use case — the presign function only generates upload URLs, it doesn't expose any sensitive data.

> **What does `--cors` do?** It tells the browser "yes, web pages from other domains are allowed to call this URL." Without this, the browser would block the request.

**✅ You should see** a JSON response with a `FunctionUrl` field. **Copy this URL.**

> **📝 Write down your Function URL:** ______________________________
>
> It looks like: `https://abc123xyz.lambda-url.us-east-1.on.aws/`

**Grant public access:**

```
aws lambda add-permission --function-name workshop-presign --statement-id FunctionURLAllowPublicAccess --action lambda:InvokeFunctionUrl --principal "*" --function-url-auth-type NONE --region us-east-1
```

```
aws lambda add-permission --function-name workshop-presign --statement-id AllowPublicInvoke --action lambda:InvokeFunction --principal "*" --region us-east-1
```

>[!CAUTION]
>**lambda:InvokeFunction is the general "run this function" permission — Lambda Function URLs require both: InvokeFunctionUrl to authorize the URL endpoint itself, and InvokeFunction to actually execute the function behind it. Without both granted to '*', public access is denied.**

> **⏳ Wait 1–2 minutes** for the permission to propagate before testing.

---

### Step 8: Configure the S3 Event Trigger

Now you will connect the input bucket to the processing Lambda so it fires automatically whenever a `.txt` file is uploaded.

**Step 8a: Grant S3 permission to invoke the Lambda**

📋 Copy and paste, **replacing `<INPUT_BUCKET>`** with your input bucket name:

```
aws lambda add-permission --function-name workshop-processor --statement-id s3-trigger --action lambda:InvokeFunction --principal s3.amazonaws.com --source-arn arn:aws:s3:::<INPUT_BUCKET> --region us-east-1
```

> **What does this do?** Tells Lambda "allow S3 to trigger this function, but only from this specific bucket."

---

**Step 8b: Create the notification configuration file**

In the VS Code file tree, create a **New File** named `s3-notification.json`. 📋 Copy and paste this into it, **replacing `<YOUR_ACCOUNT_ID>`** with your 12-digit account number:

```json
{
    "LambdaFunctionConfigurations": [
        {
            "LambdaFunctionArn": "arn:aws:lambda:us-east-1:<YOUR_ACCOUNT_ID>:function:workshop-processor",
            "Events": ["s3:ObjectCreated:*"],
            "Filter": {
                "Key": {
                    "FilterRules": [
                        {
                            "Name": "suffix",
                            "Value": ".txt"
                        }
                    ]
                }
            }
        }
    ]
}
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

> **What does this file do?** It tells S3: "Whenever a new file ending in `.txt` is uploaded to this bucket, send a notification to the `workshop-processor` Lambda function."

> **⚠️ Make sure** you replaced `<YOUR_ACCOUNT_ID>` with your actual 12-digit number inside the file before saving.

---

**Step 8c: Apply the notification configuration**

📋 Copy and paste, **replacing `<INPUT_BUCKET>`**:

```
aws s3api put-bucket-notification-configuration --bucket <INPUT_BUCKET> --notification-configuration file://s3-notification.json --region us-east-1
```

**✅ No output means success.** Your pipeline is now wired up — uploading a `.txt` file to the input bucket will automatically trigger the processing Lambda.

---

### Step 9: Configure CORS on the S3 Buckets

The browser has a security feature called CORS that blocks web pages from talking to other domains unless those domains explicitly allow it. Since your web page (on the website bucket) needs to upload to the input bucket and read from the output bucket, you need to tell those buckets "yes, allow requests from web pages."

**Step 9a: Create the CORS configuration file**

In the VS Code file tree, create a **New File** named `cors.json`. 📋 Copy and paste this into it (no placeholders to replace here):

```json
{
    "CORSRules": [
        {
            "AllowedHeaders": ["*"],
            "AllowedMethods": ["PUT", "GET"],
            "AllowedOrigins": ["*"],
            "ExposeHeaders": []
        }
    ]
}
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

> **What does this file do?** It says: "Allow any web page to upload files (PUT) and download files (GET) from this bucket."

---

**Step 9b: Apply CORS to both the input and output buckets**

📋 Copy and paste these two commands, **replacing the bucket names**:

```
aws s3api put-bucket-cors --bucket <INPUT_BUCKET> --cors-configuration file://cors.json --region us-east-1
```

```
aws s3api put-bucket-cors --bucket <OUTPUT_BUCKET> --cors-configuration file://cors.json --region us-east-1
```

**✅ No output means success** for both commands.

---

### Step 10: Make the Output Bucket Publicly Readable

The web page needs to be able to fetch the processed results from the output bucket and display them. For this, the output bucket needs to be publicly readable (just like the website bucket in Lab 1C).

**Step 10a: Disable Block Public Access on the output bucket**

📋 Copy and paste, **replacing `<OUTPUT_BUCKET>`**:

```
aws s3api put-public-access-block --bucket <OUTPUT_BUCKET> --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
```

---

**Step 10b: Create the public read policy file**

In the VS Code file tree, create a **New File** named `output-policy.json`. 📋 Copy and paste this into it, **replacing `<OUTPUT_BUCKET>`** with your actual output bucket name:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::<OUTPUT_BUCKET>/*"
        }
    ]
}
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

> **⚠️ Make sure** you replaced `<OUTPUT_BUCKET>` with your actual bucket name inside the file before saving.

---

**Step 10c: Apply the policy**

📋 Copy and paste, **replacing `<OUTPUT_BUCKET>`**:

```
aws s3api put-bucket-policy --bucket <OUTPUT_BUCKET> --policy file://output-policy.json
```

**✅ No output means success.**

---

### Step 11: Add the Processor Page to Your Website

Instead of creating a brand-new website, you will add this app as a **new page on the website you built in Lab 1C**. That bucket already has static website hosting and public read access enabled (you set that up in 1C), so there is nothing to configure here — you simply publish one more HTML file to it and link to it from your homepage.

> **🧭 Did you complete the 1D sidequest (CloudFront + HTTPS)?** Your site is served through CloudFront and the bucket is private. Everything below still works — you upload the new page exactly the same way — but you will need one extra step (a **cache invalidation**) after each upload, and you will use your CloudFront URL to view it. Look for the **"1D path"** notes in Steps 11b, 11c, and 12.

**Step 11a: Create the processor page**

In the VS Code file tree, create a **New File** named `processor.html`. 📋 Copy and paste this entire block into it, **replacing `<FUNCTION_URL>` and `<OUTPUT_BUCKET>`** with your actual values:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Cloud File Processor</title>
    <style>
        body { font-family: Arial, sans-serif; max-width: 700px; margin: 50px auto; padding: 20px; background: #f0f4f8; }
        h1 { color: #232f3e; }
        .card { background: white; border-radius: 8px; padding: 30px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); margin: 20px 0; }
        textarea { width: 100%; height: 150px; padding: 12px; border: 2px solid #ddd; border-radius: 6px; font-size: 16px; resize: vertical; }
        button { background: #ff9900; color: white; border: none; padding: 12px 24px; font-size: 16px; border-radius: 6px; cursor: pointer; margin-top: 10px; }
        button:hover { background: #ec7211; }
        button:disabled { background: #ccc; cursor: not-allowed; }
        .status { margin-top: 15px; padding: 12px; border-radius: 6px; display: none; }
        .status.success { display: block; background: #d4edda; color: #155724; border: 1px solid #c3e6cb; }
        .status.error { display: block; background: #f8d7da; color: #721c24; border: 1px solid #f5c6cb; }
        .status.loading { display: block; background: #fff3cd; color: #856404; border: 1px solid #ffeeba; }
        .result { margin-top: 15px; padding: 15px; background: #1a1a2e; color: #00ff88; border-radius: 6px; white-space: pre-wrap; font-family: monospace; font-size: 14px; }
        .badge { background: #232f3e; color: white; padding: 4px 10px; border-radius: 4px; font-size: 12px; }
        a.download-link { display: inline-block; margin-top: 10px; background: #232f3e; color: white; padding: 10px 20px; border-radius: 6px; text-decoration: none; }
        a.download-link:hover { background: #37475a; }
    </style>
</head>
<body>
    <p><a href="index.html" style="color:#232f3e;">&#8592; Back to Home</a></p>
    <h1>&#9729; Cloud File Processor</h1>
    <p>Upload a text file and watch it get processed automatically by AWS Lambda.</p>
    <p><span class="badge">Serverless</span> <span class="badge">S3</span> <span class="badge">Lambda</span></p>

    <div class="card">
        <h2>Upload a Text File</h2>
        <p>Type or paste some text below, then click Upload. The serverless pipeline will process it automatically.</p>
        <textarea id="fileContent" placeholder="Type your text here..."></textarea>
        <br>
        <label for="fileName">File name: </label>
        <input type="text" id="fileName" value="my-upload.txt" style="padding: 8px; border: 2px solid #ddd; border-radius: 4px;">
        <br>
        <button id="uploadBtn" onclick="uploadFile()">Upload & Process</button>
        <div id="status" class="status"></div>
    </div>

    <div id="resultSection" class="card" style="display:none;">
        <h2>&#9989; Processed Result</h2>
        <p>Lambda processed your file. Here is the output:</p>
        <div id="result" class="result"></div>
        <a id="downloadLink" class="download-link" href="#" target="_blank">&#128229; View Processed File</a>
    </div>

    <div class="card">
        <h2>How It Works</h2>
        <ol>
            <li>You type text and click Upload</li>
            <li>The web page asks Lambda for a <strong>presigned URL</strong> (a temporary upload link)</li>
            <li>The file is uploaded directly to the <strong>S3 input bucket</strong></li>
            <li>S3 automatically triggers a <strong>Lambda function</strong></li>
            <li>Lambda processes the file (converts to uppercase, counts words)</li>
            <li>The result is saved to the <strong>S3 output bucket</strong></li>
            <li>The processed file is displayed above and available for viewing</li>
        </ol>
        <p><strong>All of this happens without any servers running 24/7.</strong> Lambda only runs when a file is uploaded.</p>
    </div>

    <script>
        const PRESIGN_URL = '<FUNCTION_URL>';
        const OUTPUT_BUCKET = '<OUTPUT_BUCKET>';
        const OUTPUT_REGION = 'us-east-1';

        async function uploadFile() {
            const content = document.getElementById('fileContent').value;
            const filename = document.getElementById('fileName').value;
            const statusEl = document.getElementById('status');
            const btn = document.getElementById('uploadBtn');

            if (!content.trim()) {
                statusEl.className = 'status error';
                statusEl.textContent = 'Please enter some text to upload.';
                return;
            }

            btn.disabled = true;
            statusEl.className = 'status loading';
            statusEl.textContent = 'Getting upload URL from Lambda...';

            try {
                const presignResponse = await fetch(PRESIGN_URL + '?filename=' + encodeURIComponent(filename));
                const presignData = await presignResponse.json();

                statusEl.textContent = 'Uploading file to S3...';

                const uploadResponse = await fetch(presignData.uploadUrl, {
                    method: 'PUT',
                    body: content,
                    headers: { 'Content-Type': 'text/plain' }
                });

                if (!uploadResponse.ok) throw new Error('Upload failed: ' + uploadResponse.status);

                statusEl.className = 'status loading';
                statusEl.textContent = 'File uploaded! Waiting for Lambda to process...';

                const outputUrl = 'https://' + OUTPUT_BUCKET + '.s3.' + OUTPUT_REGION + '.amazonaws.com/processed-' + filename;

                await new Promise(resolve => setTimeout(resolve, 5000));

                const resultResponse = await fetch(outputUrl);
                if (resultResponse.ok) {
                    const resultText = await resultResponse.text();
                    document.getElementById('result').textContent = resultText;
                    document.getElementById('downloadLink').href = outputUrl;
                    document.getElementById('resultSection').style.display = 'block';
                    statusEl.className = 'status success';
                    statusEl.textContent = 'File "' + filename + '" processed successfully! See the result below.';
                } else {
                    document.getElementById('downloadLink').href = outputUrl;
                    document.getElementById('result').textContent = 'Processing may still be in progress. Click the link below to check.';
                    document.getElementById('resultSection').style.display = 'block';
                    statusEl.className = 'status success';
                    statusEl.textContent = 'File uploaded! Processing may take a few more seconds. Try the link below.';
                }
            } catch (error) {
                statusEl.className = 'status error';
                statusEl.textContent = 'Error: ' + error.message;
            }

            btn.disabled = false;
        }
    </script>
</body>
</html>
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

> **What does this file do?** This is your complete web application — it provides a text box for input, calls your Lambda function to get a presigned upload URL, uploads the file to S3, waits for processing, and displays the result. All in one HTML file.

> **⚠️ Make sure** you replaced both `<FUNCTION_URL>` (your Lambda Function URL from Step 7) and `<OUTPUT_BUCKET>` (your output bucket name) inside the file before saving. Look for them near the bottom of the file in the `<script>` section.

---

**Step 11b: Upload the processor page**

📋 Copy and paste, **replacing `<YOUR_WEBSITE_BUCKET>`** with your Lab 1C website bucket name:

```
aws s3 cp processor.html s3://<YOUR_WEBSITE_BUCKET>/processor.html --content-type "text/html" --region us-east-1
```

> **🧭 1D path (CloudFront):** after uploading, refresh CloudFront's cache so it serves the new page (`<YOUR_DIST_ID>` is the Distribution ID from Lab 1D):
> ```
> aws cloudfront create-invalidation --distribution-id <YOUR_DIST_ID> --paths "/*"
> ```

**Step 11c: Add a link to it from your homepage**

Now add a "File Processor" link to your existing homepage so visitors can find the new page. You will download your current `index.html`, add one link, and re-upload it.

📋 Download your current homepage into this lab folder, **replacing `<YOUR_WEBSITE_BUCKET>`**:

```
aws s3 cp s3://<YOUR_WEBSITE_BUCKET>/index.html index.html
```

Open `index.html` in VS Code (it will appear in the file tree). Find the closing `</body>` tag near the bottom and add this line **just above it**:

```html
<p><a href="processor.html">Go to the File Processor &#8594;</a></p>
```

**Save** the file (**Ctrl+S** / **Cmd+S**), then re-upload it, **replacing `<YOUR_WEBSITE_BUCKET>`**:

```
aws s3 cp index.html s3://<YOUR_WEBSITE_BUCKET>/index.html --content-type "text/html" --region us-east-1
```

> **🧭 1D path (CloudFront):** run the cache invalidation again after re-uploading `index.html`:
> ```
> aws cloudfront create-invalidation --distribution-id <YOUR_DIST_ID> --paths "/*"
> ```

---

### Step 12: Test Your Application!

Open your **File Processor** page:

```
http://<YOUR_WEBSITE_BUCKET>.s3-website-us-east-1.amazonaws.com/processor.html
```

> **🧭 1D path (CloudFront):** use your CloudFront address instead — `https://<YOUR_DIST_DOMAIN>/processor.html` (e.g., `https://d123abc.cloudfront.net/processor.html`).

1. Open this URL in your browser (or click the new **File Processor** link on your homepage)
2. Type some text in the text area
3. Click **Upload & Process**
4. Wait 5 seconds — the processed result should appear on the page
5. Click **View Processed File** to see the raw output in a new tab

**✅ Checkpoint:** You should see your text converted to UPPERCASE with a word count header.

> **🎉 You just built a complete serverless web application!** A website that accepts user input, processes it with Lambda, and displays the result — all without a single server running 24/7.

---

## What You Just Did

You built a **full-stack serverless application** using 5 AWS services working together:

1. **S3 (your Lab 1C website bucket)** — now hosts the processor page alongside your homepage
2. **Lambda (Presign Function)** — generates secure upload URLs on demand
3. **S3 (Input Bucket)** — receives uploaded files
4. **Lambda (Processor Function)** — automatically processes files when they arrive
5. **S3 (Output Bucket)** — stores and serves processed results

This architecture is used in production by companies for image processing, document conversion, data pipelines, and more. The key insight: **no servers are running when no one is using the app.** You only pay for actual usage.

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

This lab covers event-driven architectures, Lambda triggers, S3 event notifications, and presigned URLs — all heavily tested on the SAA exam.

**Sample question type:** "A company needs to automatically process images when users upload them through a web application. The solution should scale automatically and minimize operational overhead. Which architecture should they use?"

---

## Troubleshooting

| Issue | How to Fix It |
|-------|---------------|
| "Failed to fetch" when clicking Upload | The Function URL permission may not have propagated yet. Wait 2 minutes and try again. |
| CORS error in browser console | Make sure you ran the CORS configuration on both the input and output buckets (Step 9). |
| "Access-Control-Allow-Origin contains multiple values" | Your Lambda code is returning CORS headers AND the Function URL is adding them. Remove CORS headers from the Lambda code — the Function URL config handles it. |
| File uploads but no result appears | Check Lambda logs: Console → Lambda → workshop-processor → Monitor → View CloudWatch Logs. The processing Lambda may have errored. |
| 403 on the output file | Make sure you completed Step 10 (public access block disabled + bucket policy applied on the output bucket). |

---

## Cleanup

>[!IMPORTANT]
>**⚠️** Follow these steps to remove the file-processor app.

>[!NOTE]
> **💰 What if you left it running?** The whole pipeline sits within the always-free tier — Lambda covers 1 million requests/month free, and the near-empty input/output buckets cost only fractions of a cent. Realistically, leaving it in place would cost about **$0.00–$0.01/month**. Even so, it's good practice to remove what you no longer need, so we clean it up below. **We keep your Lab 1C website in place** (it's your site) and only remove the processor's backend and the extra page.

### Step 1: Remove S3 Notification

📋 Replace `<INPUT_BUCKET>`:

**Windows:**
```powershell
aws s3api put-bucket-notification-configuration --bucket <INPUT_BUCKET> --notification-configuration "{}" --region us-east-1
```

**macOS / Linux:**
```bash
aws s3api put-bucket-notification-configuration --bucket <INPUT_BUCKET> --notification-configuration '{}' --region us-east-1
```

### Step 2: Delete Lambda Functions

```
aws lambda delete-function --function-name workshop-presign --region us-east-1
aws lambda delete-function --function-name workshop-processor --region us-east-1
```

### Step 3: Empty and Delete the Input and Output Buckets

Delete only the two buckets you created for this lab. **Do not delete your Lab 1C website bucket** — you are keeping the site.

📋 Replace the bucket names:

```
aws s3 rm s3://<INPUT_BUCKET> --recursive
aws s3 rm s3://<OUTPUT_BUCKET> --recursive
aws s3 rb s3://<INPUT_BUCKET>
aws s3 rb s3://<OUTPUT_BUCKET>
```

### Step 3b: Remove the Processor Page from Your Website

Because the backend is now gone, remove the processor page and the homepage link so your site has no broken buttons.

📋 Delete the processor page, **replacing `<YOUR_WEBSITE_BUCKET>`**:

```
aws s3 rm s3://<YOUR_WEBSITE_BUCKET>/processor.html
```

Then re-open your local `index.html` in VS Code, delete the "Go to the File Processor" link line you added in Step 11c, **Save**, and re-upload it:

```
aws s3 cp index.html s3://<YOUR_WEBSITE_BUCKET>/index.html --content-type "text/html" --region us-east-1
```

> **🧭 1D path (CloudFront):** run a cache invalidation after these changes so CloudFront stops serving the removed page: `aws cloudfront create-invalidation --distribution-id <YOUR_DIST_ID> --paths "/*"`.

### Step 4: Delete the IAM Role

```
aws iam detach-role-policy --role-name workshop-pipeline-role --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
aws iam detach-role-policy --role-name workshop-pipeline-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name workshop-pipeline-role
```

### Step 5: Delete Local Files

> **⚠️ Close VS Code first.** If VS Code still has the `workshop-lab-2c` folder open, the delete will fail — especially on Windows. Choose **File → Close Folder** or quit VS Code before running the commands below. The commands also move you to your home directory (`cd ~`) first so the terminal isn't sitting inside the folder it's deleting.

Remove the project folder you created for this lab:

**macOS / Linux:**

```bash
cd ~
rm -rf ~/Desktop/workshop-lab-2c
```

**Windows (PowerShell):**

```powershell
cd ~
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-2c
```

### Step 6: Verify Cleanup

**✅ Checkpoint:**
1. **Lambda** → Functions → both functions are gone
2. **S3** → the input and output buckets are gone; **your Lab 1C website bucket remains** (with `processor.html` removed)
3. **IAM** → Roles → `workshop-pipeline-role` is gone

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **command** you ran
2. The **full error message**
3. Which **step number** you are on
