# AWS Services & Concepts Glossary

A quick reference for every AWS service and concept used in the labs. Bookmark this page and refer back whenever you forget what something is.

---

## AWS Services

| Service | What It Does (Plain English) | First Used In |
|---------|------------------------------|---------------|
| **AWS CLI** | A program on your computer that lets you control AWS by typing commands | Lab 1A |
| **IAM Identity Center** | The modern, secure login system for AWS accounts (SSO) | Lab 1A |
| **AWS Organizations** | Groups multiple AWS accounts together under one management structure | Lab 1A |
| **AWS STS** | Security Token Service — verifies your identity and issues temporary credentials | Lab 1A |
| **Amazon SNS** | Simple Notification Service — sends messages (email, SMS) to subscribers | Lab 1B |
| **AWS Budgets** | Tracks your AWS spending and sends alerts when you approach a limit | Lab 1B |
| **Amazon S3** | Simple Storage Service — cloud storage for files (buckets and objects) | Lab 1C |
| **Amazon EC2** | Elastic Compute Cloud — virtual servers you rent in the cloud | Lab 2A |
| **AWS Systems Manager** | Manages and connects to EC2 instances (Session Manager = remote terminal) | Lab 2A |
| **AWS Lambda** | Serverless compute — runs your code without managing servers, only when triggered | Lab 2C |
| **Lambda Function URL** | A public HTTP endpoint for a Lambda function (no API Gateway needed) | Lab 2C |
| **IAM** | Identity and Access Management — controls who can do what in your account | Lab 3A |
| **Amazon GuardDuty** | Continuous threat detection — monitors for suspicious activity 24/7 | Lab 4A |
| **AWS CloudTrail** | Audit log of every API call in your account (who did what, when) | Lab 4B |
| **Amazon EventBridge** | Event routing — matches events to rules and triggers actions automatically | Lab 4C |
| **AWS Security Hub** | Aggregates security findings from multiple services into one dashboard | Session 4 (lecture) |
| **AWS Config** | Tracks configuration changes to your resources over time | Session 4 (lecture) |
| **Amazon CloudWatch** | Monitoring service — collects metrics, stores logs, triggers alarms | Lab 9A |
| **CloudWatch Alarm** | A rule that watches a metric and takes action when it crosses a threshold | Lab 9A |
| **CloudWatch Logs** | Stores text output from Lambda, EC2, and other services (log groups + streams) | Lab 9B |
| **CloudWatch Logs Insights** | Query engine for searching and analysing logs at scale | Lab 9B |
| **CloudWatch Dashboard** | Customizable page showing multiple metrics in a single operational view | Lab 9C |

---

## Key Concepts

| Concept | What It Means | First Used In |
|---------|---------------|---------------|
| **Region** | A geographic area with multiple AWS data centers (e.g., us-east-1 = N. Virginia) | Lab 1A |
| **Availability Zone (AZ)** | An individual data center within a region | Session 1 (lecture) |
| **Free Tier** | AWS services you can use at no cost within specified limits | Lab 1A |
| **Always Free** | Services that are free forever (not just 12 months) within usage limits | Lab 1B |
| **Bucket** | An S3 container for storing files (like a folder in the cloud) | Lab 1B |
| **Bucket Policy** | A JSON document that controls who can access files in an S3 bucket | Lab 1C |
| **Static Website Hosting** | S3 feature that serves HTML files as a website | Lab 1C |
| **Instance** | A single virtual server running on EC2 | Lab 2A |
| **Instance Type** | The size of an EC2 server (CPU + memory). `t2.micro` = smallest/free | Lab 2A |
| **AMI** | Amazon Machine Image — the operating system template for an EC2 instance | Lab 2A |
| **Instance Profile** | A container that attaches an IAM role to an EC2 instance | Lab 2A |
| **Session Manager** | A way to connect to EC2 instances through your browser (no SSH keys needed) | Lab 2A |
| **Presigned URL** | A temporary URL that grants permission to upload/download from S3 (expires) | Lab 2C |
| **CORS** | Cross-Origin Resource Sharing — browser security that blocks cross-domain requests unless allowed | Lab 2C |
| **Event-Driven Architecture** | Code runs in response to events (not on a schedule or continuously) | Lab 2C |
| **IAM User** | An identity with permanent credentials (for systems, not humans) | Lab 3A |
| **IAM Group** | A collection of users that share the same permissions | Lab 3B |
| **IAM Role** | A temporary identity that anyone can "assume" to get specific permissions | Lab 3C |
| **IAM Policy** | A JSON document that defines what actions are allowed or denied | Lab 3A |
| **Inline Policy** | A policy attached directly to one user, group, or role (not reusable) | Lab 3A |
| **Least Privilege** | Give only the minimum permissions needed — nothing more | Lab 3A |
| **Explicit Deny** | A Deny statement in a policy — overrides any Allow, cannot be bypassed | Lab 3B |
| **Implicit Deny** | No policy = denied by default. Can be overridden by adding an Allow | Lab 3B |
| **MFA** | Multi-Factor Authentication — requires password + phone code to log in | Lab 3B |
| **Trust Policy** | Defines WHO is allowed to assume a role | Lab 3C |
| **Assume Role** | Switching to a role's identity to get its temporary credentials | Lab 3C |
| **Session Token** | Extra credential required when using temporary role credentials | Lab 3C |
| **Separation of Duties** | No single person has all the power (developers ≠ auditors ≠ admins) | Lab 3C |
| **Detector** | The GuardDuty resource that represents the service being enabled | Lab 4A |
| **Finding** | A security alert generated by GuardDuty when it detects suspicious activity | Lab 4A |
| **Severity Level** | How urgent a finding is: Low (1-3), Medium (4-6), High (7-8) | Lab 4A |
| **Trail** | A CloudTrail configuration that stores logs permanently in S3 | Lab 4B |
| **Event Pattern** | A JSON filter that tells EventBridge which events to match | Lab 4C |
| **SOAR** | Security Orchestration, Automation, and Response — automating security workflows | Lab 4C |
| **Incident Response** | The process of detecting, containing, and recovering from a security event | Lab 5A |
| **Containment** | Stopping an incident from spreading (revoke credentials, isolate systems) | Lab 5A |
| **Eradication** | Removing the threat completely (delete compromised keys, users, resources) | Lab 5A |
| **Credential Revocation** | Deactivating or deleting access keys to stop unauthorized access | Lab 5A |
| **Forensic Timeline** | A chronological reconstruction of events during an incident | Lab 5B |
| **Incident Report** | A formal document recording what happened, actions taken, and lessons learned | Lab 5B |
| **Automated Remediation** | Using Lambda to automatically fix security issues without human intervention | Lab 5C |
| **Metric** | A single measurement tracked over time (e.g., error count, duration, invocations) | Lab 9A |
| **Alarm State** | An alarm's current condition: OK (healthy), ALARM (threshold crossed), INSUFFICIENT_DATA (not enough data) | Lab 9A |
| **Log Group** | A CloudWatch Logs container for logs from one source (e.g., `/aws/lambda/<function-name>`) | Lab 9B |
| **Log Stream** | An individual sequence of log events within a log group (one per Lambda execution environment) | Lab 9B |
| **Structured Logging** | Writing logs as JSON for machine-searchability (query by field, level, request ID) | Lab 9B |
| **MTTD (Mean Time to Detect)** | How long between "something broke" and "we know it's broken" — alarms reduce this | Lab 9B |
| **MTTR (Mean Time to Resolve)** | How long between "we know it's broken" and "it's fixed" — good logs reduce this | Lab 9B |
| **Smoke Test** | A quick automated check after deployment that verifies the application runs without crashing | Lab 9C |
| **Shift Left** | Catching problems earlier in the process (in CI, not in production) | Lab 9C |
| **Defence in Depth** | Using multiple complementary safeguards (CI tests + monitoring + alarms) rather than relying on one | Lab 9C |

---

## Common CLI Commands Reference

| Command | What It Does |
|---------|-------------|
| `aws sts get-caller-identity` | Shows who you are currently authenticated as |
| `aws sso login --profile <NAME>` | Logs you in via SSO (opens browser) |
| `aws s3 mb s3://<BUCKET>` | Creates an S3 bucket |
| `aws s3 cp <SOURCE> <DEST>` | Copies files to/from S3 |
| `aws s3 rm s3://<BUCKET> --recursive` | Deletes all files in a bucket |
| `aws s3 rb s3://<BUCKET>` | Deletes an empty bucket |
| `aws iam create-user --user-name <NAME>` | Creates an IAM user |
| `aws iam create-role --role-name <NAME>` | Creates an IAM role |
| `aws iam attach-role-policy` | Attaches a managed policy to a role |
| `aws iam put-user-policy` | Attaches an inline policy to a user |
| `aws iam create-access-key` | Creates access keys for a user |
| `aws iam update-access-key --status Inactive` | Deactivates an access key |
| `aws iam delete-access-key` | Permanently deletes an access key |
| `aws ec2 run-instances` | Launches an EC2 instance |
| `aws ec2 terminate-instances` | Terminates (deletes) an EC2 instance |
| `aws lambda create-function` | Creates a Lambda function |
| `aws lambda invoke` | Manually runs a Lambda function |
| `aws guardduty create-detector --enable` | Enables GuardDuty |
| `aws guardduty delete-detector` | Disables GuardDuty |
| `aws cloudtrail lookup-events` | Searches CloudTrail event history |
| `aws events put-rule` | Creates an EventBridge rule |
| `aws events put-targets` | Adds a target to an EventBridge rule |
| `aws cloudwatch get-metric-statistics` | Retrieves metric data points for a specified period |
| `aws cloudwatch put-metric-alarm` | Creates or updates a CloudWatch alarm |
| `aws cloudwatch describe-alarms` | Shows alarm state (OK, ALARM, INSUFFICIENT_DATA) |
| `aws cloudwatch delete-alarms` | Deletes one or more alarms |
| `aws cloudwatch put-dashboard` | Creates or updates a CloudWatch dashboard |
| `aws logs describe-log-groups` | Lists CloudWatch Log groups |
| `aws logs describe-log-streams` | Lists log streams within a log group |
| `aws logs get-log-events` | Reads log entries from a specific stream |
| `aws logs start-query` | Starts a CloudWatch Logs Insights query |
| `aws logs get-query-results` | Gets results from a Logs Insights query |
| `aws lambda update-function-code` | Updates a Lambda function's code (redeploy) |
| `aws sns create-topic` | Creates an SNS notification topic |
| `aws sns subscribe` | Subscribes an email/endpoint to a topic |

---

*This glossary is updated as new sessions are added.*
