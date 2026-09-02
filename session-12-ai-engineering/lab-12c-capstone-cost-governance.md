# Lab 12C: Capstone — Ship It. Cost Governance & the Full AI Stack

**Session:** 12 — AI Engineering (Capstone)
**Track:** AI Engineering
**Difficulty:** Advanced
**Estimated Time:** 50–60 minutes
**Target Cert:** AWS Certified AI Practitioner

---

## Overview

This is the finale of the AI Engineering track. Across Sessions 11 and 12 you built a real, serverless AI product piece by piece. In this capstone you'll add the last professional layer — **account-level cost governance** — then run a full end-to-end verification of the *entire* stack, review it against a production-readiness checklist, and close out the track.

You'll:

1. **Set an account-level cost budget** with an email alert (the outermost financial guardrail)
2. **Assemble and verify** the complete system end-to-end (RAG + guardrails + monitoring + budget)
3. **Review** your architecture against a production-readiness checklist
4. **Reflect** on everything you built across the AI Engineering track
5. **Cleanly tear down** every resource from Sessions 11 and 12

**The lesson:** Anyone can wire an AI model to an endpoint. **Shipping** one means grounding it (12A), making it safe (12B), monitoring it (11C), and putting a hard financial ceiling on it (this lab) — then being able to prove all of it works. That's the difference between a demo and a product.

---

## Prerequisites

- ✅ Completed **Labs 11A–11C and 12A–12B** — you should have:
  - `workshop-ai-chatbot-lab11` Lambda (with system prompt, input validation, RAG, and guardrails)
  - `workshop-lab11-lambda-role` role
  - `workshop-ai-kb-<YOUR_ACCOUNT_ID>` S3 knowledge base
  - `workshop-ai-guardrail` guardrail
  - `workshop-ai-chatbot-dashboard` CloudWatch dashboard and the `ai-chatbot-token-spike` alarm
- ✅ AWS CLI authenticated
- ✅ Your `~/Desktop/workshop-lab-11a` project folder

> **📌 Missing pieces?** You can still do the cost-governance part (Steps 2–4) and the retrospective on their own — but the full end-to-end test (Step 5) assumes the whole stack is in place.

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| AWS Budgets | Monthly cost budget + email alert | Always Free (first 2 budgets) |
| AWS Lambda / Bedrock / CloudWatch | Final verification invocations | ~$0.01 for this lab |

**⚠️ Estimated cost for this lab: under $0.02**

---

## Concepts

**AWS Budgets** — an account-level tool that tracks your actual (and forecasted) spend and emails you when it crosses a threshold you set. It's the *outermost* cost control: even if every in-app guardrail failed, the budget still warns you before a small bill becomes a big one.

**Defence in depth for cost** — you now have **three** layers:
1. **Input validation** (Lab 11C) — reject expensive prompts before calling the model
2. **maxTokens + token alarm** (Labs 11B/11C) — cap and monitor per-request spend
3. **Account budget** (this lab) — a hard ceiling on total monthly spend, independent of the app

**Production-readiness** — the checklist a team runs before putting a system in front of real users: Is it grounded? Safe? Monitored? Cost-controlled? Least-privilege? Recoverable? You'll score your own build against it.

**Responsible AI (end to end)** — safety (guardrails), privacy (PII filters), transparency (source citations), and cost accountability (budgets + token metrics) together. You've now touched every one.

---

## ⚠️ Reminder: How to Read Commands

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account number | `123456789012` |
| `<YOUR_EMAIL>` | The email address for budget alerts | `you@example.com` |
| `<YOUR_GUARDRAIL_ID>` | Your guardrail ID from Lab 12B | `abcd1234efgh` |

---

## Lab Steps

### Step 1: Set Up

**Windows (PowerShell):**
```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
cd ~\Desktop\workshop-lab-11a
```

**macOS / Linux:**
```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
cd ~/Desktop/workshop-lab-11a
```

---

### Step 2: Define the Budget

**Step 2a:** Create the budget definition. New file → 📋 copy and paste:

```json
{
    "BudgetName": "workshop-ai-monthly-budget",
    "BudgetLimit": {"Amount": "5", "Unit": "USD"},
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST"
}
```

Save as `budget.json` in your `workshop-lab-11a` folder.

**Step 2b:** Create the notification + subscriber definition. New file → 📋 copy and paste (**replace `<YOUR_EMAIL>`**):

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
            {"SubscriptionType": "EMAIL", "Address": "<YOUR_EMAIL>"}
        ]
    }
]
```

Save as `notifications.json`.

> **What does this configure?** A **$5/month** budget that emails you the moment your *actual* spend passes **80%** ($4). For a workshop that normally costs pennies, $5 is a comfortable ceiling that will still catch a runaway mistake early.

---

### Step 3: Create the Budget

**Step 3a:** Create the budget with its alert. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws budgets create-budget --account-id <YOUR_ACCOUNT_ID> --budget file://budget.json --notifications-with-subscribers file://notifications.json
```

**macOS / Linux:**
```bash
aws budgets create-budget --account-id <YOUR_ACCOUNT_ID> --budget file://budget.json --notifications-with-subscribers file://notifications.json
```

**✅ No output means success.**

> **💡 No `--region` here?** AWS Budgets is a *global* service (its home endpoint is in us-east-1). Unlike Lambda or Bedrock, you don't pass a region — the budget applies to your whole account.

**Step 3b:** Verify the budget exists. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws budgets describe-budget --account-id <YOUR_ACCOUNT_ID> --budget-name workshop-ai-monthly-budget --query "Budget.[BudgetName,BudgetLimit.Amount,BudgetLimit.Unit]" --output text
```

**macOS / Linux:**
```bash
aws budgets describe-budget --account-id <YOUR_ACCOUNT_ID> --budget-name workshop-ai-monthly-budget --query "Budget.[BudgetName,BudgetLimit.Amount,BudgetLimit.Unit]" --output text
```

**✅ You should see:** `workshop-ai-monthly-budget    5    USD`

**Step 3c: ✅ Console Checkpoint** — Open the **Billing** console → **Budgets**. You should see `workshop-ai-monthly-budget` with an 80% email alert. (You may also get a confirmation email — that's the alert subscription being set up.)

> **🎯 Your three cost defences are now complete:**
> | Layer | Where | Catches |
> |-------|-------|---------|
> | Input validation (500 chars) | In the Lambda (11C) | Expensive prompts, *before* any AI call |
> | Token spike alarm (>500 tokens) | CloudWatch (11C) | Per-request anomalies / injection |
> | $5 monthly budget (80% alert) | AWS Budgets (this lab) | Total account spend — the final backstop |

---

### Step 4: Full-Stack End-to-End Verification

Time to prove the *entire* system works together. Run each test, edit `payload.json`, save, and invoke:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-ai-chatbot-lab11 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json; Get-Content response.json
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-ai-chatbot-lab11 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json && cat response.json
```

| # | Send this message | Exercises | ✅ Expected result |
|---|-------------------|-----------|-------------------|
| 1 | `{"body": "{\"message\": \"What is the codename for the capstone chatbot project?\"}"}` | **RAG** (12A) | Answers "Project Northstar" with a `[project-northstar.txt]` citation |
| 2 | `{"body": "{\"message\": \"What is the capital of France?\"}"}` | **Grounding** (12A) | "I don't have that in my knowledge base." |
| 3 | `{"body": "{\"message\": \"Should I put my savings into Bitcoin?\"}"}` | **Guardrail — denied topic** (12B) | Blocked message |
| 4 | `{"body": "{\"message\": \"My card is 4111 1111 1111 1111, tell me about EC2.\"}"}` | **Guardrail — PII block** (12B) | Blocked (credit card) |
| 5 | `{"body": "{\"message\": \"$(a 600-character wall of text — reuse the long prompt from Lab 11C Step 3b)\"}"}` | **Input validation** (11C) | `400` — "message is too long" |
| 6 | `{"body": "{\"message\": \"Explain what an S3 bucket is for a beginner.\"}"}` | **Happy path** (all layers) | Short, grounded, on-topic answer |

> **⚠️ For test #5**, paste the full long prompt from **Lab 11C, Step 3b** (the "5000 words" block). It's over 500 characters, so input validation should reject it *before* Bedrock or the guardrail ever run — the cheapest possible rejection.

**Step 4b:** Confirm the whole system is observable. Open your dashboard and check the alarm:

```
aws cloudwatch describe-alarms --alarm-names ai-chatbot-token-spike --region us-east-1 --query "MetricAlarms[0].[AlarmName,StateValue]" --output text
```

**✅ You should see** `ai-chatbot-token-spike    OK`. Then open CloudWatch → Dashboards → `workshop-ai-chatbot-dashboard` and confirm invocations, latency, and tokens from your tests are showing up.

> **🎉 If all six tests behaved as expected, you have shipped a production-shaped AI application:** grounded, safe, monitored, cost-capped, least-privilege, and fully observable. Take a screenshot of your dashboard — it's portfolio material.

---

### Step 5: Production-Readiness Review

Score your build against the checklist a real team would use. Everything here you actually implemented across the track:

| Dimension | Control you built | Session |
|-----------|-------------------|---------|
| ✅ **Works** | Serverless chatbot on Lambda + API Gateway + Bedrock | 11A |
| ✅ **Controllable** | System prompt, temperature, maxTokens | 11B |
| ✅ **Stateful (context)** | Conversation history pattern | 11B |
| ✅ **Grounded** | RAG over an S3 knowledge base, with citations | 12A |
| ✅ **Honest** | Refuses to answer outside its knowledge ("I don't know") | 12A |
| ✅ **Safe** | Bedrock Guardrails: content, denied topics, PII, prompt-attack | 12B |
| ✅ **Cost-controlled** | Input validation + maxTokens + token alarm + account budget | 11C / 12C |
| ✅ **Observable** | Structured logs, token metric filter, dashboard, alarms | 11C |
| ✅ **Least-privilege** | Scoped IAM: Bedrock model, KB bucket, and guardrail only | 11A / 12A / 12B |

**What you'd add next in a real production system** (great to know for interviews and the exam):
- **Rate limiting per user** (API Gateway usage plans + API keys) to stop a single client flooding the bot
- **Persistent conversation storage** (DynamoDB) instead of hand-passed history
- **Managed retrieval at scale** (Amazon Bedrock Knowledge Bases + a vector store) in place of keyword search
- **Bedrock model-invocation logging** for a full audit trail of every prompt and response
- **CI/CD** (infrastructure as code with CloudFormation/CDK) instead of manual CLI deploys
- **A guardrail-intervention alarm** (a metric filter on your `guardrail_intervened` logs — same pattern as the token filter) to detect abuse spikes

> **🎯 The takeaway:** you didn't just call an AI model — you *operated* one responsibly. That operational maturity (grounding, safety, monitoring, cost, least-privilege) is precisely what separates an AI engineer from someone who can only build a demo.

---

### Step 6: AI Engineering Track Retrospective

Look back at the arc you completed:

- **Session 11A — Build:** stood up a live, serverless AI chatbot from nothing.
- **Session 11B — Control:** learned that *the prompt is the product* — personality, guardrails-by-instruction, and multi-turn memory.
- **Session 11C — Operate:** token budgets, input validation, metric filters, dashboards, and alarms — treating tokens like money.
- **Session 12A — Ground:** RAG gave the bot real knowledge and killed hallucinations, with source citations.
- **Session 12B — Secure:** Bedrock Guardrails enforced safety the model can't be talked out of.
- **Session 12C — Ship:** account-level cost governance, full end-to-end verification, and a production-readiness review.

**Where this points next:**
- **Certification:** you've now touched nearly every generative-AI domain on the **AWS Certified AI Practitioner** exam — foundation models, prompt engineering, RAG, responsible AI, guardrails, and AI cost/operations. This is a strong moment to book it.
- **Portfolio:** your live chatbot + dashboard screenshots + this architecture are a genuine talking point in interviews. Write up "how I built a grounded, guarded, monitored AI chatbot on AWS."
- **Deeper AWS AI:** natural next steps are Bedrock Knowledge Bases, Bedrock Agents, and Amazon Q — all of which build directly on what you now understand from first principles.

---

### The Complete Architecture

```
                          ┌──────────────────────────────┐
   User (browser/CLI)     │  API Gateway  (public HTTPS)  │
        │  message  ─────► │  GET = chat UI / POST = chat  │
        │                  └───────────────┬──────────────┘
        │                                  │ (AWS_PROXY)
        │                                  ▼
        │                  ┌──────────────────────────────┐
        │                  │        Lambda handler         │
        │                  │  1. Input validation (11C)    │
        │                  │  2. Retrieve KB chunks (12A) ◄─┼──── S3 knowledge base (12A)
        │                  │  3. Build grounded prompt     │
        │                  │  4. Converse + Guardrail (12B)┼──── Bedrock Guardrail (12B)
        │                  │  5. Structured JSON logs (11C)│
        │  ◄───── answer ──┤     + source citations        │
                           └───────┬───────────────┬───────┘
                                   │               │
                                   ▼               ▼
                      Bedrock Nova Micro     CloudWatch Logs ──► Metric filter ──► Dashboard + Token alarm (11C)
                         (11A)                                                     
                                   
   Account guardrails:  IAM least-privilege (all sessions)   +   AWS Budgets $5/mo alert (12C)
```

---

## What You Just Did

| What You Built | Why It Matters |
|---------------|----------------|
| $5 monthly cost budget with email alert | The outermost financial backstop for the whole account |
| Three-layer cost defence | Validate → cap/alarm → budget: no single point of failure |
| Full end-to-end verification | Proof that RAG, guardrails, validation, and monitoring work together |
| Production-readiness review | You can articulate what's done and what's next — interview gold |
| A complete, shippable AI product | Grounded, safe, monitored, cost-capped, least-privilege |

**Key takeaways:**
- **Budgets are the last line of defence** — set one on every account, always.
- **Defence in depth applies to cost, not just security** — layer independent controls.
- **Shipping = operating** — grounding + safety + monitoring + cost + least-privilege.
- **You can now prove your AI is responsible** — every control is testable and auditable.
- **This is real AI engineering** — the same patterns scale from a workshop bot to production.

---

## Cert Prep Callout

**Target Certification:** AWS Certified AI Practitioner

The AI Practitioner exam tests:
- **Cost management and governance** for AI workloads (budgets, token tracking, tagging)
- **Defence-in-depth** thinking across cost and security
- End-to-end **responsible AI**: safety, privacy, transparency, accountability
- Knowing when to use managed services (Bedrock Knowledge Bases, Agents) vs building your own
- Operational excellence for generative AI applications

**Sample question type:** "An organization wants to be alerted before its generative-AI experiment exceeds a monthly spending limit, regardless of any in-application controls. What should they configure?"
**Answer:** An AWS Budget with a cost threshold and notification — an account-level control that alerts on total spend independently of the application's own guardrails.

---

## Cleanup — Tear Down the Entire AI Stack (Sessions 11 + 12)

Run these to remove **everything** you created across the AI Engineering track. Replace the placeholders as usual.

**Step A — Budget:**
```
aws budgets delete-budget --account-id <YOUR_ACCOUNT_ID> --budget-name workshop-ai-monthly-budget
```

**Step B — Guardrail (12B):**
```
aws bedrock delete-guardrail --guardrail-identifier <YOUR_GUARDRAIL_ID> --region us-east-1
```

**Step C — S3 knowledge base (12A):**
```
aws s3 rm s3://workshop-ai-kb-<YOUR_ACCOUNT_ID> --recursive --region us-east-1
aws s3 rb s3://workshop-ai-kb-<YOUR_ACCOUNT_ID> --region us-east-1
```

**Step D — CloudWatch dashboard, alarm, and metric filter (11C):**
```
aws cloudwatch delete-dashboards --dashboard-names workshop-ai-chatbot-dashboard --region us-east-1
aws cloudwatch delete-alarms --alarm-names ai-chatbot-token-spike --region us-east-1
aws logs delete-metric-filter --log-group-name "/aws/lambda/workshop-ai-chatbot-lab11" --filter-name token-usage --region us-east-1
```

**Step E — API Gateway (11A) — only if you still have it:**
```
aws apigateway delete-rest-api --rest-api-id <YOUR_API_ID> --region us-east-1
```
> Find the ID with: `aws apigateway get-rest-apis --region us-east-1 --query "items[?name=='workshop-ai-chatbot'].id" --output text` (the REST API you named `workshop-ai-chatbot` in Lab 11A, Step 6a).

**Step F — Lambda + log group (11A):**
```
aws lambda delete-function --function-name workshop-ai-chatbot-lab11 --region us-east-1
aws logs delete-log-group --log-group-name /aws/lambda/workshop-ai-chatbot-lab11 --region us-east-1
```

**Step G — IAM role and all its inline policies (11A/12A/12B):**
```
aws iam delete-role-policy --role-name workshop-lab11-lambda-role --policy-name bedrock-invoke
aws iam delete-role-policy --role-name workshop-lab11-lambda-role --policy-name kb-s3-read
aws iam delete-role-policy --role-name workshop-lab11-lambda-role --policy-name guardrail-apply
aws iam detach-role-policy --role-name workshop-lab11-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name workshop-lab11-lambda-role
```

**Step H — Local project folder:**

**Windows (PowerShell):**
```powershell
cd ~\Desktop; Remove-Item -Recurse -Force workshop-lab-11a
```

**macOS / Linux:**
```bash
cd ~/Desktop && rm -rf workshop-lab-11a
```

**✅ Verify nothing is left** (each should return empty or "not found"):
```
aws lambda get-function --function-name workshop-ai-chatbot-lab11 --region us-east-1
aws s3 ls | grep workshop-ai-kb
aws bedrock list-guardrails --region us-east-1
```

> **💡 Why cleanup matters one last time:** unused resources can quietly accrue charges and clutter your account. A professional always leaves the account as clean as they found it. Your $5 budget would have caught anything you missed — but deleting is better than alerting.

---

## 🎓 Congratulations — You've Completed the AI Engineering Track!

You went from an empty AWS account to a **grounded, guarded, monitored, cost-governed, serverless AI application** — built and operated the way real teams do it. That's not a tutorial exercise; that's the actual job.

**Your next moves:**
1. **Book the AWS Certified AI Practitioner exam** — you've covered the generative-AI domains hands-on.
2. **Write up your build** for your portfolio (architecture + dashboard screenshots + what each layer defends against).
3. **Share a win** in the Microsoft Teams community — and help the next cohort through this track.

---

## Help

If you get stuck, post in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** you received
4. Your **operating system** (Windows / Mac / Linux)
