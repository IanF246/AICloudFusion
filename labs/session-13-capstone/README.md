# The Capstone: Build Something You Own

**Session:** 13, Capstone
**Track:** All tracks (the finale)
**Format:** Guided discussion + self-directed project
**Estimated Time:** Your call. Most capstones land between 8 and 20 hours

---

## Read This First: This Is Not a Lab

Every other document in this repository hands you commands to copy and paste. This one doesn't, on purpose.

There are no steps to follow here, because the capstone isn't something we can write for you. It's the first thing in this workshop where **you** decide what gets built. This document exists to help you decide well.

**What the capstone is not:**

- ❌ **Redoing the labs.** Rebuilding Lab 12A with different documents in the S3 bucket is not a capstone. You already proved you can follow those instructions.
- ❌ **A test.** Nobody is grading you against a rubric. There's no pass mark.
- ❌ **A throwaway exercise.** If you delete it and never look at it again, it didn't work.

**What the capstone is:**

- ✅ **A demonstration of everything you've accumulated.** Twelve sessions of skills, combined into one thing you designed yourself.
- ✅ **Something you own.** A project you can keep improving, show to an employer, or genuinely charge someone money for.
- ✅ **Your decision.** The examples below are suggestions. Take one, bend it into a different shape, or ignore all of them and build your own idea. The choice is yours and it should be.

---

## Why This Matters More Than Any Single Lab

Here's the uncomfortable truth about hiring. Hundreds of people applying for the same junior cloud role have all done tutorials. Many of them have a certification. On paper you look similar to all of them.

What separates you is having built something that didn't come with instructions.

A certification proves you can recognise the right answer. A capstone proves you can make decisions when nobody tells you what the right answer is: which service to use, what to do when it breaks, what to cut when time runs out, how to know whether it actually works. That's the job. Not recalling that S3 is object storage, but choosing S3 over a database for a specific reason and being able to defend the choice.

There's a second reason people underestimate: **a capstone gives you something to talk about.** Interviews are conversations. When someone asks "tell me about something you've built," the candidate who talks for two minutes about a real decision they made (a bug that took a whole evening, a cost that surprised them, a design they'd change now) gets remembered. The candidate describing a tutorial does not.

And unlike a lab, a capstone keeps paying you back. You can extend it next month when you learn something new. You can put it in a portfolio. You can show it to a client. Some of you will build something people would actually pay for, and that's not a stretch goal. The examples below include ideas working freelancers charge real money for.

---

## The Shape of a Capstone: Problem → Solution → Demo

Whatever you build, it has three parts. Every example below follows this structure, your README should follow it, and it's the order you'll explain it in when someone asks.

**1. The problem statement.** Who has a problem, what it costs them, and why the existing options don't work for them. One paragraph, no technology in it at all. If you can't write this, you don't have a capstone yet: you have a pile of services.

**2. How your solution solves it.** The architecture, with every significant piece justified by the problem rather than by wanting to use the service. "Documents go in S3 because the access pattern is whole-file reads and it costs almost nothing at this volume" is a justification. "I used S3" is not.

**3. The demo.** Proof it works, shown rather than claimed. This is the part people skimp on, and it's the part that convinces anyone. Which means it needs its own discipline, below.

### The demo has to include causing a failure

Showing the happy path proves very little. Anyone can screenshot a form that worked.

A real demo also shows you **provoking the thing you built defences against, and the defences responding.** If you built monitoring, break something and show the alarm fire. If you built access control, try to get in without permission and show the denial and the log entry. If you built cost protection, exceed the limit deliberately and show it hold.

This is exactly how every lab from Session 6 onward taught you: see the bad state, apply the fix, verify the good state with the same measurement. The difference now is that it's your own system and nobody planted the fault for you.

There's a practical reason too. **If you've never seen your alarm fire, you don't know it works.** Monitoring that has never triggered is indistinguishable from monitoring that's broken. Provoking it once is the only way to be sure, and it's a genuinely impressive thing to show.

---

## You Haven't Lost Your Previous Work

Lab 12C had you tear down every AWS resource you built. That was deliberate and correct: it's what stops surprise bills. But some people read that as "everything I made is gone." It isn't.

The AWS resources are gone. **The valuable part survived**, and it's in your project folders and GitHub account right now:

| What you still have | Where it came from | Why it matters for the capstone |
|---|---|---|
| `handler.py` with RAG, validation, structured logging | Sessions 10–12 | A working Lambda pattern you can adapt to any purpose |
| OpenTofu configurations and modules | Session 7 | Recreate infrastructure in minutes instead of clicking through the console |
| GitHub Actions workflow with OIDC | Session 8 | A deployment pipeline you can point at anything |
| IAM policy documents | Sessions 3, 11, 12 | Least-privilege examples to copy and narrow |
| Metric filters, alarms, dashboard definitions | Sessions 9–10 | Observability you can bolt onto a new project |
| Guardrail and budget configurations | Session 12 | Safety and cost controls, ready to reuse |

**This is the real lesson of Infrastructure as Code.** Your infrastructure isn't the running resources, it's the code that creates them. You can rebuild your entire Session 12 stack from your own repository whenever you want. Build the capstone the same way, so it survives teardown and you can bring it back for an interview six months from now without paying to leave anything running.

---

## How to Think About Choosing a Project

This matters more than the examples, because it's the process you'll use for every project after this workshop.

### 1. Start with a problem and a person, not a service

The most common mistake is starting from technology: "I want to use DynamoDB." That leads to hollow projects, because you end up inventing a problem to justify the tool.

Start at the other end. Who has a problem, and what is it? A dance studio drowning in enquiry emails. A charity that can't find anything in five years of PDFs. An accountant emailing tax returns as attachments. Once you have a person and a problem, the services pick themselves, and you can explain every choice, which is what an interviewer is actually testing.

### 2. Name your career target and lean into it

Security, AI engineering, solutions architecture, platform work? Pick one and make your capstone deep in that direction. A security-focused capstone should go further into detection and response than any lab did. An AI capstone should have something to say about retrieval quality and cost per query.

Breadth shows you can operate. Depth in one area makes you credible for a specific job.

### 3. Demand breadth anyway

A capstone that's a single Lambda function isn't a capstone. Whatever you build should touch most of these, because the sweep is what proves twelve sessions rather than one:

| Discipline | Where you learned it | What it looks like in your capstone |
|---|---|---|
| Compute | Sessions 2, 7 | Lambda, or EC2 if you have a reason |
| Storage / data | Sessions 1, 2 | S3, DynamoDB, with a justification |
| Identity & least privilege | Sessions 3, 12 | Scoped IAM roles, no wildcard permissions |
| Infrastructure as Code | Session 7 | The whole thing defined in OpenTofu |
| CI/CD | Session 8 | Push to main, it deploys, via OIDC |
| Observability | Sessions 9, 10 | Structured logs, a metric that matters, an alarm you've seen fire |
| Cost governance | Sessions 1, 12 | A budget in place *before* you expose anything |
| Security posture | Sessions 3–5, 12 | Auth on endpoints, no public data leaks |
| AI (if relevant) | Sessions 11, 12 | Bedrock with grounding and guardrails |

You don't need all nine. You do need enough that it couldn't have been built by someone who'd only done one session.

### 4. Decide what proves it works before you build it

Before writing code, finish this sentence: **"I'll know this works when ___."**

A form submission arriving in DynamoDB. A response time that dropped from 900ms to 400ms. An alarm email landing within two minutes of a simulated failure. An unauthorised request returning 403 *and* showing up in your audit log. A cost-per-request number you can state out loud.

Projects without a defined success signal drift forever, because "done" never arrives. Projects with one finish. And this sentence becomes your demo.

### 5. Scope for something you can demonstrate

Ambition is good; unfinished is not. A small thing that genuinely works end to end, with monitoring and a pipeline, beats an ambitious half-built platform every time. You cannot demo a plan.

If you're unsure, cut features and keep quality. Depth over feature count. You can add the next thing after the workshop, and that's the point of owning it.

### 6. Plan cost and teardown from day one

Set a budget before you deploy anything. Know what each piece costs. Know how you'll tear it down, and make sure your IaC can bring it back. If you can't afford to leave it running, that's normal: build it so it redeploys in one command, record your demo, then destroy it.

### 7. Work out the two-minute story

Problem, who for, architecture shape, one decision you made and why, one thing that went wrong and what you did. Two minutes, no notes. If you can't tell that story, the project isn't finished no matter how much of it works.

---

## Three Example Capstones

These exist to show you the *thinking*, not to be assignments. Read them for how the problem drives the architecture and how the demo proves it, then go build whatever you want.

Each is deliberately something you could keep improving, put in front of a real user, and in some cases charge for.

---

### Example 1: A Grounded Document Assistant

**Leans toward:** AI engineering

**The problem.** A small organisation (a law office, a charity, a school, a plumbing company) has years of documents nobody can search. Policies, manuals, contracts, past quotes. Staff either can't find things or ask the same colleague the same question every week, and that colleague is the bottleneck for the whole office. The tools that solve this properly are priced for enterprises and cost more per month than this organisation spends on software all year. So they do nothing, and keep losing an hour a day to it.

**How your solution solves it.** Their documents live in S3, because the access pattern is whole-file reads and storage costs pennies at this volume. A Lambda retrieves the passages relevant to a question and injects them into a Bedrock prompt, instructed to answer only from that context and cite the file. That is what stops it inventing a policy that doesn't exist, the one failure that would destroy their trust in it immediately. Guardrails enforce safety independently of the prompt, so it can't be talked into going off-topic. The interface is authenticated, because these are confidential documents and a public URL would be both a data leak and someone else's free Bedrock bill. Token spend is logged per query so the running cost is a known number rather than a surprise. Everything is in OpenTofu and deployed by pipeline, so it can be rebuilt or handed over.

**The demo.** Ask it a question that's only answerable from their documents, and show the citation pointing at the right file. Then ask something outside the documents and show it saying so instead of guessing. Then provoke the defences: attempt to reach it without authenticating and show the refusal; try a prompt-injection attempt and show the guardrail hold; push a deliberately enormous request and show input validation reject it before it reaches the model. Finish on the token-spend dashboard with the cost per query on screen.

**Where the depth lives.** Retrieval quality. Lab 12A used keyword overlap, which is fine for a demo and weak on real documents. Improve it, and *measure* that you improved it. Write twenty realistic questions, record how many are answered correctly before and after, show the difference. That measurement is worth more than any feature you could add, because almost no junior candidate has ever measured an AI system's quality.

Cost per query is the second axis. State it as a number. A business will ask.

**Draws on:** Sessions 11, 12 (Bedrock, RAG, guardrails), 2 (S3), 3 (IAM), 7 (IaC), 8 (CI/CD), 9–10 (observability), 12C (budget).

**How you'd extend it later.** Multiple document collections. PDF support. A feedback button that logs bad answers so you can improve retrieval with real data. Multi-tenancy, if you want to sell it to more than one client.

---

### Example 2: A Secure Client File Portal

**Leans toward:** Security engineering

**The problem.** Accountants, solicitors, recruiters and clinics send sensitive documents to clients by email attachment, every day. Tax returns, medical forms, contracts, ID scans. Everyone involved knows it's wrong: the attachment sits in two mailboxes forever, gets forwarded, and there's no record of who opened it. If a client asks "who has seen my file?", there's no answer. The secure alternatives are enterprise products with per-seat pricing that a four-person firm won't pay, so they keep using email and hope.

**How your solution solves it.** The business signs in to an authenticated dashboard and uploads a file for a named client. The client receives a link that works for a limited time and then stops working, which removes the "forever in a mailbox" problem at its root. Every upload, link generation and download is written to an audit trail, so "who opened my file, and when" becomes a question with an answer. That audit trail *is* the product, as much as the file transfer is. Files are encrypted at rest, IAM is scoped so the Lambda can touch only this bucket, and the owner gets alerted on anything unusual, like repeated failed access attempts. A budget is in place before anything is exposed, and the whole stack is in OpenTofu behind a pipeline.

**The demo.** This is where the security work becomes visible, and it's why this project is worth building over a monitoring tool pointed at somebody else's infrastructure: **you built the thing you're defending, so you can attack it and prove the defences respond.**

Upload a file as the business, collect it as the client, then show the audit trail listing both events. Now provoke it: request the file without authenticating and show the 403. Wait for the link to expire and show it refuse. Try to reach a different client's file by editing the path and show that denied *and* recorded. Hammer it with repeated failed attempts and show the alert arrive in your inbox, with the detection time measured. Then show the audit trail again, now containing every one of those attempts.

That final screen is the whole capstone in one image: you tried to break your own system, it held, and it told you.

**Where the depth lives.** Whether an unauthorised attempt is both *blocked* and *detected*, since those are two different properties and plenty of real systems have only the first. Measure the detection time. Then ask the question a real client would: "can you show me everyone who has opened this document?" If your audit trail can answer that completely, you've built something a firm would genuinely buy.

**Draws on:** Sessions 3 (IAM, least privilege), 4–5 (detection and automated response), 2 (S3, presigned access), 6 (reliability), 9 (alarms, structured logs), 7 (IaC), 8 (CI/CD), 1 and 12C (budget), plus authentication.

**How you'd extend it later.** Client-side upload so customers can send files in too. Automatic expiry and deletion after a retention period. Email notifications on download. A branded portal per client, which is the point it becomes a product rather than a project.

---

### Example 3: A Serverless Intake and Booking Platform

**Leans toward:** Solutions architecture, platform and full-stack cloud

**The problem.** A small service business (a barber, a tutor, a mobile mechanic, a photographer) takes bookings and enquiries through Instagram DMs, WhatsApp and text messages. Requests get buried, double-booked, or missed entirely while they're with a customer. There's no record of who asked for what, no way to follow up on a quote from three weeks ago, and every missed message is money that went to a competitor instead. They're paying nothing for software and it's costing them real income.

**How your solution solves it.** A public booking or enquiry form gives every request one front door instead of four. Submissions are written to durable storage the moment they arrive, before anything else happens, because a lost booking is the one failure that makes the whole thing worse than the DMs it replaced. The owner is notified when something comes in, and reviews everything in an authenticated dashboard, authenticated because it holds customer names and contact details, and leaking those would be a genuine breach rather than an inconvenience. Optionally a Bedrock call summarises long rambling enquiries into a line the owner can read between appointments. Infrastructure in OpenTofu, deployed by pipeline, monitored, with a budget.

**The demo.** Submit an enquiry as a customer, show it appear in the owner's dashboard, show the notification arrive. Then attack the reliability claim: deliberately break the notification path and submit again. The notification fails, and the submission is still there, retrievable, exactly as promised. That's the difference between claiming durability and demonstrating it. Show the alarm that fired when the notification path broke, so the owner would actually know. Then show the dashboard refusing to load for an unauthenticated visitor.

**Where the depth lives.** Reliability and data protection. What happens to in-flight data when a component fails? Prove it rather than assert it. This is the Session 6 feedback loop applied to your own product, and it's the kind of thinking that separates an architect from someone who wires services together.

The other axis is handling real customer data properly: who can read it, how access is controlled, and what you'd do if someone asked you to delete their record. Have an answer.

**Draws on:** Sessions 2 (serverless pipeline), 1 (S3, budget), 3 (IAM), 6 (reliability), 7 (IaC), 8 (CI/CD), 9 (monitoring and alarms), 11–12 (optional AI), plus authentication.

**How you'd extend it later.** Calendar availability. Email or SMS confirmations. Deposit payments. A second client with different branding, the moment it stops being a project and starts being a product.

---

## Or Build Your Own Idea

Genuinely, please do if you have one. Something you actually care about gets finished, and something from your own life will be far more memorable in an interview than any example here.

Test your idea against these. If you can answer all of them, it's a good capstone:

- [ ] Can I write the problem statement in a paragraph, with no technology in it?
- [ ] Can I justify each major service by pointing at the problem?
- [ ] Can I describe a demo that shows it working *and* shows a defence holding when I attack it?
- [ ] Does it touch at least five of the nine disciplines in the breadth table?
- [ ] Is there something in it I don't yet know how to do? *(If not, it's too easy to teach you anything.)*
- [ ] Can I finish something demonstrable in the time I have?
- [ ] Does it push deeper toward the job I want?
- [ ] Will I still find this interesting in three weeks?
- [ ] Could I imagine someone paying for it, even a small amount?

That last one isn't about starting a business. It's a sharp test of whether you solved a real problem or invented one to fit the tools.

---

## What "Finished" Looks Like

Not a grading rubric, but a description of a capstone that works as a portfolio asset rather than a folder on your laptop. Note that it's organised as problem, solution, demo.

**A README that states the problem first.** What problem, for whom, and why it mattered, before any mention of AWS. Assume the reader is a hiring manager giving it ninety seconds.

**The architecture, with justifications.** A diagram, even hand-drawn and photographed, plus a few lines on why the significant choices were made. The diagram proves you understand your own system and it's what you'll point at when you tell the story.

**The demo, recorded.** A short screen recording covering the happy path *and* at least one defence holding under attack. **Record this before you tear anything down**, because people forget, then have nothing to show.

**Your success measurement, as a number.** Response time, cost per request, detection time, questions answered correctly. Whatever you committed to in step 4.

**The repository, not the running stack.** Everything in Git, infrastructure in OpenTofu so it can be recreated. This is what lets you destroy the resources and still own the project.

**A short honest reflection.** What broke, what you'd design differently, what's next. Interviewers trust candidates who can critique their own work far more than ones presenting it as flawless.

---

## Review Your Own Work Against the Six Pillars

From Session 6 onward, every lab was built around the Well-Architected pillars. Now apply them to something you designed yourself, the closest thing in this workshop to real architectural work.

Once your capstone runs, review it honestly:

| Pillar | Ask yourself |
|---|---|
| **Operational Excellence** | If this broke at 3am, would I know? Could I deploy a fix without manual steps? |
| **Security** | Is anything public that shouldn't be? Does every permission have a reason? Where is sensitive data and who can reach it? |
| **Reliability** | What's the single point of failure? What happens to in-flight data when a component fails? |
| **Performance Efficiency** | What's the slowest part, and do I know that from measurement or assumption? |
| **Cost Optimisation** | What does this cost per month and per request? What's the most expensive part, and is it worth it? |
| **Sustainability** | Am I storing data forever that nobody needs? Am I over-provisioned? |

Pick the weakest answer. Fix it. Measure before and after. Put it in your README. That loop of audit, fix, prove is what senior engineers do all day, and showing it puts you ahead of most junior candidates.

---

## Rules That Keep This Safe and Free

Non-negotiable, because this is the point in the workshop where you build without instructions and it's easy to get stung.

**Set a budget before you deploy anything.** Not after. Lab 1B showed you how.

**Put authentication on anything internet-facing.** A public URL that calls Bedrock is an invitation for someone else to spend your money. If it's public it needs auth, rate limiting, or both. An unauthenticated public endpoint in front of a paid API is the most expensive mistake available to you here.

**Never commit secrets.** No access keys, no API keys, not even in a commit you later removed, because Git remembers. Use OIDC for deployment as in Lab 8A. Check your history before making a repository public.

**Use fake data while building.** Especially for the file portal and booking platform. Don't collect real people's details for a practice project.

**Tear it down when you've recorded your demo.** Then rely on your IaC to bring it back. Check afterwards that things are actually gone. The usual culprits are versioned S3 buckets, running EC2 instances, and anything with a free trial that quietly starts charging.

---

## Talking About It Afterwards

**The resume line.** One sentence, concrete, with the sweep visible:

> "Designed and shipped a secure client file portal on AWS: authenticated access with time-limited links and a full audit trail, least-privilege IAM, infrastructure defined in OpenTofu, deployed via GitHub Actions with OIDC, and monitored with CloudWatch alarms on unauthorised access attempts."

**The two-minute story.** Problem, who for, architecture shape, one interesting decision, one thing that went wrong and how you fixed it. Practise it out loud. It will come up in every interview you have.

**What makes it land.** Not the list of services, but the decisions. "I chose S3 over a database because the access pattern was whole-document reads and it cut the cost to almost nothing" is worth more than naming ten services. Interviewers are listening for reasoning.

---

## Where to Go for Reference

Your own previous labs are the best documentation you have, and you've already run every command in them:

| If you need to... | Go back to |
|---|---|
| Set up a budget and alerts | [Lab 1B](../session-01-cloud-concepts/lab-1b-cost-budget-sns-alert.md) |
| Build a serverless pipeline | [Lab 2C](../session-02-core-services/lab-2c-serverless-file-pipeline.md) |
| Write least-privilege IAM policies | [Lab 3A](../session-03-cloud-security/lab-3a-iam-least-privilege.md) |
| Automate a security response | [Lab 5C](../session-05-incident-response/lab-5c-automated-remediation.md) |
| Apply the Well-Architected pillars | [Lab 6A](../session-06-well-architected/lab-6a-s3-security-reliability.md) · [Lab 6B](../session-06-well-architected/lab-6b-lambda-performance-operations.md) · [Lab 6C](../session-06-well-architected/lab-6c-cost-sustainability.md) |
| Define infrastructure in OpenTofu | [Lab 7B](../session-07-iac-fundamentals/lab-7b-lambda-with-iac.md) · [Lab 7C](../session-07-iac-fundamentals/lab-7c-event-driven-modules.md) |
| Set up a deployment pipeline with OIDC | [Lab 8A](../session-08-cicd/lab-8a-github-oidc.md) · [Lab 8B](../session-08-cicd/lab-8b-pipeline-plan-apply.md) |
| Add alarms and diagnose from logs | [Lab 9A](../session-09-monitoring/lab-9a-cloudwatch-alarms.md) · [Lab 9B](../session-09-monitoring/lab-9b-diagnose-with-logs.md) |
| Call Bedrock from Lambda | [Lab 11A](../session-11-ai-engineering/lab-11a-bedrock-chatbot.md) |
| Add RAG grounding | [Lab 12A](../session-12-rag-guardrails-capstone/lab-12a-rag-knowledge-base.md) |
| Add guardrails | [Lab 12B](../session-12-rag-guardrails-capstone/lab-12b-bedrock-guardrails.md) |
| Add cost governance | [Lab 12C](../session-12-rag-guardrails-capstone/lab-12c-capstone-cost-governance.md) |

The [Glossary](../GLOSSARY.md) covers every service, concept, and command from the whole workshop.

---

## Help

The capstone is self-directed, but you are not on your own. Post in the **Lab Help** channel on Microsoft Teams:

- **Choosing an idea?** Tell us the problem you're considering and who has it. We'll help you scope it.
- **Stuck on something technical?** The command you ran, the full error, and your operating system, same as always.
- **Want a design opinion?** Post your architecture diagram. "Is this a sensible way to do this?" is a great question, and asking it is a professional skill.

Share what you build in the channel, even unfinished. Someone else is stuck on the thing you just solved.
