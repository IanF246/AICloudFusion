# Session 12 — AI Engineering Capstone

**Track:** AI Engineering (the finale)
**Target Cert:** AWS Certified AI Practitioner
**Builds directly on:** Session 11 (the `workshop-ai-chatbot-lab11` Lambda you built in Labs 11A–11C)

---

## What this session is

Session 11 got your serverless AI chatbot **built (11A)**, **controllable (11B)**, and **operable (11C)**. Session 12 concludes the AI Engineering track by turning that chatbot into a **production-shaped product**: grounded in your own data, made safe with enforced guardrails, and capped with account-level cost governance — then fully verified and torn down.

You keep extending the *same* function and role from Session 11, so make sure that stack still exists (or rebuild it from Lab 11A Steps 3–5) before you start.

---

## The three labs

| Lab | Title | Difficulty | You'll add |
|-----|-------|-----------|-----------|
| **12A** | [Give Your Chatbot a Brain — RAG](lab-12a-rag-knowledge-base.md) | Beginner → Intermediate | An S3 knowledge base + retrieval so the bot answers from *your* documents, cites sources, and stops hallucinating |
| **12B** | [Responsible AI — Bedrock Guardrails](lab-12b-bedrock-guardrails.md) | Intermediate | Enforced content filters, a denied topic, PII protection, and prompt-attack defence — safety the model can't be talked out of |
| **12C** | [Capstone — Ship It: Cost Governance & the Full Stack](lab-12c-capstone-cost-governance.md) | Advanced | An account-level AWS Budget, a full end-to-end verification of the whole system, a production-readiness review, and complete teardown |

> **Recommended order:** 12A → 12B → 12C, keeping every resource in place until the Lab 12C cleanup. Each lab builds on the last.

---

## By the end of Session 12 you will have

- A **grounded** chatbot (RAG over S3) that cites its sources and admits when it doesn't know
- **Enforced guardrails** (content, denied topics, PII, jailbreak defence) independent of the prompt
- **Three layers of cost defence**: input validation → token alarm → account budget
- A complete, **observable** system (structured logs, metric filters, dashboard, alarms)
- **Least-privilege IAM** scoped to exactly what each part needs
- A hands-on grasp of nearly every generative-AI domain on the **AWS Certified AI Practitioner** exam

---

## Cost

Every lab uses Always Free resources plus a few cents of Bedrock/Guardrails usage. **Total for the whole session: well under $0.20**, and Lab 12C removes everything so there's no ongoing charge.

---

## Help

Stuck on any lab? Post in the **Lab Help** channel on Microsoft Teams with: the step number, the exact command you ran, the full error message, and your operating system.
