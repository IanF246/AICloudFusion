# Lab 11A: Your First AI Chatbot — Call Amazon Bedrock from Lambda

**Session:** 11 — AI Engineering  
**Track:** AI Engineering  
**Difficulty:** Beginner  
**Estimated Time:** 40–50 minutes  
**Target Cert:** AWS Certified AI Practitioner

---

## Overview

In Session 10, your chatbot called a trivia API. In this lab, you'll replace that with **Amazon Bedrock** — AWS's generative AI service. Your Lambda will send a prompt to an AI model and get back an intelligent, generated response. This is exactly how production AI applications work: ChatGPT, customer support bots, content generators — they all call an AI model API and return the response.

By the end of this lab you will have:

1. **A Lambda function** that calls Amazon Bedrock's Nova Micro model
2. **An API Gateway endpoint** that makes your chatbot accessible via a URL
3. **A browser-based chat UI** — you'll open a URL and chat with your AI bot
4. **Structured logging** tracking: latency, token usage (input/output), and model ID

**Why this matters:** AI engineering is the fastest-growing skill in tech. After this lab, you can share a URL with friends and say "I built this AI chatbot on AWS" — and describe exactly how it works.

---

## Prerequisites

- ✅ AWS CLI authenticated (`aws sts get-caller-identity`)
- ✅ Completed **Lab 1A** (AWS account + CLI setup)

> **💡 This lab is standalone.** You do NOT need Sessions 7–10 to complete it.

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| AWS Lambda | Function execution | Always Free (1M requests/month) |
| Amazon API Gateway | Public HTTP endpoint | Always Free (1M calls/month) |
| Amazon CloudWatch Logs | Log storage | Always Free (5GB/month) |
| Amazon Bedrock (Nova Micro) | AI model inference | ~$0.01–0.05 for this lab |

**⚠️ Estimated cost for this lab: under $0.05**

Amazon Bedrock charges per token (a token ≈ 1 word). Nova Micro is the cheapest model. With ~20 invocations of short prompts, total cost is fractions of a cent. **Complete the Cleanup section** to ensure no ongoing charges.

---

## Concepts

**Amazon Bedrock** — AWS's fully managed AI service. It gives you access to foundation models (large language models) from Amazon, Anthropic, Meta, and others — all through a simple API call. You don't need to train a model, manage GPUs, or understand machine learning. You send a prompt, you get a response.

**Foundation Model** — a large AI model trained on vast amounts of text. It can answer questions, summarize, write code, and have conversations. In this lab you'll use **Amazon Nova Micro** — Amazon's fastest and cheapest text model.

**Token** — the unit AI models use to measure text. Roughly 1 token ≈ 1 word. You're charged per token: input tokens (your prompt) + output tokens (the AI's response). Tracking tokens = tracking cost.

**The Converse API** — Bedrock's unified API for calling any model. You send messages in a standard format, and it works the same regardless of which model you choose.

**API Gateway** — an AWS service that creates a public URL endpoint for your Lambda function. Without it, your Lambda can only be called via the CLI. With it, anyone with the URL can interact with your chatbot from a browser.

**Why Lambda + API Gateway + Bedrock?** Lambda runs the code, API Gateway provides the public URL, and Bedrock provides the AI. Together, they give you a serverless AI application accessible from anywhere.

---

## ⚠️ Reminder: How to Read Commands

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account number | `123456789012` |

---

## Lab Steps

### Step 1: Set Your AWS Profile and Create a Project Folder

**Step 1a:** Set your AWS profile.

**Windows (PowerShell):**
```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**macOS / Linux:**
```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

> **⚠️ This setting is lost if you close your terminal.** If any command later gives an auth error, come back here and re-set it.

**Step 1b:** Verify you're authenticated. 📋 Copy and paste:

```
aws sts get-caller-identity
```

**✅ You should see** your account ID. If you get an expired token error, run `aws sso login --profile <YOUR_PROFILE_NAME>`, approve in browser, and try again.

**📝 Write down your 12-digit account ID** — you'll need it several times.

**Step 1c:** Create a project folder.

**Windows (PowerShell):**
```powershell
mkdir ~\Desktop\workshop-lab-11a
cd ~\Desktop\workshop-lab-11a
pwd
```

**macOS / Linux:**
```bash
mkdir ~/Desktop/workshop-lab-11a
cd ~/Desktop/workshop-lab-11a
pwd
```

**✅ You should see** the path to your new folder.

---

### Step 2: Test Bedrock Directly (Verify Access)

Before building the Lambda, let's verify Bedrock works on your account.

**Step 2a:** Create a test prompt file. Open your text editor → new file. 📋 Copy and paste:

```json
{
    "modelId": "amazon.nova-micro-v1:0",
    "messages": [
        {
            "role": "user",
            "content": [{"text": "What is AWS Lambda in one sentence?"}]
        }
    ],
    "inferenceConfig": {
        "maxTokens": 100,
        "temperature": 0.7
    }
}
```

**Step 2b:** Save it as `test-prompt.json` in your `workshop-lab-11a` folder.

**Step 2c:** Call Bedrock. 📋 Copy and paste (from your `workshop-lab-11a` folder):

```
aws bedrock-runtime converse --cli-input-json file://test-prompt.json --region us-east-1
```

**✅ You should see** JSON output with the AI's response, token usage (`inputTokens`, `outputTokens`), and `latencyMs`.

> **🎉 You just called an AI model!** Everything in this lab builds on this single API call.

> **⚠️ If you get "Operation not allowed":** Your account may need Bedrock activated. Open AWS Console → Bedrock → Model catalog → click Nova Micro → try the Playground. If both fail, contact AWS Support.

---

### Step 3: Create the AI Chatbot Lambda Code

This Lambda does three things: serves a chat UI on GET requests, handles AI conversations on POST requests, and handles CORS preflight on OPTIONS requests.

**Step 3a:** Open your text editor → new file. 📋 Copy and paste this entire code block:

```python
import json
import logging
import time
import boto3

logger = logging.getLogger()
logger.setLevel(logging.INFO)

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")
MODEL_ID = "amazon.nova-micro-v1:0"


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

    logger.info(json.dumps({
        "level": "INFO",
        "event": "request_received",
        "request_id": request_id,
        "user_message": user_message,
        "model_id": MODEL_ID
    }))

    # Call Bedrock
    api_start = time.time()
    try:
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
                "api_latency_ms": api_latency_ms,
                "total_duration_ms": total_duration_ms,
                "input_tokens": input_tokens,
                "output_tokens": output_tokens,
                "total_tokens": total_tokens
            }
        })
    }
```

**Step 3b:** Save as `handler.py` in your `workshop-lab-11a` folder.

> ⚠️ **Windows users:** in Notepad, change "Save as type" to **"All Files"**.

**Step 3c: What does this code do?**

This Lambda handles three types of requests:

| Request Type | What Happens |
|-------------|--------------|
| **GET** (browser visits URL) | Returns an HTML chat page — the user interface |
| **POST** (chat message sent) | Sends the message to Bedrock, returns AI response |
| **OPTIONS** (browser CORS check) | Returns permission headers so the browser allows the request |

The chat UI is embedded directly in the Lambda — no separate hosting needed. When you open the URL in a browser, the Lambda returns the HTML page. When you type a message and hit Send, the page sends a POST request back to the same URL, and the Lambda calls Bedrock and returns the AI's answer.

---

### Step 4: Create the Lambda Role (with Bedrock Permission)

**Step 4a:** Create the trust policy. Open your text editor → new file. 📋 Copy and paste:

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

**Step 4b:** Save as `lambda-trust.json` in your `workshop-lab-11a` folder.

**Step 4c:** Create the Bedrock permission policy. Open your text editor → new file. 📋 Copy and paste:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "bedrock:InvokeModel"
            ],
            "Resource": "arn:aws:bedrock:us-east-1::foundation-model/amazon.nova-micro-v1:0"
        }
    ]
}
```

**Step 4d:** Save as `bedrock-policy.json` in your `workshop-lab-11a` folder.

> **What does this do?** Gives Lambda permission to call ONLY the Nova Micro model. This is **least privilege** — your function can invoke one specific model and nothing more.

**Step 4e:** Create the role and attach both policies. 📋 Copy and paste each command (from your `workshop-lab-11a` folder):

```
aws iam create-role --role-name workshop-lab11-lambda-role --assume-role-policy-document file://lambda-trust.json
```

**✅ You should see** JSON with the role ARN.

```
aws iam attach-role-policy --role-name workshop-lab11-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

**✅ No output means success.**

```
aws iam put-role-policy --role-name workshop-lab11-lambda-role --policy-name bedrock-invoke --policy-document file://bedrock-policy.json
```

**✅ No output means success.**

> **💡 Wait 10 seconds** for IAM role propagation before the next step.

---

### Step 5: Package and Deploy the Lambda

> **⚠️ Verify you're in the `workshop-lab-11a` folder** (`pwd`).

**Step 5a:** Zip the handler.

**Windows (PowerShell):**
```powershell
Compress-Archive -Path handler.py -DestinationPath function.zip -Force
```

**macOS / Linux:**
```bash
zip function.zip handler.py
```

**Step 5b:** Create the Lambda function. 📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>`**:

```
aws lambda create-function --function-name workshop-ai-chatbot-lab11 --runtime python3.12 --role arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-lab11-lambda-role --handler handler.lambda_handler --zip-file fileb://function.zip --timeout 30 --memory-size 256 --region us-east-1
```

**✅ You should see** JSON output with `"State": "Pending"` or `"Active"`.

> **Why these settings?**
> - `--timeout 30` — Bedrock can take a few seconds to respond (especially on cold start)
> - `--memory-size 256` — the boto3 SDK + Bedrock client needs more memory than a simple function

---

### Step 6: Create a Public URL with API Gateway

Right now your Lambda can only be called via the CLI. API Gateway gives it a public URL so you can access it from a browser.

> **⚠️ In this step you'll store IDs in variables.** This means you must run all commands in the SAME terminal session. If you close your terminal, the variables are lost and you'll need to look up the IDs manually.

**Step 6a:** Create the API and store its ID. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
$API_ID = aws apigateway create-rest-api --name workshop-ai-chatbot --region us-east-1 --query "id" --output text
Write-Host "API ID: $API_ID"
```

**macOS / Linux:**
```bash
API_ID=$(aws apigateway create-rest-api --name workshop-ai-chatbot --region us-east-1 --query "id" --output text)
echo "API ID: $API_ID"
```

**✅ You should see** `API ID:` followed by a short ID like `u4qkyk0w25`.

**Step 6b:** Get the root resource ID. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
$ROOT_ID = aws apigateway get-resources --rest-api-id $API_ID --region us-east-1 --query "items[0].id" --output text
Write-Host "Root ID: $ROOT_ID"
```

**macOS / Linux:**
```bash
ROOT_ID=$(aws apigateway get-resources --rest-api-id $API_ID --region us-east-1 --query "items[0].id" --output text)
echo "Root ID: $ROOT_ID"
```

**✅ You should see** `Root ID:` followed by another short ID.

**Step 6c:** Create a `/chat` resource. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
$CHAT_ID = aws apigateway create-resource --rest-api-id $API_ID --parent-id $ROOT_ID --path-part chat --region us-east-1 --query "id" --output text
Write-Host "Chat ID: $CHAT_ID"
```

**macOS / Linux:**
```bash
CHAT_ID=$(aws apigateway create-resource --rest-api-id $API_ID --parent-id $ROOT_ID --path-part chat --region us-east-1 --query "id" --output text)
echo "Chat ID: $CHAT_ID"
```

**✅ You should see** `Chat ID:` followed by another short ID.

**Step 6d:** Add GET, POST, and OPTIONS methods. 📋 Copy and paste all three:

**Windows (PowerShell):**
```powershell
aws apigateway put-method --rest-api-id $API_ID --resource-id $CHAT_ID --http-method GET --authorization-type NONE --region us-east-1 | Out-Null
aws apigateway put-method --rest-api-id $API_ID --resource-id $CHAT_ID --http-method POST --authorization-type NONE --region us-east-1 | Out-Null
aws apigateway put-method --rest-api-id $API_ID --resource-id $CHAT_ID --http-method OPTIONS --authorization-type NONE --region us-east-1 | Out-Null
Write-Host "Methods created: GET, POST, OPTIONS"
```

**macOS / Linux:**
```bash
aws apigateway put-method --rest-api-id $API_ID --resource-id $CHAT_ID --http-method GET --authorization-type NONE --region us-east-1 > /dev/null
aws apigateway put-method --rest-api-id $API_ID --resource-id $CHAT_ID --http-method POST --authorization-type NONE --region us-east-1 > /dev/null
aws apigateway put-method --rest-api-id $API_ID --resource-id $CHAT_ID --http-method OPTIONS --authorization-type NONE --region us-east-1 > /dev/null
echo "Methods created: GET, POST, OPTIONS"
```

**✅ You should see** `Methods created: GET, POST, OPTIONS`

**Step 6e:** Connect each method to your Lambda. 📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>`** (1 place):

**Windows (PowerShell):**
```powershell
$LAMBDA_URI = "arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:<YOUR_ACCOUNT_ID>:function:workshop-ai-chatbot-lab11/invocations"
aws apigateway put-integration --rest-api-id $API_ID --resource-id $CHAT_ID --http-method GET --type AWS_PROXY --integration-http-method POST --uri $LAMBDA_URI --region us-east-1 | Out-Null
aws apigateway put-integration --rest-api-id $API_ID --resource-id $CHAT_ID --http-method POST --type AWS_PROXY --integration-http-method POST --uri $LAMBDA_URI --region us-east-1 | Out-Null
aws apigateway put-integration --rest-api-id $API_ID --resource-id $CHAT_ID --http-method OPTIONS --type AWS_PROXY --integration-http-method POST --uri $LAMBDA_URI --region us-east-1 | Out-Null
Write-Host "Integrations connected"
```

**macOS / Linux:**
```bash
LAMBDA_URI="arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:<YOUR_ACCOUNT_ID>:function:workshop-ai-chatbot-lab11/invocations"
aws apigateway put-integration --rest-api-id $API_ID --resource-id $CHAT_ID --http-method GET --type AWS_PROXY --integration-http-method POST --uri $LAMBDA_URI --region us-east-1 > /dev/null
aws apigateway put-integration --rest-api-id $API_ID --resource-id $CHAT_ID --http-method POST --type AWS_PROXY --integration-http-method POST --uri $LAMBDA_URI --region us-east-1 > /dev/null
aws apigateway put-integration --rest-api-id $API_ID --resource-id $CHAT_ID --http-method OPTIONS --type AWS_PROXY --integration-http-method POST --uri $LAMBDA_URI --region us-east-1 > /dev/null
echo "Integrations connected"
```

> **⚠️ Replace `<YOUR_ACCOUNT_ID>` in the first line** with your 12-digit account ID. This is the only placeholder in this step — the `$API_ID` and `$CHAT_ID` variables handle the rest automatically.

**✅ You should see** `Integrations connected`

**Step 6f:** Give API Gateway permission to invoke your Lambda. 📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>`**:

**Windows (PowerShell):**
```powershell
aws lambda add-permission --function-name workshop-ai-chatbot-lab11 --statement-id apigateway-invoke --action lambda:InvokeFunction --principal apigateway.amazonaws.com --source-arn "arn:aws:execute-api:us-east-1:<YOUR_ACCOUNT_ID>:$API_ID/*" --region us-east-1 | Out-Null
Write-Host "Lambda permission added"
```

**macOS / Linux:**
```bash
aws lambda add-permission --function-name workshop-ai-chatbot-lab11 --statement-id apigateway-invoke --action lambda:InvokeFunction --principal apigateway.amazonaws.com --source-arn "arn:aws:execute-api:us-east-1:$(aws sts get-caller-identity --query Account --output text):$API_ID/*" --region us-east-1 > /dev/null
echo "Lambda permission added"
```

> **⚠️ Windows users: Replace `<YOUR_ACCOUNT_ID>`** with your 12-digit account ID. The `$API_ID` part is handled by the variable you set in Step 6a — don't replace that.

**✅ You should see** `Lambda permission added`

**Step 6g:** Deploy the API. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws apigateway create-deployment --rest-api-id $API_ID --stage-name live --region us-east-1 | Out-Null
Write-Host "Deployed! Your chatbot URL is:"
Write-Host "https://$API_ID.execute-api.us-east-1.amazonaws.com/live/chat"
```

**macOS / Linux:**
```bash
aws apigateway create-deployment --rest-api-id $API_ID --stage-name live --region us-east-1 > /dev/null
echo "Deployed! Your chatbot URL is:"
echo "https://$API_ID.execute-api.us-east-1.amazonaws.com/live/chat"
```

**✅ You should see** your chatbot URL printed. **📝 Copy this URL** — you're about to open it in your browser!

> **💡 What just happened?** You created a public HTTP endpoint that routes browser requests to your Lambda. When someone visits the URL, API Gateway sends the request to Lambda, Lambda returns the chat page (on GET) or an AI response (on POST).

---

### Step 7: 🎉 Open Your Chatbot in the Browser!

**Step 7a:** Open your browser and navigate to your chatbot URL:

```
https://<API_ID>.execute-api.us-east-1.amazonaws.com/live/chat
```

> **⚠️ Replace `<API_ID>`** with the ID you wrote down in Step 6a.

**✅ You should see** a dark-themed chat interface titled "AI Cloud Tutor" with a text input and Send button.

**Step 7b:** Type a question and hit Send (or press Enter). Try:
- "What is S3?"
- "Explain IAM roles like I'm 5"
- "Write a haiku about cloud computing"

**✅ You should see:**
1. Your message appears on the right (blue bubble)
2. "Thinking..." appears briefly
3. The AI's response appears on the left (dark bubble)
4. Below the response, you'll see metadata: latency in ms, token count (input/output)

**Step 7c:** Try several different questions. Notice:
- Each response takes ~500–800ms (the time Bedrock needs to generate text)
- Short questions use few input tokens (5–15)
- Longer responses use more output tokens (50–200)
- **Every token costs money** — the metadata shows you exactly how much each question "costs" in tokens

> **💡 Share this URL with a friend!** Anyone with the link can chat with your AI bot. This is a real, live, serverless AI application running on AWS — and you built it.

> **⚠️ Remember:** Every message costs a tiny amount (~$0.00002 per request). The URL will keep working until you delete the resources in Cleanup. Don't leave it running indefinitely if you're worried about cost — but 100 messages would cost less than $0.01.

---

### Step 8: ✅ Console Checkpoint — View Logs in CloudWatch

**Step 8a:** Open CloudWatch → **Logs** → **Log groups** → `/aws/lambda/workshop-ai-chatbot-lab11`

**Step 8b:** Click the most recent log stream. You should see structured entries for each chat message:
- `"event": "request_received"` with the user's message
- `"event": "bedrock_call_success"` with `api_latency_ms`, `input_tokens`, `output_tokens`
- `"event": "request_completed"` with total duration

> **💡 Every message sent through the chat UI shows up here.** This is the same observability pattern from Sessions 9–10 — but now you're monitoring an AI application.

---

## What You Just Did

| What You Built | Why It Matters |
|---------------|----------------|
| Called Bedrock directly from CLI | Proved AI model access works |
| Lambda function calling Bedrock | Serverless AI — scales automatically, costs nothing when idle |
| API Gateway with public URL | Your chatbot is accessible from any browser, anywhere |
| Browser-based chat UI served from Lambda | A complete AI application — no separate frontend hosting needed |
| Structured logging with token tracking | You know exactly what each request costs |

**Key takeaways:**
- **Bedrock is just an API call** — send a prompt, get a response. No ML expertise needed.
- **Lambda + API Gateway + Bedrock = a complete AI app** — serverless, scalable, publicly accessible
- **The chat UI is served by the same Lambda** — GET returns HTML, POST calls the AI. One function does everything.
- **Tokens = cost.** The metadata in every response shows exactly what you're spending.
- **You can share this URL** — it's a portfolio piece. "I built a serverless AI chatbot on AWS."

---

## Cert Prep Callout

**Target Certification:** AWS Certified AI Practitioner

The AI Practitioner exam tests:
- Amazon Bedrock as a managed AI service
- Foundation models and token-based pricing
- API Gateway as an endpoint for Lambda-based applications
- Serverless architecture patterns for AI
- The Converse API as the unified Bedrock interface

**Sample question type:** "A developer wants to build a publicly accessible AI chatbot with minimal infrastructure management. Which AWS services should they combine?"  
**Answer:** Amazon Bedrock for AI inference, AWS Lambda for compute, and Amazon API Gateway for a public HTTP endpoint. This creates a fully serverless AI application that scales automatically.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `Operation not allowed` on Bedrock | Account-level restriction | Try Bedrock Playground in console. If both fail, open an AWS Support ticket. Students with standalone accounts typically don't have this issue. |
| `AccessDeniedException` on invoke | Lambda role missing bedrock permission | Re-run the `put-role-policy` command from Step 4e |
| `The role cannot be assumed` | IAM propagation delay | Wait 10 seconds and try again |
| Chat UI loads but messages return errors | API Gateway or Lambda permission issue | Check Step 6f — the `add-permission` command must include the correct API ID |
| `Missing Authentication Token` in browser | You're hitting the wrong URL path | Make sure the URL ends with `/live/chat` (not just `/live/` or `/chat`) |
| Very slow first response (>10s) | Lambda cold start + Bedrock warmup | Normal on first call. Subsequent calls should be 500–800ms |
| `Unable to import module 'handler'` | Zip structure wrong | Delete function.zip, verify you're in the right folder, re-zip |
| Chat UI shows but sends to wrong URL | API ID mismatch in the HTML | The HTML auto-detects the API URL from the request — this shouldn't happen. Re-deploy if it does. |
| `Invalid base64` on CLI invoke | Missing `--cli-binary-format raw-in-base64-out` flag | Add the flag to CLI invoke commands |

---

## Cleanup

> **📌 If continuing to Lab 11B:** Keep the function, role, and API Gateway. Lab 11B updates the same chatbot.

**If stopping here:**

**Step 1:** Delete the API Gateway (replace `<API_ID>` with your API ID from Step 6a, or use `$API_ID` if your terminal is still open):

**Windows (PowerShell):**
```powershell
aws apigateway delete-rest-api --rest-api-id $API_ID --region us-east-1
```

**macOS / Linux:**
```bash
aws apigateway delete-rest-api --rest-api-id $API_ID --region us-east-1
```

> **💡 If `$API_ID` is empty** (you closed your terminal), find it with: `aws apigateway get-rest-apis --region us-east-1 --query "items[?name=='workshop-ai-chatbot'].id" --output text`

**Step 2:** Delete the Lambda function:
```
aws lambda delete-function --function-name workshop-ai-chatbot-lab11 --region us-east-1
```

**Step 3:** Delete the log group:
```
aws logs delete-log-group --log-group-name /aws/lambda/workshop-ai-chatbot-lab11 --region us-east-1
```

**Step 4:** Remove the role:
```
aws iam delete-role-policy --role-name workshop-lab11-lambda-role --policy-name bedrock-invoke
aws iam detach-role-policy --role-name workshop-lab11-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name workshop-lab11-lambda-role
```

**Step 5:** Delete the project folder:

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
