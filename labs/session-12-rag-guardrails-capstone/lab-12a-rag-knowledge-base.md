# Lab 12A: Give Your Chatbot a Brain — Retrieval-Augmented Generation (RAG)

**Session:** 12 — AI Engineering (Capstone)    
**Track:** AI Engineering    
**Difficulty:** Beginner → Intermediate    
**Estimated Time:** 45–55 minutes    
**Target Cert:** AWS Certified AI Practitioner    

---

## Overview

By the end of Session 11 your chatbot could talk, follow a system prompt, remember a conversation, and stay on budget. But it has one big weakness: **it only knows what the model was trained on.** Ask it about *your* company, *your* documents, or anything newer than its training cut-off and it will either say "I don't know" — or worse, confidently **make something up** (this is called a *hallucination*).

In this lab you'll fix that with **Retrieval-Augmented Generation (RAG)** — the single most important pattern in real-world AI engineering. Instead of hoping the model already knows the answer, you'll:

1. **Store your own knowledge** as documents in an S3 bucket
2. **Retrieve** the most relevant pieces at question time
3. **Inject** them into the prompt as context
4. **Ground** the AI so it answers *only* from your documents — and cites its sources
5. **Prove** it works by asking a question that is *only* answerable from your knowledge base

**Why this matters:** "RAG is how nearly every production AI assistant answers questions about private or up-to-date data — without retraining the model." Customer-support bots, internal wikis, documentation assistants, legal and medical tools: almost all of them are RAG under the hood. Knowing this pattern is the difference between a demo and a real product.

---

## Prerequisites

- ✅ Completed **Lab 11A, 11B, and 11C** — the `workshop-ai-chatbot-lab11` function and `workshop-lab11-lambda-role` role must exist, with the system prompt (11B) and input validation (11C) already in your `handler.py`
- ✅ AWS CLI authenticated
- ✅ Your `~/Desktop/workshop-lab-11a` project folder with `handler.py` and `payload.json`

> **📌 If you cleaned up Session 11:** Follow Lab 11A Steps 3–5 to recreate the role and function, then re-apply the Lab 11B system prompt before starting here.

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| Amazon S3 | Stores your knowledge documents | Always Free (5 GB) |
| AWS Lambda | Function execution | Always Free |
| Amazon Bedrock (Nova Micro) | AI inference (slightly more input tokens due to injected context) | ~$0.01–0.03 for this lab |

**⚠️ Estimated cost for this lab: under $0.05**

---

## Concepts

**RAG (Retrieval-Augmented Generation)** — instead of asking the model to answer from memory, you first *retrieve* relevant text from a knowledge source and *augment* the prompt with it. The model then *generates* an answer grounded in that text. Retrieve → Augment → Generate.

**Grounding** — forcing the AI to base its answer on provided facts rather than its own training. A grounded bot can cite where each fact came from, and can honestly say "I don't know" when the answer isn't in the source.

**Hallucination** — when an AI confidently states something false. RAG dramatically reduces hallucination because the model is told: "answer *only* from this context."

**Chunking** — splitting documents into small passages so you can retrieve just the relevant piece instead of stuffing an entire document (and its token cost) into every request.

**Retrieval** — finding the chunks most relevant to the user's question. Production systems use *vector embeddings* (semantic similarity). In this lab we use a simpler **keyword-overlap** score so you can see the mechanism clearly — no vector database required.

**RAG vs Fine-Tuning** — two ways to give a model new knowledge. Fine-tuning re-trains the model (expensive, slow, hard to update). RAG just changes what you send in the prompt (cheap, instant, easy to update — edit a file in S3). For facts that change, **RAG almost always wins.**

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

---

### Step 2: Create a Knowledge Base in S3

Your chatbot needs something to *know*. You'll create three small documents — including one with a fact the AI could **not** possibly know unless it reads your file. That's how you'll prove RAG is working later.

**Step 2a:** Create a folder for your knowledge documents. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
mkdir knowledge-base
```

**macOS / Linux:**
```bash
mkdir knowledge-base
```

**Step 2b:** Create the first document. Open your text editor → new file. 📋 Copy and paste **exactly**:

```
AI Cloud Fusion Workshop — Frequently Asked Questions

The AI Cloud Fusion workshop meets every Wednesday afternoon at 2:00 PM Atlantic Standard Time.
Each session includes a lecture, three hands-on labs (Beginner, Intermediate, Advanced),
and a career preparation segment.

Recorded sessions focus on lecture and Beginner lab and are then posted in the Microsoft Teams channel within 24 hours.

The recommended certification path is: AWS Solutions Architect - Associate first,
then AWS Certified AI Practitioner, then a specialty certification of your choice.
```

Save it as `workshop-faq.txt` inside the `knowledge-base` folder.

> ⚠️ **Windows users:** in Notepad, change "Save as type" to **"All Files"** so it doesn't save as `workshop-faq.txt.txt`.

**Step 2c:** Create the second document. New file. 📋 Copy and paste **exactly**:

```
AWS Mini-Glossary for Beginners

S3 (Simple Storage Service): object storage for files. Think of it as an infinite hard drive in the cloud.
Lambda: serverless compute. Your code runs on demand without you managing any servers.
Bedrock: a managed service for calling foundation models (AI models) through one API.
CloudWatch: AWS's monitoring service for logs, metrics, dashboards, and alarms.
IAM: Identity and Access Management. Controls who and what can do what in your account.
API Gateway: creates public HTTPS endpoints that trigger your Lambda functions.
```

Save it as `aws-glossary.txt` inside the `knowledge-base` folder.

**Step 2d:** Create the third document — this one contains a made-up fact the model has never seen. New file. 📋 Copy and paste **exactly**:

```
AI Cloud Fusion — Internal Project Codes

The internal codename for the workshop's capstone chatbot project is "Project Northstar".
The support escalation password for facilitators is: FUSION-2024-CLOUD.
The workshop's cloud budget alert email is sent whenever spend exceeds 80 percent of the monthly limit.
```

Save it as `project-northstar.txt` inside the `knowledge-base` folder.

> **💡 Why the made-up fact?** No AI model on earth was trained on "Project Northstar" — it exists *only* in your file. So if the bot can tell you the codename later, you'll know for certain the answer came from **retrieval**, not the model's memory. That's your proof.

**Step 2e:** Create a globally-unique S3 bucket (bucket names must be unique across all of AWS, so we add your account ID). 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3 mb s3://workshop-ai-kb-<YOUR_ACCOUNT_ID> --region us-east-1
```

**macOS / Linux:**
```bash
aws s3 mb s3://workshop-ai-kb-<YOUR_ACCOUNT_ID> --region us-east-1
```

**✅ You should see:** `make_bucket: workshop-ai-kb-<YOUR_ACCOUNT_ID>`

**Step 2f:** Upload your three documents. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3 cp knowledge-base s3://workshop-ai-kb-<YOUR_ACCOUNT_ID>/ --recursive --region us-east-1
```

**macOS / Linux:**
```bash
aws s3 cp knowledge-base s3://workshop-ai-kb-<YOUR_ACCOUNT_ID>/ --recursive --region us-east-1
```

**✅ You should see** three `upload:` lines, one per file.

**Step 2g: ✅ Console Checkpoint** — Open the S3 console → your `workshop-ai-kb-...` bucket. You should see `workshop-faq.txt`, `aws-glossary.txt`, and `project-northstar.txt`.

---

### Step 3: Let Lambda Read the Knowledge Base (IAM)

Your function can call Bedrock, but it can't read S3 yet. You'll add a small, tightly-scoped read permission.

**Step 3a:** Create the policy file. Open your text editor → new file. 📋 Copy and paste (**replace `<YOUR_ACCOUNT_ID>` in both places**):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": ["s3:GetObject", "s3:ListBucket"],
            "Resource": [
                "arn:aws:s3:::workshop-ai-kb-<YOUR_ACCOUNT_ID>",
                "arn:aws:s3:::workshop-ai-kb-<YOUR_ACCOUNT_ID>/*"
            ]
        }
    ]
}
```

Save as `kb-s3-policy.json` in your `workshop-lab-11a` folder.

> ⚠️ **Windows users:** in Notepad, change "Save as type" to **"All Files"** so it doesn't save as `kb-s3-policy.json.txt`.

**Step 3b:** Attach the policy to the role from the file. 📋 Copy and paste (same command on every OS):

```
aws iam put-role-policy --role-name workshop-lab11-lambda-role --policy-name kb-s3-read --policy-document file://kb-s3-policy.json --region us-east-1
```

**✅ No output means success.**

> **💡 Why a file instead of inline JSON?** On Windows PowerShell, quotes inside an inline `--policy-document "{...}"` get mangled before the AWS CLI sees them (you'll get `Unknown options: Version...`). Passing the JSON as a `file://` reference — exactly how Lab 11A created its IAM policies — sidesteps the quoting problem entirely and works identically on Windows, macOS, and Linux.

> **What did you just grant?** Only `s3:GetObject` (read a file) and `s3:ListBucket` (list files) — and **only** on your KB bucket. This is *least privilege*: the function can read its knowledge base and nothing else. The AI Practitioner exam loves this principle.

---

### Step 4: Add Retrieval to Your Lambda

Now you'll teach `handler.py` to load the knowledge base, find the relevant chunks for each question, and inject them into the prompt.

> **💡 Back up your Lab 11C handler first.** Before editing, save a copy of your current (end-of-11C) `handler.py` so you can restore it later — the Cleanup section's full reset uses this backup. Run once from your `workshop-lab-11a` folder:
>
> **Windows (PowerShell):** `Copy-Item handler.py handler-11c-backup.py`
> **macOS / Linux:** `cp handler.py handler-11c-backup.py`
>
> If your edits go wrong at any point, you can also just paste the [complete `handler.py` reference](#-complete-handlerpy-reference-end-of-step-4) below and re-deploy.

**Step 4a:** Open `handler.py`. Near the **top of the file**, find where the other clients/constants are defined (e.g. `MODEL_ID = ...` and the `bedrock = boto3.client(...)` line). **Add these lines** just below them:

```python
import os
import re

s3 = boto3.client("s3", region_name="us-east-1")
KB_BUCKET = os.environ.get("KB_BUCKET", "")

# Cache the knowledge base across warm invocations (loaded once per cold start)
_KB_CACHE = None
```

> **💡 Cold starts & caching.** `_KB_CACHE` lives *outside* the handler function, so it persists between invocations while the Lambda stays warm. The KB is downloaded from S3 **once** (on the first, "cold" request) and reused after that — fewer S3 calls, faster responses. This is a real serverless optimisation pattern.

**Step 4b:** Add the two helper functions. Paste them **above** your `lambda_handler` function (a good spot is right after the `build_messages` function you added in Lab 11B):

```python
def load_knowledge_base():
    """Load and cache all KB chunks from S3 (runs once per cold start)."""
    global _KB_CACHE
    if _KB_CACHE is not None:
        return _KB_CACHE

    chunks = []
    if KB_BUCKET:
        listing = s3.list_objects_v2(Bucket=KB_BUCKET)
        for obj in listing.get("Contents", []):
            key = obj["Key"]
            text = s3.get_object(Bucket=KB_BUCKET, Key=key)["Body"].read().decode("utf-8")
            # Split each document into paragraph-sized chunks (blank line = new chunk)
            for para in re.split(r"\n\s*\n", text):
                para = para.strip()
                if para:
                    chunks.append({"source": key, "text": para})

    _KB_CACHE = chunks
    return chunks


def retrieve_context(query, top_k=3):
    """Return the top_k KB chunks most relevant to the query (keyword overlap)."""
    chunks = load_knowledge_base()
    query_words = set(re.findall(r"[a-z0-9]+", query.lower()))

    scored = []
    for chunk in chunks:
        chunk_words = set(re.findall(r"[a-z0-9]+", chunk["text"].lower()))
        overlap = len(query_words & chunk_words)
        if overlap > 0:
            scored.append((overlap, chunk))

    scored.sort(key=lambda item: item[0], reverse=True)
    return [chunk for _score, chunk in scored[:top_k]]
```

> **What's the retrieval doing?** It scores every chunk by *how many words it shares with the question* and returns the best matches. Simple, but it demonstrates the exact RAG mechanism. Production systems swap this keyword score for **semantic embeddings** (which match meaning, not just words) — but the retrieve-then-inject flow is identical.

**Step 4c:** Now wire retrieval into the POST handler. Find the `# Call Bedrock` comment (just above `api_start = time.time()`). Paste the block below **immediately before that comment** — i.e. right after the `request_received` `logger.info(...)` block.

> **⚠️ Indentation matters in Python.** This block lives directly in the `lambda_handler` function body, so every line here is indented **exactly 4 spaces** (the `if`/`else` bodies go to 8). It must line up with the `logger.info(` and `# Call Bedrock` lines around it — **not** with the deeper-indented code inside the `try:` block. Paste it exactly as shown; don't add extra leading spaces.

```python
    # --- RAG: retrieve relevant knowledge and build a grounded prompt ---
    retrieved = retrieve_context(user_message)
    if retrieved:
        context_block = "\n\n".join(
            f"[Source: {c['source']}]\n{c['text']}" for c in retrieved
        )
        sources = sorted({c["source"] for c in retrieved})
    else:
        context_block = "(no relevant documents found)"
        sources = []

    logger.info(json.dumps({
        "level": "INFO",
        "event": "rag_retrieval",
        "request_id": request_id,
        "chunks_retrieved": len(retrieved),
        "sources": sources
    }))

    system_prompt = (
        "You are the AI Cloud Fusion workshop assistant. "
        "Answer the user's question using ONLY the context provided below. "
        "If the answer is not in the context, reply exactly: "
        "\"I don't have that in my knowledge base.\" "
        "Never invent information. When you use the context, cite the source "
        "filename in square brackets, e.g. [workshop-faq.txt].\n\n"
        "=== CONTEXT ===\n" + context_block + "\n=== END CONTEXT ==="
    )
```

> **💡 Why before the `try:` block?** Retrieval runs before the Bedrock call so the result is ready when you build the prompt. Keeping it out of the `try:` also means your `api_latency_ms` measures only the model call, not the S3 lookup.

**Step 4d:** Update the `bedrock.converse()` call to use this new grounded prompt. **Find** the `system=[...]` line inside the converse call (from Lab 11B) and **replace it** with:

```python
            system=[{"text": system_prompt}],
```

**Step 4e:** (Optional but recommended) surface the sources in the response so callers can see them. Find where the POST handler builds its `metadata` block for the return value and add a `sources` field to it, for example:

```python
                "sources": sources,
```

**Step 4f:** Save `handler.py`.

---

### 📄 Complete `handler.py` Reference (End of Step 4)

If your inline edits produced an error (a stray indent, a missing line, a misplaced block), don't debug it line by line — **replace the entire file** with the complete, known-good version below, save, and continue to Step 5. This is exactly what `handler.py` should contain at the end of Step 4 (it already includes everything from Labs 11A, 11B, and 11C plus this lab's RAG additions).

> **⚠️ Copy the whole block.** In Python, indentation *is* syntax — select from the first `import` to the final `}` and replace your file's full contents so nothing is left over from a bad paste.

```python
import json
import logging
import os
import re
import time
import boto3

logger = logging.getLogger()
logger.setLevel(logging.INFO)

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")
s3 = boto3.client("s3", region_name="us-east-1")
MODEL_ID = "amazon.nova-micro-v1:0"
KB_BUCKET = os.environ.get("KB_BUCKET", "")

# Cache the knowledge base across warm invocations (loaded once per cold start)
_KB_CACHE = None


def get_chat_html(event):
    """Return the chat UI HTML with the API URL auto-configured."""
    stage = event.get("requestContext", {}).get("stage", "live")
    api_id = event.get("requestContext", {}).get("apiId", "")
    api_url = f"https://{api_id}.execute-api.us-east-1.amazonaws.com/{stage}/chat"

    return f'''<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8"><meta name="viewport" content="width=device-width,initial-scale=1.0">
<title>AI Cloud Tutor</title>
<style>
*{{margin:0;padding:0;box-sizing:border-box}}
body{{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;background:#1a1a2e;color:#eee;height:100vh;display:flex;flex-direction:column}}
header{{background:#16213e;padding:16px 24px;border-bottom:1px solid #0f3460}}
header h1{{font-size:18px;color:#4fc3f7}}
header p{{font-size:12px;color:#888;margin-top:4px}}
#chat{{flex:1;overflow-y:auto;padding:24px;display:flex;flex-direction:column;gap:16px}}
.msg{{max-width:80%;padding:12px 16px;border-radius:12px;line-height:1.5;font-size:14px}}
.user{{align-self:flex-end;background:#0f3460;color:#e0e0e0}}
.bot{{align-self:flex-start;background:#222;border:1px solid #333}}
.bot pre{{white-space:pre-wrap;font-family:inherit}}
.meta{{font-size:11px;color:#666;margin-top:6px}}
.typing{{align-self:flex-start;color:#888;font-style:italic}}
#input-area{{background:#16213e;padding:16px 24px;border-top:1px solid #0f3460;display:flex;gap:12px}}
#input-area input{{flex:1;padding:12px 16px;border-radius:8px;border:1px solid #333;background:#1a1a2e;color:#eee;font-size:14px;outline:none}}
#input-area input:focus{{border-color:#4fc3f7}}
#input-area button{{padding:12px 24px;border-radius:8px;border:none;background:#4fc3f7;color:#000;font-weight:600;cursor:pointer;font-size:14px}}
#input-area button:disabled{{background:#333;color:#666;cursor:not-allowed}}
</style>
</head>
<body>
<header><h1>&#9729; AI Cloud Tutor</h1><p>Powered by Amazon Bedrock (Nova Micro)</p></header>
<div id="chat"></div>
<div id="input-area">
<input type="text" id="msg" placeholder="Ask a question about AWS..." autofocus>
<button id="send" onclick="sendMessage()">Send</button>
</div>
<script>
const API_URL='{api_url}';
const chat=document.getElementById('chat');
const input=document.getElementById('msg');
const btn=document.getElementById('send');
input.addEventListener('keydown',e=>{{if(e.key==='Enter'&&!btn.disabled)sendMessage()}});
async function sendMessage(){{
const message=input.value.trim();if(!message)return;
addMsg(message,'user');input.value='';btn.disabled=true;
const typing=document.createElement('div');typing.className='msg typing';typing.textContent='Thinking...';chat.appendChild(typing);chat.scrollTop=chat.scrollHeight;
try{{
const res=await fetch(API_URL,{{method:'POST',headers:{{'Content-Type':'application/json'}},body:JSON.stringify({{message}})}});
const data=await res.json();typing.remove();
addMsg(data.response||'No response','bot',data.metadata||{{}});
}}catch(err){{typing.remove();addMsg('Error: '+err.message,'bot')}}
btn.disabled=false;input.focus()}}
function addMsg(text,role,meta){{
const div=document.createElement('div');div.className='msg '+role;
if(role==='bot'){{const pre=document.createElement('pre');pre.textContent=text;div.appendChild(pre);
if(meta&&meta.total_tokens){{const m=document.createElement('div');m.className='meta';m.textContent=meta.api_latency_ms+'ms | '+meta.total_tokens+' tokens ('+meta.input_tokens+' in / '+meta.output_tokens+' out)';div.appendChild(m)}}
}}else{{div.textContent=text}}
chat.appendChild(div);chat.scrollTop=chat.scrollHeight}}
</script>
</body></html>'''


def build_messages(body, current_message):
    """Build message list with optional conversation history."""
    messages = []

    # Add prior turns if the caller sent any
    for msg in body.get("history", []):
        messages.append({
            "role": msg["role"],
            "content": [{"text": msg["content"]}]
        })

    # Add the current message
    messages.append({
        "role": "user",
        "content": [{"text": current_message}]
    })

    return messages


def load_knowledge_base():
    """Load and cache all KB chunks from S3 (runs once per cold start)."""
    global _KB_CACHE
    if _KB_CACHE is not None:
        return _KB_CACHE

    chunks = []
    if KB_BUCKET:
        listing = s3.list_objects_v2(Bucket=KB_BUCKET)
        for obj in listing.get("Contents", []):
            key = obj["Key"]
            text = s3.get_object(Bucket=KB_BUCKET, Key=key)["Body"].read().decode("utf-8")
            # Split each document into paragraph-sized chunks (blank line = new chunk)
            for para in re.split(r"\n\s*\n", text):
                para = para.strip()
                if para:
                    chunks.append({"source": key, "text": para})

    _KB_CACHE = chunks
    return chunks


def retrieve_context(query, top_k=3):
    """Return the top_k KB chunks most relevant to the query (keyword overlap)."""
    chunks = load_knowledge_base()
    query_words = set(re.findall(r"[a-z0-9]+", query.lower()))

    scored = []
    for chunk in chunks:
        chunk_words = set(re.findall(r"[a-z0-9]+", chunk["text"].lower()))
        overlap = len(query_words & chunk_words)
        if overlap > 0:
            scored.append((overlap, chunk))

    scored.sort(key=lambda item: item[0], reverse=True)
    return [chunk for _score, chunk in scored[:top_k]]


def lambda_handler(event, context):
    """AI Chatbot powered by Amazon Bedrock Nova Micro."""
    # Handle CORS preflight
    if event.get("httpMethod") == "OPTIONS":
        return {
            "statusCode": 200,
            "headers": {
                "Access-Control-Allow-Origin": "*",
                "Access-Control-Allow-Headers": "Content-Type",
                "Access-Control-Allow-Methods": "POST, GET, OPTIONS"
            },
            "body": ""
        }

    # Serve chat UI on GET requests (browser visits the URL)
    if event.get("httpMethod") == "GET":
        return {
            "statusCode": 200,
            "headers": {"Content-Type": "text/html"},
            "body": get_chat_html(event)
        }

    # POST requests = chat messages from the UI
    start_time = time.time()
    request_id = context.aws_request_id

    # Parse user input
    body = {}
    if event.get("body"):
        try:
            body = json.loads(event["body"])
        except (json.JSONDecodeError, TypeError):
            body = {}

    user_message = body.get("message", "What is cloud computing?")

    # Input validation — reject prompts that are too long
    MAX_INPUT_CHARS = 500  # ~125 tokens max input
    if len(user_message) > MAX_INPUT_CHARS:
        logger.warning(json.dumps({
            "level": "WARNING",
            "event": "input_rejected",
            "request_id": request_id,
            "reason": "prompt_too_long",
            "input_length": len(user_message),
            "max_allowed": MAX_INPUT_CHARS
        }))
        return {
            "statusCode": 400,
            "headers": {
                "Content-Type": "application/json",
                "Access-Control-Allow-Origin": "*"
            },
            "body": json.dumps({
                "error": "Your message is too long. Please keep it under 500 characters.",
                "input_length": len(user_message),
                "max_allowed": MAX_INPUT_CHARS
            })
        }

    logger.info(json.dumps({
        "level": "INFO",
        "event": "request_received",
        "request_id": request_id,
        "user_message": user_message,
        "model_id": MODEL_ID
    }))

    # --- RAG: retrieve relevant knowledge and build a grounded prompt ---
    retrieved = retrieve_context(user_message)
    if retrieved:
        context_block = "\n\n".join(
            f"[Source: {c['source']}]\n{c['text']}" for c in retrieved
        )
        sources = sorted({c["source"] for c in retrieved})
    else:
        context_block = "(no relevant documents found)"
        sources = []

    logger.info(json.dumps({
        "level": "INFO",
        "event": "rag_retrieval",
        "request_id": request_id,
        "chunks_retrieved": len(retrieved),
        "sources": sources
    }))

    system_prompt = (
        "You are the AI Cloud Fusion workshop assistant. "
        "Answer the user's question using ONLY the context provided below. "
        "If the answer is not in the context, reply exactly: "
        "\"I don't have that in my knowledge base.\" "
        "Never invent information. When you use the context, cite the source "
        "filename in square brackets, e.g. [workshop-faq.txt].\n\n"
        "=== CONTEXT ===\n" + context_block + "\n=== END CONTEXT ==="
    )

    # Call Bedrock
    api_start = time.time()
    try:
        response = bedrock.converse(
            modelId=MODEL_ID,
            system=[{"text": system_prompt}],
            messages=build_messages(body, user_message),
            inferenceConfig={
                "maxTokens": 150,
                "temperature": 0.5
            }
        )

        api_latency_ms = round((time.time() - api_start) * 1000, 2)
        assistant_message = response["output"]["message"]["content"][0]["text"]
        input_tokens = response["usage"]["inputTokens"]
        output_tokens = response["usage"]["outputTokens"]
        total_tokens = response["usage"]["totalTokens"]

        logger.info(json.dumps({
            "level": "INFO",
            "event": "bedrock_call_success",
            "request_id": request_id,
            "model_id": MODEL_ID,
            "api_latency_ms": api_latency_ms,
            "input_tokens": input_tokens,
            "output_tokens": output_tokens,
            "total_tokens": total_tokens
        }))

        bot_response = assistant_message

    except Exception as e:
        api_latency_ms = round((time.time() - api_start) * 1000, 2)
        logger.error(json.dumps({
            "level": "ERROR",
            "event": "bedrock_call_failed",
            "request_id": request_id,
            "model_id": MODEL_ID,
            "api_latency_ms": api_latency_ms,
            "error": str(e)
        }))
        bot_response = "Sorry, I couldn't generate a response right now. Try again later!"
        input_tokens = 0
        output_tokens = 0
        total_tokens = 0

    total_duration_ms = round((time.time() - start_time) * 1000, 2)
    logger.info(json.dumps({
        "level": "INFO",
        "event": "request_completed",
        "request_id": request_id,
        "total_duration_ms": total_duration_ms,
        "api_latency_ms": api_latency_ms
    }))

    return {
        "statusCode": 200,
        "headers": {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "*",
            "Access-Control-Allow-Headers": "Content-Type",
            "Access-Control-Allow-Methods": "POST, GET, OPTIONS"
        },
        "body": json.dumps({
            "response": bot_response,
            "metadata": {
                "model_id": MODEL_ID,
                "sources": sources,
                "api_latency_ms": api_latency_ms,
                "total_duration_ms": total_duration_ms,
                "input_tokens": input_tokens,
                "output_tokens": output_tokens,
                "total_tokens": total_tokens
            }
        })
    }
```

> **💡 After pasting the full file**, save it and go straight to Step 5 to deploy. The `KB_BUCKET` value is read from the environment variable you set in Step 5b — you never hard-code it, which is why the same file works in any environment.

---

### Step 5: Deploy and Point the Function at the Bucket

**Step 5a:** Re-deploy the code. 📋 Copy and paste:

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

**Step 5b:** Tell the function which bucket is its knowledge base by setting the `KB_BUCKET` environment variable. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws lambda update-function-configuration --function-name workshop-ai-chatbot-lab11 --environment "Variables={KB_BUCKET=workshop-ai-kb-<YOUR_ACCOUNT_ID>}" --region us-east-1 --query "LastUpdateStatus" --output text
```

**macOS / Linux:**
```bash
aws lambda update-function-configuration --function-name workshop-ai-chatbot-lab11 --environment "Variables={KB_BUCKET=workshop-ai-kb-<YOUR_ACCOUNT_ID>}" --region us-east-1 --query "LastUpdateStatus" --output text
```

**✅ You should see:** `InProgress` or `Successful`

> **💡 Why an environment variable instead of hard-coding the bucket name?** Config that changes between environments (dev, test, prod) belongs *outside* your code. Same code, different `KB_BUCKET` per environment. This is a 12-factor app best practice — and an exam-friendly one.

---

### Step 6: Prove RAG Works

**Step 6a: Ask a question only your KB can answer.** Edit `payload.json`:

```json
{"body": "{\"message\": \"What is the codename for the capstone chatbot project?\"}"}
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

**✅ You should see** the bot answer **"Project Northstar"** and cite **[project-northstar.txt]**. The model never learned this — it *retrieved* it. **That's RAG.** 🎉

**Step 6b: Ask something in a different document.** Edit `payload.json`:

```json
{"body": "{\"message\": \"When does the workshop meet and what certification should I get first?\"}"}
```

Save and invoke.

**✅ You should see** an answer grounded in `workshop-faq.txt` — "Wednesday afternoon at 2:00 PM Atlantic Standard Time" and "AWS Solutions Architect – Associate first" — with a `[workshop-faq.txt]` citation.

**Step 6c: Ask something NOT in the knowledge base.** Edit `payload.json`:

```json
{"body": "{\"message\": \"What is the capital of France?\"}"}
```

Save and invoke.

**✅ You should see** the bot reply **"I don't have that in my knowledge base."** — instead of answering "Paris." This is grounding in action: the bot refuses to answer outside its sources, even when it obviously *could*. In a real support bot, this is exactly what stops it from confidently inventing a refund policy that doesn't exist.

> **🎯 What you just proved:**
> - **On-topic + in KB** → grounded answer *with a citation*
> - **Not in KB** → honest "I don't know" instead of a hallucination
>
> Same model, same infrastructure. The only new ingredient is *retrieved context*.

**Step 6d:** Look at the CloudWatch log for your `rag_retrieval` event. 📋 Copy and paste:

**Windows (PowerShell) / macOS / Linux:**
```bash
aws logs filter-log-events --log-group-name "/aws/lambda/workshop-ai-chatbot-lab11" --filter-pattern "rag_retrieval" --region us-east-1 --query "events[-3:].message" --output text
```

**✅ You should see** JSON log lines showing `chunks_retrieved` and the `sources` list for your recent questions — your retrieval is observable, just like everything else you monitored in Session 11.

---

## What You Just Did

| What You Built | Why It Matters |
|---------------|----------------|
| S3 knowledge base | Your own data, editable any time — no model retraining |
| Retrieval function | Finds the relevant facts for each question (the "R" in RAG) |
| Grounded system prompt | Forces the AI to answer *only* from your documents |
| Source citations | Every answer is traceable back to a file — trust and auditability |
| Honest "I don't know" | Hallucination control — the mark of a production-grade assistant |

**Key takeaways:**
- **RAG > memory** for private or changing facts — update a file, not the model.
- **Grounding kills hallucinations** — "answer only from this context" is a powerful instruction.
- **Citations build trust** — users (and auditors) can verify every claim.
- **Retrieval is just search + inject** — the concept scales from keyword matching to vector embeddings.
- **RAG vs fine-tuning** is a classic exam trade-off: RAG is cheaper, faster to update, and better for facts.

---

## Cert Prep Callout

**Target Certification:** AWS Certified AI Practitioner

The AI Practitioner exam tests:
- **RAG** as the standard technique for grounding models in external/private data
- **RAG vs fine-tuning** trade-offs (cost, freshness, effort)
- Reducing **hallucination** through grounding and context
- Amazon Bedrock Knowledge Bases (the managed, embeddings-based version of what you built by hand)
- Least-privilege IAM for AI workloads

**Sample question type:** "A company wants its chatbot to answer questions using internal policy documents that change monthly, without retraining a model. Which approach should they use?"
**Answer:** Retrieval-Augmented Generation (RAG) — store the documents, retrieve relevant passages at query time, and include them in the prompt. It's cheaper and far easier to keep current than fine-tuning.

> **💡 What you built by hand, AWS offers as a managed service.** *Amazon Bedrock Knowledge Bases* automates chunking, embedding, vector storage (OpenSearch Serverless), and retrieval. You built the simplified version so you understand what's happening under the hood — that understanding is exactly what the exam checks.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| Bot says "I don't have that" for everything | `KB_BUCKET` not set, or bucket empty | Re-run Step 5b; confirm files uploaded (Step 2f) |
| `AccessDenied` in the logs | IAM S3 policy missing or wrong account ID | Re-check `kb-s3-policy.json` has your real account ID, then re-run Step 3b |
| Answers ignore the KB | Old code still deployed | Confirm Step 4 edits are saved, re-zip, re-deploy (Step 5a) |
| `NoSuchBucket` | Bucket name typo or wrong region | Bucket must be `workshop-ai-kb-<YOUR_ACCOUNT_ID>` in us-east-1 |
| Citation filename missing | Model didn't follow the prompt | Lower `temperature` (e.g. 0.3); confirm the `[Source: ...]` format is in `context_block` |
| Changes to a KB file not reflected | Old KB still cached in a warm Lambda | Edit + re-upload to S3, then re-deploy the function (Step 5a) to force a cold start |

---

## Cleanup

Pick the path that matches what you're doing next.

### Path A — Continuing to Lab 12B (recommended)

Keep everything — the function, role, S3 bucket, knowledge base, and your updated `handler.py`. Labs 12B and 12C build directly on all of it. **Do nothing here** and move on to Lab 12B.

### Path B — Full reset back to the end of Lab 11C

Use this to remove **everything Lab 12A created** and return your chatbot to its exact end-of-Lab-11C state — ideal for re-running this lab from a clean slate, or for stopping with zero ongoing footprint. Run from your `workshop-lab-11a` folder and replace `<YOUR_ACCOUNT_ID>` throughout.

**Step 1 — Delete the S3 knowledge base (objects, then the bucket):**
```
aws s3 rm s3://workshop-ai-kb-<YOUR_ACCOUNT_ID> --recursive --region us-east-1
aws s3 rb s3://workshop-ai-kb-<YOUR_ACCOUNT_ID> --region us-east-1
```

**Step 2 — Remove the S3-read permission from the role:**
```
aws iam delete-role-policy --role-name workshop-lab11-lambda-role --policy-name kb-s3-read
```

**Step 3 — Clear the `KB_BUCKET` environment variable** (11C had no environment variables):
```
aws lambda update-function-configuration --function-name workshop-ai-chatbot-lab11 --environment "Variables={}" --region us-east-1 --query "LastUpdateStatus" --output text
```

**Step 4 — Restore the Lambda code to the Lab 11C version.** Redeploy the backup you made at the start of Step 4.

**Windows (PowerShell):**
```powershell
Copy-Item handler-11c-backup.py handler.py -Force
Compress-Archive -Path handler.py -DestinationPath function.zip -Force
aws lambda update-function-code --function-name workshop-ai-chatbot-lab11 --zip-file fileb://function.zip --region us-east-1 --query "LastUpdateStatus" --output text
```

**macOS / Linux:**
```bash
cp handler-11c-backup.py handler.py
zip function.zip handler.py
aws lambda update-function-code --function-name workshop-ai-chatbot-lab11 --zip-file fileb://function.zip --region us-east-1 --query "LastUpdateStatus" --output text
```

> **💡 No backup?** The end-of-11C code is this lab's file with the 12A additions removed: delete the `import os` / `import re` / `s3 = ...` / `KB_BUCKET = ...` / `_KB_CACHE = None` lines, the `load_knowledge_base` and `retrieve_context` functions, and the `# --- RAG ...` block; change `system=[{"text": system_prompt}]` back to your Lab 11B literal system prompt; and remove `"sources": sources,` from the metadata. Then re-zip and re-deploy.

**Step 5 — Remove the local 12A working files:**

**Windows (PowerShell):**
```powershell
Remove-Item -Recurse -Force knowledge-base; Remove-Item -Force kb-s3-policy.json
```

**macOS / Linux:**
```bash
rm -rf knowledge-base && rm -f kb-s3-policy.json
```

**✅ Verify you're back to the 11C baseline:**
```
aws s3 ls | grep workshop-ai-kb
aws lambda get-function-configuration --function-name workshop-ai-chatbot-lab11 --region us-east-1 --query "Environment" --output text
```
The first command should return nothing (bucket gone); the second should return `None` (no env vars). Then invoke a normal question like "What is S3?" — the bot should answer directly again, exactly as it did at the end of Lab 11C.

> **📌 Tearing down the *entire* AI stack** (Lambda, role, API Gateway, and all Session 11 resources)? That complete teardown lives in **Lab 12C**.

---

## Help

If you get stuck, post in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** you received
4. Your **operating system** (Windows / Mac / Linux)
