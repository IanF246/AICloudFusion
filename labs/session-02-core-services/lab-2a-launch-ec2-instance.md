# Lab 2A: Launch and Connect to a Virtual Server (EC2)

**Session:** 2 — Core AWS Services  
**Track:** Cloud Basics  
**Difficulty:** Beginner  
**Estimated Time:** 20–25 minutes

---

## Overview

In this lab, you will launch a **virtual server** on AWS using Amazon EC2 (Elastic Compute Cloud). This is one of the most fundamental cloud operations — spinning up a computer in the cloud that you can connect to and use. You will connect to it using **Session Manager** (no SSH keys or passwords needed), run some commands on it, and then terminate (delete) it.

**What you will do:**
- Create an IAM role so the server can communicate with AWS services
- Launch one of the smallest EC2 instances available, the `t3a.nano` (as of 2026)
- Connect to it using Session Manager (a browser-based terminal)
- Explore the instance (check OS, memory, disk)
- Terminate the instance and verify it's gone

---

## Prerequisites

- ✅ Completed **Lab 1A** (AWS account, Identity Center, CLI configured)
- ✅ AWS CLI authenticated — run `aws sts get-caller-identity` and confirm it returns your account

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|----------------|
| Amazon EC2 (e.g., t3a.nano) | Virtual server | $0.0047/hr |
| AWS Systems Manager (Session Manager) | Remote connection to instances | Always Free |
| IAM | Identity and access management | Always Free |

**Estimated cost for this lab: ~$0.01** (a few cents for the short time the instance runs)

>[!CAUTION]
>If you leave the EC2 instance running _by mistake_, a full month would cost roughly **$3.80** (compute plus a small EBS storage charge) — which is exactly why we terminate resources at the end of every lab.
>
>Always check your Billing and Costing service within the console to see if any services are running unexpected costs.

---

## Concepts

**Amazon EC2 (Elastic Compute Cloud)** is a service that lets you rent virtual servers in the cloud. Each server is called an **instance**. You choose the size (how much CPU and memory), the operating system, and the region. It's like renting a computer that lives in an AWS data center — you can connect to it, install software, run applications, and delete it when you're done.

**Instance type** describes the size of the server. `t3a.nano` is the smallest currently — 2 vCPU, 0.5 GB memory and perfect for learning.

**AMI (Amazon Machine Image)** is the operating system template. We will use **Amazon Linux 2023** — a Linux distribution made by AWS that comes with useful tools pre-installed.

**Session Manager** is a way to connect to your instance through your browser or CLI — no SSH keys, no passwords, no open ports needed. It uses the **SSM Agent** (pre-installed on Amazon Linux) to create a secure connection.

**IAM Instance Profile** is a way to give your EC2 instance permissions to use other AWS services. Instead of putting access keys on the server (insecure), you attach a role that grants specific permissions.

**Amazon VPC (Virtual Private Cloud)** is your isolated network inside AWS where your EC2 instances and other resources live. It controls IP ranges, subnets, routing, and security. Every AWS account includes a default VPC in each region. If you launch an EC2 instance without choosing a VPC, AWS automatically places it into that region’s default VPC.

---

## ⚠️ Placeholders in This Lab

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name from Lab 1A | `AdministratorAccess-123456789012` |
| `<YOUR_INSTANCE_ID>` | The instance ID returned when you launch EC2 (you'll get this during the lab) | `i-0abc123def456789` |

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

**Verify it works.** 📋 Copy and paste:

```
aws sts get-caller-identity
```

**✅ You should see** your account ID and role. If you get an error about an expired token, run `aws sso login --profile <YOUR_PROFILE_NAME>`

---

### Step 2: Create Your Project Folder

Before creating any files, let's set up a dedicated folder for this lab. This keeps your files organized and ensures your terminal can find them.

**Step 2a: Create the folder**

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
mkdir ~\Desktop\workshop-lab-2a
cd ~\Desktop\workshop-lab-2a
```

**macOS / Linux:**

📋 Copy and paste:

```bash
mkdir ~/Desktop/workshop-lab-2a
cd ~/Desktop/workshop-lab-2a
```

> **What does this do?** Creates a new folder on your Desktop called `workshop-lab-2a` and moves your terminal into that folder. All commands you run and files you create will be in this folder.

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

**✅ You should see** a path ending in `workshop-lab-2a` (e.g., `C:\Users\YourName\Desktop\workshop-lab-2a` on Windows or `/Users/YourName/Desktop/workshop-lab-2a` on Mac).

> **💡 From now on, save ALL files you create in this lab to this folder.** When the lab says "save the file," save it here. This is where your terminal is looking for files.

**Step 2c: Open the folder in VS Code**

📋 Copy and paste:

```
code .
```

> **What does this do?** This opens VS Code with `workshop-lab-2a` as its **file tree** on the left, so the file you create in the next step lands in the right place. (You set up the `code` command in Lab 1B — if you see `'code' is not recognized`, close and reopen your terminal, or revisit Lab 1B, Step 6.)

---

### Step 3: Create an IAM Role for the EC2 Instance

Before launching the instance, we need to create a **role** that gives it permission to use Session Manager. Without this role, you won't be able to connect to the instance.

1. In the VS Code file tree, click the **New File** icon and name the file `ec2-trust-policy.json`.

2. 📋 Copy and paste this entire block into it:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

3. **Save** the file (**Ctrl+S** / **Cmd+S**). You should see `ec2-trust-policy.json` appear in the file tree.

> **⚠️ Common mistake:** Make sure the name is exactly `ec2-trust-policy.json`, not `ec2-trust-policy.json.txt`. Naming it in the file tree with a `.json` ending sets the file type automatically.

> **What does this file do?** This is a "trust policy" — it tells AWS that EC2 instances are allowed to use this role. Think of it as saying "this role is for EC2 servers."

**Now create the role.** 📋 Copy and paste:

```
aws iam create-role --role-name workshop-ec2-ssm-role --assume-role-policy-document file://ec2-trust-policy.json
```

> **What does this do?** Creates a new IAM role called `workshop-ec2-ssm-role` that EC2 instances can use.

**Attach the Session Manager policy to the role.** 📋 Copy and paste:

```
aws iam attach-role-policy --role-name workshop-ec2-ssm-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

> **What does this do?** Gives the role permission to communicate with Session Manager. Without this, you can't connect to the instance remotely.

**Create an instance profile and add the role to it.** 📋 Copy and paste these two commands:

```
aws iam create-instance-profile --instance-profile-name workshop-ec2-ssm-profile
```

```
aws iam add-role-to-instance-profile --instance-profile-name workshop-ec2-ssm-profile --role-name workshop-ec2-ssm-role
```

> **What is an instance profile?** It's a container that holds the role and attaches it to an EC2 instance. Think of it as a badge holder — the role is the badge, and the instance profile clips it onto the server.

**Wait 10 seconds** for AWS to propagate the changes (IAM changes take a moment to take effect globally).

---

### Step 4: Get the Latest Amazon Linux AMI ID

We need the ID of the operating system image to use. This command looks up the latest Amazon Linux 2023 AMI:

📋 Copy and paste:

```
aws ssm get-parameters-by-path --path "/aws/service/ami-amazon-linux-latest" --region us-east-1 --query "Parameters[?Name=='/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64'].Value" --output text
```

> **What does this do?** Asks AWS for the latest Amazon Linux 2023 image ID. The ID changes periodically as AWS releases updates.

**✅ You should see** an AMI ID like: `ami-0a59ec92177ec3fad`

> **📝 Write down the AMI ID:** ______________________________

---

### Step 5: Launch the EC2 Instance

Now launch your virtual server. 📋 Copy and paste this command, **replacing `<AMI_ID>`** with the ID from Step 4:

**macOS / Linux:**

```bash
aws ec2 run-instances \
    --image-id <AMI_ID> \
    --instance-type t3a.nano \
    --iam-instance-profile Name=workshop-ec2-ssm-profile \
    --region us-east-1 \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=workshop-session2}]' \
    --query "Instances[0].InstanceId" \
    --output text
```

**Windows (PowerShell):**

```powershell
aws ec2 run-instances --image-id <AMI_ID> --instance-type t3a.nano --iam-instance-profile Name=workshop-ec2-ssm-profile --region us-east-1 --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=workshop-session2}]" --query "Instances[0].InstanceId" --output text
```

> **What does this do?**
> - `--image-id` — which operating system to use (Amazon Linux 2023)
> - `--instance-type t3a.nano` — the smallest server size
> - `--iam-instance-profile` — attaches the role so Session Manager works
> - `--tag-specifications` — gives the instance a name so you can find it in the console
> - `--query` / `--output text` — returns just the instance ID

**✅ You should see** an instance ID like: `i-0abc123def456789`

> **📝 Write down your Instance ID:** ______________________________

**✅ Checkpoint — Verify in the AWS Console:**
1. Go to the AWS Console → search for **EC2** → click it
2. In the left sidebar, click **Instances**
3. You should see an instance named **workshop-session2** with state **Running** (it may say "Pending" for a minute — wait and refresh)

---

### Step 6: Wait for the Instance to Be Ready

The instance needs about 1–2 minutes to fully start up and register with Session Manager.

📋 Copy and paste this command, **replacing `<YOUR_INSTANCE_ID>`**:

```
aws ec2 wait instance-running --instance-ids <YOUR_INSTANCE_ID> --region us-east-1
```

> **What does this do?** Waits until the instance is in the "running" state. The command will hang (no output) until it's ready, then return to the prompt.

**Now wait for Session Manager to register** (about 30 more seconds after the instance is running):

📋 Copy and paste, **replacing `<YOUR_INSTANCE_ID>`**:

```
aws ssm describe-instance-information --region us-east-1 --filters "Key=InstanceIds,Values=<YOUR_INSTANCE_ID>" --query "InstanceInformationList[0].PingStatus" --output text
```

**✅ You should see:** `Online`

> If you see `None` or an empty result, wait 30 seconds and try again. The SSM agent needs time to register.

---

### Step 7: Connect to Your Instance via Session Manager

Now you will connect to your virtual server — you'll get a terminal session on the remote machine.

**Option A: Connect via the AWS Console (easiest for beginners):**

1. Go to **EC2** → **Instances** in the AWS Console
2. Select your instance (check the box next to it)
3. Click the **Connect** button at the top
4. Choose the **Session Manager** tab
5. Click **Connect**
6. A new browser tab will open with a terminal — you are now on your EC2 instance!

**Option B: Connect via the CLI:**

📋 Copy and paste, **replacing `<YOUR_INSTANCE_ID>`**:

```
aws ssm start-session --target <YOUR_INSTANCE_ID> --region us-east-1
```

> **Note:** This requires the Session Manager plugin for the AWS CLI. If you get an error, use Option A (the console) instead.

---

### Step 8: Explore Your Instance

You are now connected to a Linux server running in the cloud. Try these commands to explore:

📋 Copy and paste each command one at a time:

**Check the operating system:**
```bash
cat /etc/os-release
```
> You should see "Amazon Linux 2023"

**Check how much memory the server has:**
```bash
free -h
```
> You should see about 0.5 GB / 512 MB (this is a t3a.nano)

**Check disk space:**
```bash
df -h
```
> You should see the root filesystem with about 8 GB

**Check who you are:**
```bash
whoami
```
> You should see `ssm-user` (the user Session Manager connects as)

**Check the instance's private IP address:**
```bash
hostname -I
```

> **🎉 You are running commands on a computer in an AWS data center!** This is exactly what cloud engineers do — manage remote servers from their local machine.

**When you're done exploring, disconnect:**
```bash
exit
```

Or if you used the console, just close the browser tab.

---

### Step 9: View Instance Details from the CLI

Back on your local terminal, let's check the instance details:

📋 Copy and paste, **replacing `<YOUR_INSTANCE_ID>`**:

```
aws ec2 describe-instances --instance-ids <YOUR_INSTANCE_ID> --region us-east-1 --query "Reservations[0].Instances[0].{State:State.Name,Type:InstanceType,AZ:Placement.AvailabilityZone,PublicIP:PublicIpAddress,LaunchTime:LaunchTime}" --output table
```

**✅ You should see** a table showing the instance is `running`, type `t3a.nano`, with an availability zone and public IP.

---

## What You Just Did

You launched a **virtual server in the cloud** — the same operation that cloud engineers, DevOps engineers, and SREs perform daily. You:

1. Created an IAM role with Session Manager permissions
2. Launched a t3a.nano EC2 instance with Amazon Linux 2023
3. Connected to it remotely using Session Manager (no SSH keys needed)
4. Explored the server's OS, memory, and disk
5. Viewed instance details from the CLI

In a real job, you would use EC2 instances to run web applications, databases, batch processing jobs, and more. The t3a.nano you launched is the same type of server — just smaller — that powers production workloads at companies worldwide.

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

EC2 is one of the most heavily tested services on the SAA exam. Understanding instance types, AMIs, security groups, and IAM roles for EC2 is essential.

**Sample question type:** "A company needs to run a web application that requires 4 vCPUs and 16 GB of memory. Which EC2 instance family should they use?"

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `InvalidParameterValue` when launching | The AMI ID may be wrong or expired | Re-run the command in Step 4 to get the latest AMI ID |
| Session Manager says "not connected" | The SSM agent hasn't registered yet | Wait 1–2 minutes and try again. Make sure the IAM role was attached correctly. |
| `An error occurred (UnauthorizedOperation)` | Your CLI profile doesn't have EC2 permissions | Make sure you're using the AdministratorAccess profile |
| Instance stuck in "Pending" state | Normal — it takes 1–2 minutes to start | Wait and refresh the console page |

---

## Cleanup

**⚠️ Important:** Always terminate EC2 instances when you're done. A running instance uses credits.

### Step 1: Terminate the Instance

📋 Copy and paste, **replacing `<YOUR_INSTANCE_ID>`**:

```
aws ec2 terminate-instances --instance-ids <YOUR_INSTANCE_ID> --region us-east-1
```

**✅ You should see** the state change to `shutting-down`.

### Step 2: Wait for Termination

```
aws ec2 wait instance-terminated --instance-ids <YOUR_INSTANCE_ID> --region us-east-1
```

### Step 3: Delete the IAM Role and Instance Profile

📋 Copy and paste these commands in order:

```
aws iam detach-role-policy --role-name workshop-ec2-ssm-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

```
aws iam remove-role-from-instance-profile --instance-profile-name workshop-ec2-ssm-profile --role-name workshop-ec2-ssm-role
```

```
aws iam delete-instance-profile --instance-profile-name workshop-ec2-ssm-profile
```

```
aws iam delete-role --role-name workshop-ec2-ssm-role
```

### Step 4: Delete Local Files

> **⚠️ Close VS Code first.** If VS Code still has the `workshop-lab-2a` folder open, the delete will fail — especially on Windows. Choose **File → Close Folder** or quit VS Code before running the commands below. The commands also move you to your home directory (`cd ~`) first so the terminal isn't sitting inside the folder it's deleting.

Remove the project folder you created for this lab:

**macOS / Linux:**

```bash
cd ~
rm -rf ~/Desktop/workshop-lab-2a
```

**Windows (PowerShell):**

```powershell
cd ~
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-2a
```

### Step 5: Verify Cleanup

**✅ Checkpoint — In the AWS Console:**
1. Go to **EC2** → **Instances** — your instance should show state **Terminated** (it will disappear after a few hours)
2. Go to **IAM** → **Roles** — `workshop-ec2-ssm-role` should be gone

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **command** you ran (copy and paste it)
2. The **full error message** you received
3. Which **step number** you are on
