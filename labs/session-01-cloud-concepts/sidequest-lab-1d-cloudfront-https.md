# Lab 1D: Side Quest — HTTPS for Your Static Website with CloudFront

**Session:** 1 — Cloud Concepts  
**Track:** Cloud Basics  
**Type:** ⚡ Side Quest — Optional bonus lab  
**Difficulty:** Intermediate  
**Estimated Time:** 30–40 minutes (plus ~15 min CloudFront deploy wait)  
**Prerequisite Lab:** Lab 1C — S3 Static Website  
**Target Cert:** AWS Solutions Architect Associate (SAA-C03)

---

## Why This Lab Exists

After completing Lab 1C, your static website is live — but if you look closely at the URL, it starts with `http://`. Modern browsers flag this as **"Not Secure"**, and for good reason: HTTP traffic can be read or modified by anyone between your visitor and the server.

This side quest fixes that. You will put **Amazon CloudFront** in front of your S3 bucket. CloudFront is AWS's global Content Delivery Network (CDN). It handles HTTPS for you automatically — no certificate to buy or manage. As a bonus, you will also make your S3 bucket **private**, so the only way anyone can reach your content is through CloudFront. The old HTTP S3 link stops working entirely.

---

## Overview

In this lab, you will:

1. **Create an Origin Access Control (OAC)** — the secure link between CloudFront and your private S3 bucket
2. **Deploy a CloudFront distribution** — the HTTPS-enabled front door to your website
3. **Make the S3 bucket private** — remove public access, update the bucket policy to CloudFront-only
4. **Test and verify** — confirm HTTPS works, the old HTTP S3 URL is blocked, and your error page still loads
5. **Understand custom domains** — learn what is needed to use your own URL instead of the CloudFront-provided one

---

## Prerequisites

- ✅ Completed **Lab 1C** (S3 Static Website) — your bucket must still exist with `index.html` and `error.html` uploaded
- ✅ AWS CLI installed and authenticated — run `aws sts get-caller-identity` and confirm it works
- ✅ Your S3 bucket name from Lab 1C — you will use it throughout this lab

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| Amazon CloudFront | Global CDN + HTTPS | 1 TB data transfer + 10M requests **always free** |
| Amazon S3 | Static file storage | Already within free tier from Lab 1C |

**Estimated cost for this lab: $0.00** — CloudFront's free tier is generous. A student portfolio site serves well under 1 TB per month.

---

## Concepts

Before we start, here is what each concept means:

**Amazon CloudFront** is AWS's Content Delivery Network (CDN). A CDN copies your content to dozens of **edge locations** around the world — data centres close to your visitors. When someone in London visits your site, they get the files from a London edge location, not from your S3 bucket in us-east-1. This makes the site faster AND lets CloudFront handle HTTPS on your behalf.

**HTTPS / TLS** — HTTPS is HTTP with encryption. The encryption is provided by TLS (Transport Layer Security). CloudFront automatically provisions a free TLS certificate from AWS Certificate Manager for your `*.cloudfront.net` domain. You do not need to buy or renew anything.

**Origin** is what CloudFront fetches content from. In this lab, your S3 bucket is the origin. CloudFront caches what it fetches so it does not have to go back to S3 for every visitor.

**Origin Access Control (OAC)** is a security feature that proves to S3 that a request is coming from your specific CloudFront distribution and not from someone else. Without OAC, you would need to keep the bucket public. With OAC, the bucket can be completely private and only CloudFront can read it.

> **💡 OAC vs. OAI:** You may see older tutorials using "Origin Access Identity (OAI)." OAI is the legacy method — AWS deprecated it. OAC is the current best practice and is what this lab uses. Expect the Security Specialty and Solutions Architect exams to test this distinction.

**Viewer Protocol Policy** controls what happens when a visitor uses HTTP instead of HTTPS:
- `redirect-to-https` — automatically redirects `http://` requests to `https://` (what you will use)
- `https-only` — rejects HTTP requests entirely (stricter, but breaks visitors who type the URL without `https://`)

**Price Class** controls which CloudFront edge locations serve your content. More edge locations = faster for global users = more expensive. `PriceClass_100` uses North America and Europe only — cheapest, and fine for most workshop sites.

---

## ⚠️ Reminder: How to Read Commands in This Lab

Commands are inside gray code boxes. **📋 Copy and paste** them into your terminal.

**Placeholders** look like `<THIS>` — replace them (including the `<` and `>`) with your actual values.

Here are the placeholders you will use in this lab:

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name from Lab 1A | `AdministratorAccess-123456789012` |
| `<YOUR_BUCKET_NAME>` | Your S3 bucket name from Lab 1C | `jane-doe-my-portfolio` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account ID | `123456789012` |
| `<YOUR_REGION>` | The AWS region your bucket is in | `us-east-1` |

As you run commands, you will collect these additional values:

| Value | Where You Get It | Write It Down |
|-------|-----------------|---------------|
| OAC ID | Step 4 output | ______________________________ |
| Distribution ID | Step 6 output | ______________________________ |
| Distribution Domain | Step 6 output | `dXXXXXX.cloudfront.net` ______________________________ |

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

**Verify authentication and get your Account ID:**

```
aws sts get-caller-identity
```

> **📝 Write down your Account ID** (the 12-digit number): ______________________________

If you see an error about an expired token, run `aws sso login --profile <YOUR_PROFILE_NAME>` first.

---

### Step 2: Create Your Project Folder

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
mkdir ~\Desktop\workshop-lab-1d
cd ~\Desktop\workshop-lab-1d
```

**macOS / Linux:**

```bash
mkdir ~/Desktop/workshop-lab-1d
cd ~/Desktop/workshop-lab-1d
```

**Now open the folder in VS Code.** 📋 Copy and paste:

```
code .
```

> This opens VS Code with `workshop-lab-1d` as its **file tree**, so the config files you create in the next steps land in the right place. (If you see `'code' is not recognized`, close and reopen your terminal, or revisit Lab 1B, Step 6.)

---

### Step 3: Confirm Your S3 Bucket Still Exists

📋 Copy and paste, **replacing `<YOUR_BUCKET_NAME>`**:

```
aws s3 ls s3://<YOUR_BUCKET_NAME>
```

**✅ You should see** `index.html` and `error.html` listed. If the bucket is empty or doesn't exist, go back and complete Lab 1C first.

---

### Step 4: Create an Origin Access Control (OAC)

OAC is the secure handshake that lets CloudFront prove to S3 that requests are coming from your distribution.

**Step 4a: Create the OAC config file**

In the VS Code file tree, click the **New File** icon and name the file `oac-config.json`. 📋 Copy and paste this into it:

```json
{
  "Name": "workshop-lab-1d-oac",
  "Description": "OAC for Lab 1D S3 static website",
  "SigningProtocol": "sigv4",
  "SigningBehavior": "always",
  "OriginAccessControlOriginType": "s3"
}
 
```
**Save** the file (**Ctrl+S** / **Cmd+S**). You should see `oac-config.json` in the file tree.

> **What does this config mean?**
> - `SigningProtocol: sigv4` — CloudFront will sign every request to S3 using AWS Signature Version 4. S3 verifies this signature to confirm the request came from CloudFront.
> - `SigningBehavior: always` — sign every request, no exceptions.
> - `OriginAccessControlOriginType: s3` — this OAC is for an S3 origin (as opposed to a custom HTTP origin).

**Step 4b: Create the OAC**

📋 Copy and paste:

```
aws cloudfront create-origin-access-control --origin-access-control-config file://oac-config.json
```

**✅ You should see** output like this:

```json
{
    "OriginAccessControl": {
        "Id": "EABCDEF1234567",
        "OriginAccessControlConfig": { ... }
    }
}
```

> **📝 Write down the `Id` value** (looks like `EABCDEF1234567`): ______________________________
>
> This is your **OAC ID**. You will use it in the next step.

---

### Step 5: Create the CloudFront Distribution Config

This is the full configuration for your CloudFront distribution. You will write it to a JSON file, then deploy it with one command.

In the VS Code file tree, create a **New File** named `distribution-config.json`. 📋 Copy and paste the entire block into it, **replacing placeholders**:
- `<YOUR_BUCKET_NAME>` — your S3 bucket name
- `<YOUR_REGION>` — the region your bucket is in (e.g., `us-east-1`)
- `<YOUR_OAC_ID>` — the OAC ID from Step 4b

```json
{
  "CallerReference": "workshop-lab-1d-dist",
  "DefaultRootObject": "index.html",
  "Comment": "Workshop Lab 1D: Static website with HTTPS",
  "Enabled": true,
  "HttpVersion": "http2",
  "PriceClass": "PriceClass_100",
  "Origins": {
    "Quantity": 1,
    "Items": [
      {
        "Id": "S3Origin",
        "DomainName": "<YOUR_BUCKET_NAME>.s3.<YOUR_REGION>.amazonaws.com",
        "OriginAccessControlId": "<YOUR_OAC_ID>",
        "S3OriginConfig": {
          "OriginAccessIdentity": ""
        }
      }
    ]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "S3Origin",
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "AllowedMethods": {
      "Quantity": 2,
      "Items": ["GET", "HEAD"],
      "CachedMethods": {
        "Quantity": 2,
        "Items": ["GET", "HEAD"]
      }
    },
    "Compress": true
  },
  "CustomErrorResponses": {
    "Quantity": 1,
    "Items": [
      {
        "ErrorCode": 403,
        "ResponsePagePath": "/error.html",
        "ResponseCode": "404",
        "ErrorCachingMinTTL": 10
      }
    ]
  }
}
```

**Save** the file (**Ctrl+S** / **Cmd+S**).

> **What do the key settings do?**
> - `DefaultRootObject: index.html` — when someone visits `https://dXXXX.cloudfront.net/` with no file specified, serve `index.html`
> - `ViewerProtocolPolicy: redirect-to-https` — anyone who visits the `http://` version is automatically sent to `https://`
> - `CachePolicyId: 658327ea...` — uses the AWS-managed "CachingOptimized" policy (no configuration needed)
> - `PriceClass_100` — uses only North America and Europe edge locations (cheapest option)
> - `Compress: true` — CloudFront automatically compresses files (gzip/brotli) before sending them, making pages load faster
> - `CustomErrorResponses` — when S3 returns a `403` for a missing file (because the bucket is private, S3 can't return `404`), CloudFront serves your `error.html` with a `404` status code instead

---

### Step 6: Deploy the CloudFront Distribution

📋 Copy and paste:

```
aws cloudfront create-distribution --distribution-config file://distribution-config.json
```

**✅ You should see** a large JSON block. Find and write down two values:

```json
{
    "Distribution": {
        "Id": "EXXXXXXXXXXXXXXXXX",
        "DomainName": "dXXXXXXXXXXXXX.cloudfront.net",
        "Status": "InProgress",
        ...
    }
}
```

> **📝 Write down the Distribution `Id`:** ______________________________
>
> **📝 Write down the `DomainName`** (e.g., `d1234abcdef.cloudfront.net`): ______________________________

---

### Step 7: ⏳ Wait for CloudFront to Deploy (~15 minutes)

CloudFront propagates your distribution to all edge locations around the world. This takes approximately 5–20 minutes. The status starts as `InProgress` and changes to `Deployed` when complete.

**Check status with this command** (📋 copy and paste, **replacing `<YOUR_DIST_ID>`**):

```
aws cloudfront get-distribution --id <YOUR_DIST_ID> --query 'Distribution.Status' --output text
```

Run this every 2–3 minutes until you see `Deployed`.

---

> ## 💡 What's Happening While You Wait
>
> CloudFront has **over 450 edge locations** in 90+ countries. When you deploy a distribution,
> AWS propagates your configuration — origin details, cache rules, SSL certificate — to every
> one of them. This is why it takes a few minutes.
>
> When a visitor in Tokyo requests your site, here is what happens in milliseconds:
>
> 1. Their browser looks up `dXXXX.cloudfront.net` in DNS — AWS returns the IP of the **nearest edge location** to Tokyo
> 2. The browser opens an HTTPS connection to that edge location — TLS is terminated there, not in your S3 bucket
> 3. If the edge location has a cached copy of `index.html`, it serves it immediately
> 4. If not (cache miss), the edge makes a **signed request** to your S3 bucket using OAC — S3 verifies the signature and returns the file
> 5. The edge caches the file and serves it to the visitor
>
> All of this happens in under 50ms for a cached response. The visitor never makes a direct connection to your S3 bucket — they only ever talk to CloudFront.

---

### Step 8: Lock Down the S3 Bucket (CloudFront-Only Access)

While CloudFront is deploying, update your S3 bucket. You are going to:
1. Remove the public-read bucket policy from Lab 1C
2. Apply a new policy that only allows your CloudFront distribution to read files
3. Re-enable Block Public Access (the bucket becomes private)

**Step 8a: Remove the old public-read policy**

📋 Copy and paste, **replacing `<YOUR_BUCKET_NAME>`**:

```
aws s3api delete-bucket-policy --bucket <YOUR_BUCKET_NAME>
```

**✅ No output means success.**

**Step 8b: Write the new CloudFront-only bucket policy**

In the VS Code file tree, create a **New File** named `cloudfront-bucket-policy.json`. 📋 Copy and paste this into it, **replacing `<YOUR_BUCKET_NAME>`, `<YOUR_ACCOUNT_ID>`, and `<YOUR_DIST_ID>`**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::<YOUR_BUCKET_NAME>/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::<YOUR_ACCOUNT_ID>:distribution/<YOUR_DIST_ID>"
        }
      }
    }
  ]
}
```
**Save** the file (**Ctrl+S** / **Cmd+S**).


> **What does this policy do?** It allows `s3:GetObject` (read a file) to the AWS CloudFront service principal — but **only** when the request comes from your specific distribution (the `AWS:SourceArn` condition ties it to your Distribution ID). If someone tries to access S3 directly, or if a different CloudFront distribution tries to read your bucket, the request is denied.

**Step 8c: Apply the new bucket policy**

📋 Copy and paste, **replacing `<YOUR_BUCKET_NAME>`**:

```
aws s3api put-bucket-policy --bucket <YOUR_BUCKET_NAME> --policy file://cloudfront-bucket-policy.json
```

**✅ No output means success.**

**Step 8d: Re-enable Block Public Access**

📋 Copy and paste, **replacing `<YOUR_BUCKET_NAME>`**:

```
aws s3api put-public-access-block --bucket <YOUR_BUCKET_NAME> --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

> **What does this do?** Re-enables all four public access blocks that you disabled in Lab 1C. Your bucket is now private — no one can access it directly from the internet. The only allowed reader is your CloudFront distribution via OAC.

**✅ No output means success.** Your S3 bucket is now private.

---

### Step 9: Console Checkpoint

Before testing, verify the full setup in the AWS Console:

**Check CloudFront:**

1. Go to [https://console.aws.amazon.com/](https://console.aws.amazon.com/)
2. Search for **CloudFront** in the top search bar and click it
3. You should see your distribution in the list
4. The **Status** column should show **Enabled** and **Last modified** should be recent
5. Click on the distribution → **General** tab:
   - **Distribution domain name** should show `dXXXXXX.cloudfront.net`
   - **Default root object** should show `index.html`
6. Click the **Origins** tab — confirm the origin is your S3 bucket and the **Origin access** shows your OAC name
7. Click the **Behaviors** tab — confirm the **Viewer protocol policy** shows `Redirect HTTP to HTTPS`

**Check S3:**

8. Search for **S3** and click it
9. Click on your bucket
10. Click the **Permissions** tab
11. Scroll to **Block public access** — all four settings should be **On**
12. Scroll to **Bucket policy** — you should see the `AllowCloudFrontServicePrincipal` statement

**✅ Checkpoint:** CloudFront distribution is Enabled with your OAC-secured S3 origin and redirect-to-HTTPS behavior. S3 bucket shows all public access blocked and a CloudFront-only bucket policy.

---

### Step 10: Test Your HTTPS Website

If the distribution status is `Deployed`, test it now. If it still shows `InProgress`, wait a few more minutes and check again using the command from Step 7.

**Test 1 — HTTPS works:**

Open your browser and navigate to:

```
https://<YOUR_DIST_DOMAIN>
```

(Replace `<YOUR_DIST_DOMAIN>` with your `dXXXXXX.cloudfront.net` domain from Step 6)

**✅ You should see** your static website — with a padlock icon in the browser address bar showing the connection is secure. Click the padlock to confirm: certificate is issued for `*.cloudfront.net` by Amazon.

**Test 2 — HTTP redirects to HTTPS:**

Try visiting the HTTP version:

```
http://<YOUR_DIST_DOMAIN>
```

**✅ You should be automatically redirected** to the HTTPS version. The browser address bar will change to `https://` without you doing anything.

**Test 3 — Old S3 URL is blocked:**

Try visiting your original S3 website URL from Lab 1C:

```
http://<YOUR_BUCKET_NAME>.s3-website-<YOUR_REGION>.amazonaws.com
```

**✅ You should see** an `Access Denied` or `403 Forbidden` error. The bucket is now private and the S3 website endpoint no longer works. CloudFront is the only valid entry point.

**Test 4 — Error page still works:**

Visit a URL that does not exist:

```
https://<YOUR_DIST_DOMAIN>/this-page-does-not-exist
```

**✅ You should see** your custom `error.html` page (from Lab 1C) served with a 404 status. This works because of the `CustomErrorResponses` configuration in Step 5 — S3 returns a 403 for missing files in a private bucket, and CloudFront maps that to your error page.

---

### Step 11: What's Next — Custom Domains

> ## 🌐 Want to Use Your Own Domain? Here's What You Need
>
> Your CloudFront URL (`dXXXXXX.cloudfront.net`) is randomly assigned by AWS. You cannot change
> the `dXXXXXX` part — it is tied to your distribution. If you want a URL like
> `https://www.myportfolio.com` instead, here is what is required:
>
> **1. A registered domain name**
> Register one through AWS Route 53 (~$13/year for `.com`), Namecheap, GoDaddy, or any registrar.
> You own the domain; AWS does not provide one for free.
>
> **2. An ACM SSL certificate in us-east-1**
> AWS Certificate Manager (ACM) provides free public TLS certificates.
> **Important:** CloudFront is a global service and can only use ACM certificates created in the
> `us-east-1 (N. Virginia)` region, even if your S3 bucket is in a different region.
> Request the certificate in us-east-1, validate it via DNS (ACM adds a record to your DNS zone),
> and it renews automatically forever.
>
> **3. Add the domain as a CNAME in CloudFront**
> In your distribution settings, add your domain (e.g., `www.myportfolio.com`) as an
> **Alternate Domain Name (CNAME)**. Select your ACM certificate.
>
> **4. Create a DNS record pointing to CloudFront**
> In Route 53: create an **Alias record** for `www.myportfolio.com` pointing to your
> `dXXXXXX.cloudfront.net` domain.
> At another registrar: create a **CNAME record** doing the same.
>
> Once DNS propagates (usually 1–5 minutes with Route 53, up to 48 hours with other registrars),
> `https://www.myportfolio.com` will serve your site through CloudFront with a valid certificate.
>
> **This is out of scope for this lab** — it requires owning a domain. But if you are building
> a real portfolio site, this is exactly how you would set it up.

---

## What You Just Did

You upgraded a basic HTTP S3 website into a production-grade HTTPS deployment:

1. **Created an OAC** — the cryptographic link allowing CloudFront to access a private S3 bucket
2. **Deployed a CloudFront distribution** — a globally distributed HTTPS endpoint backed by 450+ edge locations
3. **Made S3 private** — the bucket can no longer be accessed directly; all traffic flows through CloudFront
4. **Configured custom error responses** — 403s from private S3 map cleanly to your error page
5. **Tested four scenarios** — HTTPS, HTTP redirect, S3 block, error page

This is the standard architecture used for static websites at companies of all sizes — from side projects to Fortune 500 marketing pages.

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect Associate (SAA-C03)

CloudFront is one of the most tested services on the SAA-C03:
- **OAC vs. OAI:** OAI is legacy and deprecated — OAC is current best practice. The exam increasingly tests OAC. Know the difference: OAC uses IAM Signature Version 4 signing on every request; OAI used a special CloudFront identity in the bucket policy.
- **Viewer Protocol Policy:** `redirect-to-https` vs `https-only`. Redirect is friendlier for users; https-only is stricter. Know when each is appropriate.
- **ACM certificate region:** For CloudFront, the certificate **must** be in `us-east-1` regardless of where your content lives. This is a classic exam trick question.
- **Price Classes:** PriceClass_100 (cheapest, NA + Europe), PriceClass_200 (adds Asia/Middle East/Africa), PriceClass_All (every edge location). Trade-off: cost vs. latency for global users.
- **Cache invalidation:** When you update a file in S3, CloudFront serves the old cached version until the cache expires. You can force an update with `aws cloudfront create-invalidation --distribution-id <ID> --paths "/*"`.

**Sample question type:** "A company hosts a static website on S3. They need HTTPS and want to ensure the S3 bucket cannot be accessed directly. What is the correct solution?"  
**Answer:** Create a CloudFront distribution with an Origin Access Control (OAC). Apply a bucket policy that allows `s3:GetObject` only from `cloudfront.amazonaws.com` conditioned on the specific distribution ARN. Enable Block Public Access on the bucket.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `NoSuchDistribution` error | Distribution ID is wrong or does not exist | Run `aws cloudfront list-distributions --query 'DistributionList.Items[].{Id:Id,Domain:DomainName}'` to find the correct ID |
| Website shows `AccessDenied` XML after deploying | Bucket policy not applied, OAC ID in distribution-config.json does not match, or distribution not yet Deployed | Verify: (1) bucket policy file has correct IDs, (2) CloudFront status is `Deployed` not `InProgress`, (3) OAC ID in distribution matches what you created |
| Website serves stale content after updating files | CloudFront is serving cached content | Run a cache invalidation: `aws cloudfront create-invalidation --distribution-id <ID> --paths "/*"` |
| Error page shows XML `AccessDenied` instead of your custom error.html | `CustomErrorResponses` not configured, or `error.html` was not uploaded to S3 | Verify `error.html` exists in the bucket with `aws s3 ls s3://<BUCKET_NAME>` |
| `MalformedXML` or `InvalidArgument` when applying bucket policy | JSON in `cloudfront-bucket-policy.json` has a placeholder not replaced | Open the file and verify `<YOUR_BUCKET_NAME>`, `<YOUR_ACCOUNT_ID>`, and `<YOUR_DIST_ID>` are all replaced with real values |
| Old S3 HTTP URL still works | Block Public Access not yet applied | Re-run Step 8d |
| Distribution stuck on `InProgress` for more than 25 minutes | Rare CloudFront propagation delay | Wait up to 30 minutes total. If still stuck, check the CloudFront console for errors on the distribution. |

---

## Cleanup

**⚠️ Important:** CloudFront distributions must be **disabled** before they can be deleted. Disabling takes ~5 minutes. Follow these steps in order.

> **Note:** If you want to keep your HTTPS website running, you can skip cleanup. CloudFront is within the always-free tier for a portfolio-sized site.

### Step 1: Disable the Distribution

📋 Copy and paste, **replacing `<YOUR_DIST_ID>`**:

First, get the current ETag (required for updates):

```
aws cloudfront get-distribution-config --id <YOUR_DIST_ID> --query 'ETag' --output text
```

> **📝 Write down the ETag:** ______________________________ (looks like `E2QWRUHEXAMPLE`)

Get the current config and save it to a file:

**Windows (PowerShell):**

```powershell
aws cloudfront get-distribution-config --id <YOUR_DIST_ID> --query "DistributionConfig" | Out-File -FilePath current-dist-config.json -Encoding ascii
```

**macOS / Linux:**

```bash
aws cloudfront get-distribution-config --id <YOUR_DIST_ID> --query "DistributionConfig" > current-dist-config.json
```

Open `current-dist-config.json` in VS Code. Find the top-level **`"Enabled": true`** field (near the top of the file) and change it to **`"Enabled": false`**, then save. (Tip: press **Ctrl+F** / **Cmd+F** and search for `Enabled` to jump straight to it.) Then run:

📋 Copy and paste, **replacing `<YOUR_DIST_ID>` and `<ETAG>`**:

```
aws cloudfront update-distribution --id <YOUR_DIST_ID> --distribution-config file://current-dist-config.json --if-match <ETAG>
```

**✅ You should see** `"Status": "InProgress"` — the distribution is being disabled.

### Step 2: Wait for the Distribution to Disable

📋 Copy and paste, **replacing `<YOUR_DIST_ID>`**:

```
aws cloudfront get-distribution --id <YOUR_DIST_ID> --query 'Distribution.Status' --output text
```

Run this every 2 minutes until you see `Deployed` (this means the disable has been deployed). The `Enabled` flag in the config is now `false`.

You can further confirm by running the below command and verify the false variable has been deployed:

```
aws cloudfront get-distribution --id <YOUR_DIST_ID> --query "{Status:Distribution.Status,Enabled:Distribution.DistributionConfig.Enabled}" --output table
```

### Step 3: Delete the Distribution

Get the current ETag again (it changes when you updated the distribution):

```
aws cloudfront get-distribution --id <YOUR_DIST_ID> --query 'ETag' --output text
```

📋 Copy and paste, **replacing `<YOUR_DIST_ID>` and `<NEW_ETAG>`**:

```
aws cloudfront delete-distribution --id <YOUR_DIST_ID> --if-match <NEW_ETAG>
```

**✅ No output means success.**

### Step 4: Delete the OAC

📋 Copy and paste, **replacing `<YOUR_OAC_ID>`**:

First get the OAC ETag:

```
aws cloudfront get-origin-access-control --id <YOUR_OAC_ID> --query 'ETag' --output text
```

Then delete it:

```
aws cloudfront delete-origin-access-control --id <YOUR_OAC_ID> --if-match <OAC_ETAG>
```

### Step 5: Delete Local Files

> **⚠️ Close VS Code first.** If VS Code still has the `workshop-lab-1d` folder open, the delete will fail — especially on Windows. Choose **File → Close Folder** or quit VS Code before running the commands below.

**Windows (PowerShell):**

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-1d
```

**macOS / Linux:**

```bash
cd ~/Desktop
rm -rf ~/Desktop/workshop-lab-1d
```

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** you received (copy and paste it)
4. Your **operating system** (Windows, Mac, or Linux)
