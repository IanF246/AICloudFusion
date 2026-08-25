# Lab 11B: Shape Your AI — Prompt Engineering & System Prompts

**Session:** 11 — AI Engineering  
**Track:** AI Engineering  
**Difficulty:** Intermediate  
**Estimated Time:** 40–50 minutes  
**Target Cert:** AWS Certified AI Practitioner

---

## Overview

In Lab 11A, your chatbot answered questions — but it had no personality, no guardrails, and no constraints. Ask it to write a 5,000-word essay and it will try (burning tokens). Ask it an off-topic question and it will answer anything.

In this lab you'll learn **prompt engineering** — the skill of controlling AI behaviour through carefully crafted instructions:

1. **Add a system prompt** that gives the bot a personality and constraints
2. **Compare responses** with and without system prompts
3. **Control output length** to manage cost (fewer tokens = less money)
4. **Implement conversation history** — multi-turn chat where the AI remembers context
5. **Monitor token usage** across different prompt strategies

**Why this matters:** Prompt engineering is the #1 skill employers look for in AI engineering roles. The difference between a useful AI application and a useless one is almost always the prompt design, not the model choice.

---

## Prerequisites

- ✅ Completed **Lab 11A** — your `workshop-ai-chatbot-lab11` function and role must exist
- ✅ AWS CLI authenticated

> **📌 If you cleaned up Lab 11A:** Follow Lab 11A Steps 3–5 to recreate the role and function.

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| AWS Lambda | Function execution | Always Free |
| Amazon Bedrock (Nova Micro) | AI inference | ~$0.01–0.03 for this lab |

**⚠️ Estimated cost for this lab: under $0.03**

---

## Concepts

**System Prompt** — invisible instructions that control how the AI behaves. The user sends their message, but the system prompt tells the AI "who you are" and "how to respond." Example: "You are a concise AWS tutor. Answer in 2 sentences max." The AI will then keep every response short, no matter what the user asks.

**Prompt Engineering** — the practice of designing prompts to get better, more consistent, and more cost-effective AI responses. It's not coding — it's crafting instructions. Small changes in phrasing can dramatically change output quality.

**Temperature** — controls randomness. 0 = always gives the same answer (deterministic). 1 = creative/varied. For factual Q&A, use 0.3–0.5. For creative tasks, use 0.7–1.0.

**maxTokens** — the hard limit on response length. If the AI hits this limit, it stops mid-sentence. Set it based on your use case: 50 for short answers, 200 for paragraphs, 500+ for long-form content. Lower = cheaper.

**Conversation History** — sending previous messages along with the new one so the AI has context. Without history, every message is independent (the AI doesn't "remember" anything). With history, you can have multi-turn conversations.

---

## ⚠️ Reminder: How to Read Commands

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account number | `123456789012` |

---

## Lab Steps

### Step 1: Set Up

**Step 1a:** Set your profile and navigate to your project folder.

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

**Step 1b:** Create the invoke payload file. Throughout this lab you'll test your chatbot from the CLI with `aws lambda invoke`, which needs a payload file to send. In the browser (via API Gateway), this envelope is built for you automatically; from the CLI, you supply it yourself.

Open your text editor → new file. 📋 Copy and paste **exactly** (the inner quotes must stay backslash-escaped):

```
{"body": "{\"message\": \"What is S3?\"}"}
```

Save as `payload.json` in your `workshop-lab-11a` folder.

> ⚠️ **Windows users:** in Notepad, change "Save as type" to **"All Files"** so it doesn't save as `payload.json.txt`.
>
> ⚠️ **Why the escaping?** The Lambda handler reads `event["body"]` and then runs `json.loads()` on it, so `body` must be a JSON *string*, not a nested object. That's why the inner `{...}` is wrapped in quotes with `\"` escapes. Getting this wrong is the most common cause of a `KeyError` or an empty/HTML response on invoke.

**Step 1c:** Verify the chatbot still works. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-ai-chatbot-lab11 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json; Get-Content response.json
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-ai-chatbot-lab11 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json && cat response.json
```

**✅ You should see** a response with an AI-generated answer. If the function doesn't exist, go back to Lab 11A (Steps 3-5).

> **💡 Troubleshooting Step 1c:**
> - `Unable to load paramfile file://payload.json` → the file isn't in your current folder. Run `pwd` and confirm you're in `workshop-lab-11a`.
> - Response looks like the HTML chat page instead of an answer → your `body` isn't a properly escaped JSON string. Re-check Step 1b.
> - `Invalid base64` → the `--cli-binary-format raw-in-base64-out` flag is missing from your command. It's included above — don't remove it.

---

### Step 2: Add a System Prompt — Give the Bot a Personality

You'll update the Lambda code to include a system prompt. The system prompt is sent with every request but invisible to the user.

**Step 2a:** Open `handler.py` in your text editor and **find the `bedrock.converse()` call** in the POST handler section (search for `bedrock.converse`). Replace the entire call block:

Find this:
```python
        response = bedrock.converse(
            modelId=MODEL_ID,
            messages=[
                {
                    "role": "user",
                    "content": [{"text": user_message}]
                }
            ],
            inferenceConfig={
                "maxTokens": 200,
                "temperature": 0.7
            }
        )
```

Replace it with:
```python
        response = bedrock.converse(
            modelId=MODEL_ID,
            system=[{"text": "You are a friendly AWS cloud tutor for beginners. Explain concepts in simple language with real-world analogies. Keep responses under 3 sentences. If asked something unrelated to cloud computing or AWS, politely redirect to cloud topics."}],
            messages=[
                {
                    "role": "user",
                    "content": [{"text": user_message}]
                }
            ],
            inferenceConfig={
                "maxTokens": 150,
                "temperature": 0.5
            }
        )
```

**Step 2b:** Save `handler.py`.

> **What changed?**
> - Added `system=[{"text": "..."}]` — this is the system prompt
> - Changed `maxTokens` from 200 to 150 (shorter responses = less cost)
> - Changed `temperature` from 0.7 to 0.5 (more consistent factual answers)

**Step 2c:** Re-deploy. 📋 Copy and paste:

> **⚠️ Verify you're in the `workshop-lab-11a` folder** (`pwd`).

**Windows (PowerShell):**
```powershell
Compress-Archive -Path handler.py -DestinationPath function.zip -Force
aws lambda update-function-code --function-name workshop-ai-chatbot-lab11 --zip-file fileb://function.zip --region us-east-1 --query "LastUpdateStatus" --output text
```

**macOS / Linux:**
```bash
zip function.zip handler.py
aws lambda update-function-code --function-name workshop-ai-chatbot-lab11 --zip-file fileb://function.zip --region us-east-1 --query "LastUpdateStatus" --output text
```

**✅ You should see:** `InProgress` or `Successful`

> **💡 Browser test too!** If you set up the API Gateway URL in Lab 11A, refresh that page in your browser. The chat UI will now use the updated system prompt. Try asking both AWS and off-topic questions through the browser interface.

---

### Step 3: Test the System Prompt

**Step 3a:** Ask an AWS question. Edit `payload.json`:

```json
{"body": "{\"message\": \"What is an S3 bucket?\"}"}
```

Save and invoke:

**Windows (PowerShell):**
```powershell
aws lambda invoke --function-name workshop-ai-chatbot-lab11 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json; Get-Content response.json
```

**macOS / Linux:**
```bash
aws lambda invoke --function-name workshop-ai-chatbot-lab11 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json && cat response.json
```

**✅ You should see** a short, beginner-friendly answer with an analogy (because the system prompt said "explain with analogies, under 3 sentences").

**Step 3b:** Now ask an off-topic question. Edit `payload.json`:

```json
{"body": "{\"message\": \"What is the best pizza topping?\"}"}
```

Save and invoke again.

**✅ You should see** the bot politely redirect to cloud topics (because the system prompt said "if unrelated to cloud, redirect").

> **🎯 The system prompt changed the bot's behaviour without changing any code logic.** It's still the same function, the same model — but the prompt controls what the AI does. This is prompt engineering.

**Step 3c:** Test the guardrails and capabilities. Try these specific prompts to verify the system prompt is working:

| What to Ask | What It Tests | Expected Behaviour |
|-------------|---------------|-------------------|
| "What is a VPC?" | On-topic AWS question | Short answer (≤3 sentences) with a real-world analogy |
| "Who won the World Cup in 2022?" | Off-topic question | Polite redirect back to cloud topics |
| "Explain EC2 vs Lambda in detail with 10 examples" | Attempts to override brevity | Still gives ≤3 sentences despite asking for "detail" (maxTokens enforces the limit) |

Try each one — edit `payload.json`, save, and invoke. Or use the browser chat UI if you set up the API Gateway in Lab 11A.

> **🎯 What you're proving:** The system prompt constrains the AI in three ways: topic (AWS only), length (3 sentences), and style (analogies + beginner-friendly). No code change needed — just words in the prompt.

**Step 3d:** Compare token usage between the two responses. Look at the `metadata` in each response:
- The AWS answer should use fewer output tokens (constrained to 3 sentences)
- The redirect should also be short

> **💡 Notice how `maxTokens: 150` and the "under 3 sentences" instruction work together.** The system prompt guides the AI to be brief, and maxTokens is the hard backstop in case it doesn't listen.

---

### Step 4: Experiment with Different Personalities

Now try a completely different system prompt to see how dramatically it changes the bot.

**Step 4a:** Open `handler.py` and change ONLY the system prompt text (the string inside `system=[{"text": "..."}]`) to:

```
You are a pirate AI assistant who explains AWS cloud concepts. Respond entirely in pirate speak. Use nautical metaphors. Keep responses under 50 words. End every response with 'Arrr!'
```

**Step 4b:** Re-deploy (same commands as Step 2c — zip and update-function-code).

**Step 4c:** Ask a question. Edit `payload.json`:

```json
{"body": "{\"message\": \"What is a Lambda function?\"}"}
```

Save and invoke.

**✅ You should see** a pirate-themed explanation of Lambda functions, ending with "Arrr!" — in under 50 words.

> **💡 Same model, same code, different prompt = completely different behaviour.** This is why prompt engineering is so powerful — and why it's a dedicated job role at AI companies.

**Step 4d:** Try one more personality. Change the system prompt to:

```
You are a strict technical interviewer. When asked about AWS, respond with a follow-up question that tests deeper understanding. Never give the answer directly — make the user think. Keep responses to one question only.
```

Re-deploy and invoke with:

```json
{"body": "{\"message\": \"What is EC2?\"}"}
```

**✅ You should see** the bot respond with a probing follow-up question instead of an answer.

> **🎯 Three system prompts, three completely different bots.** Same model, same infrastructure, same code — the prompt is the product.

---

### Step 5: Implement Conversation History (Multi-Turn Chat)

Right now, each invocation is independent — the AI doesn't remember previous messages. Let's add conversation history so you can have a real back-and-forth.

**Step 5a:** Change your system prompt back to the tutor version:

```
You are a friendly AWS cloud tutor for beginners. Explain concepts in simple language with real-world analogies. Keep responses under 3 sentences.
```

**Step 5b:** Edit `payload.json` to include a conversation history:

```json
{"body": "{\"message\": \"What about storage?\", \"history\": [{\"role\": \"user\", \"content\": \"What are the main AWS compute services?\"}, {\"role\": \"assistant\", \"content\": \"The main AWS compute services are EC2 (virtual servers), Lambda (serverless functions), and ECS/EKS (containers). Think of EC2 as renting an apartment, Lambda as hiring a task worker, and containers as food trucks.\"}]}"}
```

**Step 5c:** Update `handler.py` to use conversation history. Find the `bedrock.converse()` call and update the `messages` section:

Replace:
```python
            messages=[
                {
                    "role": "user",
                    "content": [{"text": user_message}]
                }
            ],
```

With:
```python
            messages=build_messages(body, user_message),
```

And add this function ABOVE `lambda_handler`:

```python
def build_messages(body, current_message):
    """Build message list with optional conversation history."""
    messages = []
    
    # Add history if provided
    history = body.get("history", [])
    for msg in history:
        messages.append({
            "role": msg["role"],
            "content": [{"text": msg["content"]}]
        })
    
    # Add current message
    messages.append({
        "role": "user",
        "content": [{"text": current_message}]
    })
    
    return messages
```

**Step 5d:** Save, re-deploy (zip + update-function-code), and invoke.

**✅ You should see** the AI answer about AWS storage, knowing from the history that you were previously asking about compute services. It has **context** — it knows this is a follow-up question.

> **💡 Conversation history = more tokens = higher cost.** Every previous message is sent again with each request. This is why production chatbots limit history length (e.g., last 10 messages only). Longer conversations burn more input tokens.

---

### Step 6: Compare Token Usage Across Strategies

Let's see the cost impact of different prompt approaches.

**Step 6a:** Reset `payload.json` to a simple message (no history):

```json
{"body": "{\"message\": \"What is S3?\"}"}
```

Invoke and note the `total_tokens` in the response.

**Step 6b:** Now invoke with conversation history (the payload from Step 5b). Note the `total_tokens`.

**✅ You should see** significantly more tokens used with history — because ALL the previous messages are re-sent as input tokens.

> **🎯 Token economics summary:**
> | Approach | Typical Input Tokens | Why |
> |----------|---------------------|-----|
> | No system prompt, short question | 5–15 | Just the user's message |
> | With system prompt | 30–60 | System prompt + user message |
> | With history (3 turns) | 100–200+ | System prompt + all previous messages + new message |
>
> **The lesson:** System prompts and history add up. In production, you'd set a budget: "max 500 input tokens per request" and trim history to stay within it.

---

## What You Just Did

| What You Built | Why It Matters |
|---------------|----------------|
| System prompt that controls AI behaviour | The bot follows rules without code changes |
| Multiple personalities from one codebase | Same infrastructure, different product — just by changing the prompt |
| Off-topic redirection | AI stays on-task (essential for customer-facing bots) |
| Conversation history | Multi-turn chat with context (how real chatbots work) |
| Token awareness across strategies | You understand the cost implications of each design choice |

**Key takeaways:**
- **The system prompt IS the product.** Change it, and you have a different AI application.
- **Temperature controls consistency.** Low for facts, high for creativity.
- **maxTokens is your cost backstop.** Never deploy without one.
- **History = context but also cost.** More turns = more input tokens.
- **Prompt engineering is a skill** — small wording changes → big behaviour changes.

---

## Cert Prep Callout

**Target Certification:** AWS Certified AI Practitioner

The AI Practitioner exam tests:
- Prompt engineering as a technique for controlling AI outputs
- System prompts, temperature, and token limits as configuration tools
- Cost optimization for AI applications (token management)
- Conversation history and context window management
- Responsible AI: constraining AI behaviour to intended use cases

**Sample question type:** "A company's AI chatbot occasionally responds to off-topic questions about politics. How should the engineering team prevent this?"  
**Answer:** Add a system prompt that instructs the model to stay on-topic and politely redirect off-topic questions. This is more cost-effective and maintainable than fine-tuning the model.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| Bot ignores the system prompt | System prompt wasn't included in the converse call | Verify `system=[{"text": "..."}]` is in the `bedrock.converse()` call, re-deploy |
| Bot still gives long responses despite "3 sentences" instruction | AI models don't always follow length instructions perfectly | Lower `maxTokens` as a hard limit (e.g., 100). The system prompt is a guideline; maxTokens is enforced. |
| `KeyError: 'history'` | Code tries to access history but payload doesn't have it | Use `body.get("history", [])` with a default empty list (the code in Step 5c does this) |
| Response says "context length exceeded" | History + system prompt + message is too many tokens | Reduce history length or shorten the system prompt |
| Higher cost than expected | Long history being sent with every request | Limit history to last 5–10 messages in the `build_messages` function |

---

## Cleanup

> **📌 If continuing to Lab 11C:** Keep the function and role.

**If stopping here:**

```
aws lambda delete-function --function-name workshop-ai-chatbot-lab11 --region us-east-1
aws logs delete-log-group --log-group-name /aws/lambda/workshop-ai-chatbot-lab11 --region us-east-1
aws iam delete-role-policy --role-name workshop-lab11-lambda-role --policy-name bedrock-invoke
aws iam detach-role-policy --role-name workshop-lab11-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name workshop-lab11-lambda-role
```

Delete the project folder:

**Windows (PowerShell):**
```powershell
cd ~\Desktop; Remove-Item -Recurse -Force workshop-lab-11a
```

**macOS / Linux:**
```bash
cd ~/Desktop && rm -rf workshop-lab-11a
```

---

## Help

If you get stuck, post in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** you received
4. Your **operating system** (Windows / Mac / Linux)
