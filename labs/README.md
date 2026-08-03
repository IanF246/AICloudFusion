# AI Cloud Fusion — Lab Guides

Welcome to the AI Cloud Fusion workshop lab guides. These hands-on labs accompany the weekly workshop sessions and are designed to be completed independently using only the instructions in each guide.

## Getting Started

**New here?** Start with [Lab 1A](session-01-cloud-concepts/lab-1a-aws-cli-setup.md) — it walks you through creating your AWS account and setting up everything you need from scratch. No prior experience required.

## How to Use These Labs

Each session has its own folder containing three labs at different difficulty levels:

| Level | What to Expect |
|-------|---------------|
| **Lab A (Beginner)** | Foundational setup or introductory concepts |
| **Lab B (Intermediate)** | Core hands-on task with multiple AWS services |
| **Lab C (Advanced)** | Real-world engineering task — you build something practical |

Work through the labs in order (A → B → C). Each lab includes:

- 📋 Copy-and-paste commands with clear instructions
- 🔄 Placeholders clearly marked so you know what to replace
- 💡 Annotations explaining what each service does and why
- ✅ Console checkpoints to verify your work visually
- 🧹 Cleanup/decommission steps so you don't incur any costs

## Session Labs

| Session | Topic | Labs |
|---------|-------|------|
| 1 | Cloud Concepts & AWS Global Infrastructure | [Lab 1A: AWS Account & CLI Setup](session-01-cloud-concepts/lab-1a-aws-cli-setup.md) · [Lab 1B: Cost Budget & SNS Alert](session-01-cloud-concepts/lab-1b-cost-budget-sns-alert.md) · [Lab 1C: S3 Static Website](session-01-cloud-concepts/lab-1c-s3-static-website.md) |
| 2 | Core AWS Services | [Lab 2A: Launch EC2 Instance](session-02-core-services/lab-2a-launch-ec2-instance.md) · [Lab 2B: EC2 + S3 with IAM Role](session-02-core-services/lab-2b-ec2-s3-iam-role.md) · [Lab 2C: Serverless File Pipeline](session-02-core-services/lab-2c-serverless-file-pipeline.md) |
| 3 | Cloud Security Fundamentals | [Lab 3A: IAM Least Privilege](session-03-cloud-security/lab-3a-iam-least-privilege.md) · [Lab 3B: Groups, Custom Policy & MFA](session-03-cloud-security/lab-3b-groups-custom-policy-mfa.md) · [Lab 3C: Roles & CloudTrail Audit](session-03-cloud-security/lab-3c-roles-cloudtrail-audit.md) |
| 4 | Threat Detection & AWS Security Services | [Lab 4A: GuardDuty Findings](session-04-threat-detection/lab-4a-guardduty-findings.md) · [Lab 4B: CloudTrail Investigation](session-04-threat-detection/lab-4b-cloudtrail-investigation.md) · [Lab 4C: Automated Security Alerts](session-04-threat-detection/lab-4c-automated-security-alerts.md) |
| 5 | Incident Response on AWS | [Lab 5A: Credential Revocation](session-05-incident-response/lab-5a-credential-revocation.md) · [Lab 5B: Investigate & Report](session-05-incident-response/lab-5b-investigate-and-report.md) · [Lab 5C: Automated Remediation](session-05-incident-response/lab-5c-automated-remediation.md) |
| 6 | AWS Well-Architected Framework | [Lab 6A: S3 Security & Reliability](session-06-well-architected/lab-6a-s3-security-reliability.md) *(6B, 6C coming soon)* |
| 7 | IaC Fundamentals with OpenTofu | [Lab 7A: OpenTofu Setup](session-07-iac-fundamentals/lab-7a-opentofu-setup.md) · [Lab 7B: Lambda with IaC](session-07-iac-fundamentals/lab-7b-lambda-with-iac.md) · [Lab 7C: Event-Driven Modules](session-07-iac-fundamentals/lab-7c-event-driven-modules.md) |
| 8 | CI/CD for Infrastructure | [Lab 8A: GitHub OIDC](session-08-cicd/lab-8a-github-oidc.md) · [Lab 8B: Pipeline Plan & Apply](session-08-cicd/lab-8b-pipeline-plan-apply.md) · [Lab 8C: Drift Detection](session-08-cicd/lab-8c-drift-detection.md) |
| 9 | Monitoring & Observability | [Lab 9A: CloudWatch Alarms](session-09-monitoring/lab-9a-cloudwatch-alarms.md) · [Lab 9B: Diagnose with Logs](session-09-monitoring/lab-9b-diagnose-with-logs.md) · [Lab 9C: CI Smoke Test](session-09-monitoring/lab-9c-ci-smoke-test.md) |
| 10 | Chatbot Observability | [Lab 10A: Deploy Chatbot](session-10-chatbot-observability/lab-10a-deploy-chatbot.md) · [Lab 10B: Monitor Dependencies](session-10-chatbot-observability/lab-10b-monitor-dependencies.md) · [Lab 10C: Resilient Fallback](session-10-chatbot-observability/lab-10c-resilient-fallback.md) |

*More sessions coming soon.*

## Prerequisites

- An email address, phone number, and credit/debit card (for AWS account creation)
- A computer running Windows, macOS, or Linux
- No prior AWS or cloud experience needed — Lab 1A covers everything from scratch

## Cost

All labs use **Always Free** AWS resources and include cleanup steps at the end. Each lab has a cost notice listing every service used and confirming the estimated cost is $0.00. **Always complete the Cleanup section at the end of each lab** to ensure no resources are left running.

## Need Help?

If you get stuck on any lab, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **command** you ran (copy and paste it)
2. The **full error message** you received
3. Which **step number** you are on
