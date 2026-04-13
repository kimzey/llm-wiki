# RAG Agent — ช่องทางการเชื่อมต่อและใช้งานทั้งหมด

## สารบัญ

1. [ภาพรวม Architecture](#1-ภาพรวม-architecture)
2. [LINE Official Account](#2-line-official-account)
3. [Slack Bot](#3-slack-bot)
4. [Discord Bot](#4-discord-bot)
5. [Web Chat Widget](#5-web-chat-widget)
6. [Facebook Messenger](#6-facebook-messenger)
7. [Microsoft Teams](#7-microsoft-teams)
8. [Telegram Bot](#8-telegram-bot)
9. [WhatsApp Business](#9-whatsapp-business)
10. [Mobile App (React Native)](#10-mobile-app)
11. [Internal Dashboard (Next.js)](#11-internal-dashboard)
12. [Google Chat](#12-google-chat)
13. [Email (Auto-reply)](#13-email-auto-reply)
14. [Voice Assistant (โทรศัพท์)](#14-voice-assistant)
15. [API สำหรับ Third-party](#15-api-สำหรับ-third-party)
16. [Low-code Platforms](#16-low-code-platforms)
17. [เปรียบเทียบทุกช่องทาง](#17-เปรียบเทียบทุกช่องทาง)
18. [แนะนำสำหรับ Sellsuki](#18-แนะนำสำหรับ-sellsuki)

---

## 1. ภาพรวม Architecture

```
                         ┌──────────────────────────┐
                         │    RAG Agent API          │
                         │    (FastAPI Backend)      │
                         │                           │
                         │  POST /chat               │
                         │  { message, session_id }  │
                         │  → { answer, sources }    │
                         └────────────┬─────────────┘
                                      │
            ┌─────────────────────────┼──────────────────────────┐
            │            │            │            │              │
     ┌──────▼──┐  ┌──────▼──┐  ┌─────▼───┐ ┌─────▼───┐  ┌──────▼──────┐
     │  LINE   │  │  Slack  │  │  Web    │ │ Discord │  │  Messenger  │
     │  Bot    │  │  Bot    │  │  Chat   │ │  Bot    │  │    Bot      │
     └─────────┘  └─────────┘  └─────────┘ └─────────┘  └─────────────┘
            │            │            │            │              │
     ┌──────▼──┐  ┌──────▼──┐  ┌─────▼───┐ ┌─────▼───┐  ┌──────▼──────┐
     │  Teams  │  │Telegram │  │WhatsApp │ │ Google  │  │   Email     │
     │  Bot    │  │  Bot    │  │Business │ │  Chat   │  │  Auto-reply │
     └─────────┘  └─────────┘  └─────────┘ └─────────┘  └─────────────┘
            │            │
     ┌──────▼──┐  ┌──────▼──────┐
     │  Mobile │  │  Voice/Phone│
     │  App    │  │  (Twilio)   │
     └─────────┘  └─────────────┘
```

**หลักการ:** ทุกช่องทางเชื่อมต่อกับ **RAG Agent API ตัวเดียวกัน** ผ่าน HTTP POST
แต่ละช่องทางมี "Adapter" ที่รับ message จาก platform → ส่งไปหา API → ส่งคำตอบกลับ

---

## 2. LINE Official Account

### ทำไม LINE สำคัญ
- คนไทยใช้ LINE มากที่สุด (มากกว่า 54 ล้าน users ในไทย)
- เหมาะสำหรับ internal (พนักงาน) และ external (ลูกค้า)
- มี Rich Menu, Flex Message, Quick Reply ทำ UX ได้ดี

### สิ่งที่ต้องเตรียม
1. LINE Official Account (สมัครที่ [LINE for Business](https://manager.line.biz/))
2. เปิด Messaging API
3. Channel Access Token & Channel Secret

### Architecture

```
User (LINE App)
    │
    ▼
LINE Platform
    │ Webhook (POST)
    ▼
Your Server (Webhook Handler)
    │
    ▼
RAG Agent API (/chat)
    │
    ▼
Your Server
    │ Reply API
    ▼
LINE Platform → User
```

### Implementation

```python
# pip install line-bot-sdk fastapi uvicorn httpx

from fastapi import FastAPI, Request, HTTPException
from linebot.v3 import WebhookHandler
from linebot.v3.messaging import (
    Configuration, ApiClient, MessagingApi,
    ReplyMessageRequest, TextMessage, FlexMessage,
    FlexContainer,
)
from linebot.v3.webhooks import MessageEvent, TextMessageContent
import httpx
import os

app = FastAPI()

# === LINE Config ===
configuration = Configuration(
    access_token=os.getenv("LINE_CHANNEL_ACCESS_TOKEN")
)
handler = WebhookHandler(os.getenv("LINE_CHANNEL_SECRET"))

# === RAG Agent API URL ===
AGENT_API_URL = "http://localhost:8000/chat"  # หรือ production URL

# === Webhook Endpoint ===
@app.post("/webhook/line")
async def line_webhook(request: Request):
    signature = request.headers.get("X-Line-Signature", "")
    body = (await request.body()).decode("utf-8")
    
    try:
        handler.handle(body, signature)
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))
    
    return {"status": "ok"}

# === Handle Text Messages ===
@handler.add(MessageEvent, message=TextMessageContent)
def handle_text_message(event: MessageEvent):
    user_id = event.source.user_id
    user_message = event.message.text
    
    # เรียก RAG Agent
    with httpx.Client() as client:
        response = client.post(
            AGENT_API_URL,
            json={
                "message": user_message,
                "session_id": f"line_{user_id}",  # ใช้ LINE user_id เป็น session
            },
            timeout=30.0,
        )
        data = response.json()
    
    answer = data["answer"]
    sources = data.get("sources", [])
    
    # สร้าง reply
    with ApiClient(configuration) as api_client:
        messaging_api = MessagingApi(api_client)
        
        # ถ้ามี sources → ใช้ Flex Message (สวยกว่า)
        if sources:
            reply = create_flex_message(answer, sources)
        else:
            reply = TextMessage(text=answer)
        
        messaging_api.reply_message(
            ReplyMessageRequest(
                reply_token=event.reply_token,
                messages=[reply]
            )
        )

def create_flex_message(answer: str, sources: list) -> FlexMessage:
    """สร้าง Flex Message แสดงคำตอบ + แหล่งอ้างอิง"""
    source_texts = "\n".join([f"• {s.get('title', 'N/A')}" for s in sources[:3]])
    
    flex_content = {
        "type": "bubble",
        "body": {
            "type": "box",
            "layout": "vertical",
            "contents": [
                {
                    "type": "text",
                    "text": "🤖 Suki Bot",
                    "weight": "bold",
                    "size": "md",
                    "color": "#1DB446"
                },
                {
                    "type": "text",
                    "text": answer,
                    "wrap": True,
                    "margin": "md",
                    "size": "sm"
                },
                {
                    "type": "separator",
                    "margin": "lg"
                },
                {
                    "type": "text",
                    "text": f"📎 อ้างอิง:\n{source_texts}",
                    "wrap": True,
                    "margin": "md",
                    "size": "xs",
                    "color": "#888888"
                }
            ]
        }
    }
    
    return FlexMessage(
        alt_text=answer[:400],
        contents=FlexContainer.from_dict(flex_content)
    )


# === Rich Menu (เมนูด้านล่าง) ===
# สร้างผ่าน LINE Official Account Manager UI
# หรือ API:
def create_rich_menu():
    """สร้าง Rich Menu สำหรับเข้าถึง feature ต่างๆ"""
    with ApiClient(configuration) as api_client:
        messaging_api = MessagingApi(api_client)
        
        rich_menu = {
            "size": {"width": 2500, "height": 1686},
            "selected": True,
            "name": "Suki Bot Menu",
            "chatBarText": "เมนู",
            "areas": [
                {
                    "bounds": {"x": 0, "y": 0, "width": 1250, "height": 843},
                    "action": {"type": "message", "text": "สวัสดิการบริษัทมีอะไรบ้าง?"}
                },
                {
                    "bounds": {"x": 1250, "y": 0, "width": 1250, "height": 843},
                    "action": {"type": "message", "text": "นโยบายการลามีอะไรบ้าง?"}
                },
                {
                    "bounds": {"x": 0, "y": 843, "width": 1250, "height": 843},
                    "action": {"type": "message", "text": "วิธีใช้งาน Sellsuki"}
                },
                {
                    "bounds": {"x": 1250, "y": 843, "width": 1250, "height": 843},
                    "action": {"type": "message", "text": "ติดต่อ HR"}
                },
            ]
        }
        # Upload via API...


# === Quick Reply (ปุ่มตอบเร็ว) ===
def create_quick_reply_message(answer: str) -> TextMessage:
    """เพิ่มปุ่ม Quick Reply ใต้ข้อความ"""
    return TextMessage(
        text=answer,
        quick_reply={
            "items": [
                {
                    "type": "action",
                    "action": {"type": "message", "label": "ลาพักร้อน", "text": "ลาพักร้อนทำยังไง?"}
                },
                {
                    "type": "action",
                    "action": {"type": "message", "label": "สวัสดิการ", "text": "สวัสดิการมีอะไรบ้าง?"}
                },
                {
                    "type": "action",
                    "action": {"type": "message", "label": "ติดต่อ HR", "text": "ขอติดต่อ HR"}
                },
            ]
        }
    )
```

### ค่าใช้จ่าย LINE

```
LINE Official Account Plans:
  Free    : 200 messages/เดือน  → ฟรี (ทดสอบ)
  Light   : 5,000 messages/เดือน → ~500 บาท/เดือน
  Standard: 25,000 messages/เดือน → ~1,500 บาท/เดือน
  
  * message = 1 reply (Flex Message นับ 1)
  * Webhook รับ message ไม่เสียเงิน
  * Internal use 50 คน ใช้ Light plan พอ
```

---

## 3. Slack Bot

### เหมาะกับ
- บริษัทที่ใช้ Slack เป็นหลัก
- Internal knowledge bot
- ถามตอบในแต่ละ channel ได้

### สิ่งที่ต้องเตรียม
1. สร้าง Slack App ที่ [api.slack.com/apps](https://api.slack.com/apps)
2. เปิด Event Subscriptions + Bot Token Scopes
3. Bot Token (xoxb-xxx)

### Implementation

```python
# pip install slack-bolt httpx

from slack_bolt import App
from slack_bolt.adapter.fastapi import SlackRequestHandler
from fastapi import FastAPI, Request
import httpx

# === Slack Config ===
slack_app = App(
    token=os.getenv("SLACK_BOT_TOKEN"),
    signing_secret=os.getenv("SLACK_SIGNING_SECRET"),
)

AGENT_API_URL = "http://localhost:8000/chat"

# === Handle Direct Messages ===
@slack_app.event("message")
def handle_message(event, say, client):
    # ไม่ตอบ bot messages (ป้องกัน loop)
    if event.get("bot_id"):
        return
    
    user_id = event["user"]
    text = event.get("text", "")
    channel = event["channel"]
    
    # แสดง typing indicator
    # (Slack ไม่มี typing API แต่ใช้ emoji react แทนได้)
    client.reactions_add(channel=channel, name="hourglass_flowing_sand", timestamp=event["ts"])
    
    # เรียก RAG Agent
    with httpx.Client() as http_client:
        response = http_client.post(
            AGENT_API_URL,
            json={
                "message": text,
                "session_id": f"slack_{user_id}_{channel}",
            },
            timeout=30.0,
        )
        data = response.json()
    
    # ลบ typing indicator
    client.reactions_remove(channel=channel, name="hourglass_flowing_sand", timestamp=event["ts"])
    
    # ตอบกลับ
    answer = data["answer"]
    sources = data.get("sources", [])
    
    blocks = [
        {
            "type": "section",
            "text": {"type": "mrkdwn", "text": answer}
        }
    ]
    
    if sources:
        source_text = "\n".join([f"• _{s.get('title', 'N/A')}_" for s in sources[:3]])
        blocks.append({"type": "divider"})
        blocks.append({
            "type": "context",
            "elements": [
                {"type": "mrkdwn", "text": f"📎 *อ้างอิง:*\n{source_text}"}
            ]
        })
    
    say(blocks=blocks, text=answer)


# === Handle @mention in channels ===
@slack_app.event("app_mention")
def handle_mention(event, say, client):
    text = event.get("text", "").split(">", 1)[-1].strip()  # ตัด @mention ออก
    user_id = event["user"]
    channel = event["channel"]
    thread_ts = event.get("thread_ts", event["ts"])  # ตอบใน thread
    
    with httpx.Client() as http_client:
        response = http_client.post(
            AGENT_API_URL,
            json={
                "message": text,
                "session_id": f"slack_{channel}_{thread_ts}",
            },
            timeout=30.0,
        )
        data = response.json()
    
    say(text=data["answer"], thread_ts=thread_ts)


# === Slash Command ===
@slack_app.command("/ask")
def handle_ask_command(ack, command, respond):
    ack()  # ต้อง acknowledge ภายใน 3 วินาที
    
    question = command["text"]
    user_id = command["user_id"]
    
    with httpx.Client() as http_client:
        response = http_client.post(
            AGENT_API_URL,
            json={
                "message": question,
                "session_id": f"slack_cmd_{user_id}",
            },
            timeout=30.0,
        )
        data = response.json()
    
    respond({
        "response_type": "in_channel",  # หรือ "ephemeral" (เห็นแค่คนถาม)
        "text": f"*Q:* {question}\n*A:* {data['answer']}"
    })


# === Mount to FastAPI ===
fastapi_app = FastAPI()
slack_handler = SlackRequestHandler(slack_app)

@fastapi_app.post("/webhook/slack/events")
async def slack_events(request: Request):
    return await slack_handler.handle(request)

@fastapi_app.post("/webhook/slack/commands")
async def slack_commands(request: Request):
    return await slack_handler.handle(request)
```

### ค่าใช้จ่าย Slack

```
Slack App: ฟรี (ไม่มีค่า bot)
Slack Workspace:
  Free   : ฟรี (จำกัด history 90 วัน)
  Pro    : $8.75/user/เดือน
  Business: $15/user/เดือน
```

---

## 4. Discord Bot

### เหมาะกับ
- ทีมที่ใช้ Discord (tech teams, gaming companies)
- Community support bot

### Implementation

```python
# pip install discord.py httpx

import discord
from discord.ext import commands
import httpx

AGENT_API_URL = "http://localhost:8000/chat"

# === Bot Setup ===
intents = discord.Intents.default()
intents.message_content = True
bot = commands.Bot(command_prefix="!", intents=intents)

# === ตอบทุก message ใน designated channel ===
ALLOWED_CHANNELS = ["ask-suki-bot", "general"]

@bot.event
async def on_message(message):
    if message.author.bot:
        return
    if message.channel.name not in ALLOWED_CHANNELS:
        await bot.process_commands(message)
        return
    
    async with message.channel.typing():
        async with httpx.AsyncClient() as client:
            response = await client.post(
                AGENT_API_URL,
                json={
                    "message": message.content,
                    "session_id": f"discord_{message.author.id}",
                },
                timeout=30.0,
            )
            data = response.json()
    
    # สร้าง Embed (สวยกว่า text ธรรมดา)
    embed = discord.Embed(
        title="🤖 Suki Bot",
        description=data["answer"],
        color=discord.Color.green()
    )
    
    sources = data.get("sources", [])
    if sources:
        source_text = "\n".join([f"• {s.get('title', 'N/A')}" for s in sources[:3]])
        embed.add_field(name="📎 อ้างอิง", value=source_text, inline=False)
    
    await message.reply(embed=embed)
    await bot.process_commands(message)

# === Slash Command ===
@bot.tree.command(name="ask", description="ถาม Suki Bot")
async def ask(interaction: discord.Interaction, question: str):
    await interaction.response.defer()
    
    async with httpx.AsyncClient() as client:
        response = await client.post(
            AGENT_API_URL,
            json={
                "message": question,
                "session_id": f"discord_{interaction.user.id}",
            },
            timeout=30.0,
        )
        data = response.json()
    
    await interaction.followup.send(f"**Q:** {question}\n**A:** {data['answer']}")

bot.run(os.getenv("DISCORD_BOT_TOKEN"))
```

---

## 5. Web Chat Widget

### เหมาะกับ
- ฝังในเว็บไซต์บริษัท / intranet
- Customer support widget
- ใครก็ใช้ได้ ไม่ต้องติดตั้งอะไร

### Option A: สร้างเอง (React)

```tsx
// ChatWidget.tsx — ฝังในเว็บไซต์ได้เลย
import React, { useState, useRef, useEffect } from 'react';

interface Message {
  role: 'user' | 'assistant';
  content: string;
  sources?: { title: string }[];
}

const AGENT_API_URL = 'https://your-agent-api.com/chat';

export default function ChatWidget() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState('');
  const [loading, setLoading] = useState(false);
  const [isOpen, setIsOpen] = useState(false);
  const [sessionId] = useState(`web_${Date.now()}`);
  const messagesEnd = useRef<HTMLDivElement>(null);

  useEffect(() => {
    messagesEnd.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages]);

  const sendMessage = async () => {
    if (!input.trim() || loading) return;

    const userMsg: Message = { role: 'user', content: input };
    setMessages(prev => [...prev, userMsg]);
    setInput('');
    setLoading(true);

    try {
      const res = await fetch(AGENT_API_URL, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          message: input,
          session_id: sessionId,
        }),
      });
      const data = await res.json();
      
      setMessages(prev => [...prev, {
        role: 'assistant',
        content: data.answer,
        sources: data.sources,
      }]);
    } catch {
      setMessages(prev => [...prev, {
        role: 'assistant',
        content: 'ขออภัย เกิดข้อผิดพลาด กรุณาลองใหม่',
      }]);
    }

    setLoading(false);
  };

  // ... render UI (ย่อไว้)
  // Floating button มุมขวาล่าง → กดเปิด chat window
}
```

### Option B: ใช้ Library สำเร็จรูป

```
Libraries ยอดนิยม:
  - Chatbot UI (open source, Next.js): github.com/mckaywrigley/chatbot-ui
  - Botpress Webchat: ฝัง iframe ได้เลย
  - Chainlit: pip install chainlit (UI สำหรับ LangChain)
  - Streamlit: pip install streamlit (สร้าง UI เร็วมาก)
  - Gradio: pip install gradio (สร้าง UI เร็วมาก)
```

### Chainlit (เร็วที่สุดในการ setup)

```python
# pip install chainlit httpx

# app_chainlit.py
import chainlit as cl
import httpx

AGENT_API_URL = "http://localhost:8000/chat"

@cl.on_message
async def on_message(message: cl.Message):
    session_id = cl.user_session.get("id")
    
    async with httpx.AsyncClient() as client:
        response = await client.post(
            AGENT_API_URL,
            json={
                "message": message.content,
                "session_id": f"chainlit_{session_id}",
            },
            timeout=30.0,
        )
        data = response.json()
    
    # ส่งคำตอบ
    answer = data["answer"]
    sources = data.get("sources", [])
    
    if sources:
        source_text = "\n".join([f"- {s.get('title', 'N/A')}" for s in sources[:3]])
        answer += f"\n\n---\n📎 **อ้างอิง:**\n{source_text}"
    
    await cl.Message(content=answer).send()

# รัน: chainlit run app_chainlit.py
# เปิด http://localhost:8000
```

### Streamlit (Prototype เร็วมาก)

```python
# pip install streamlit httpx

# app_streamlit.py
import streamlit as st
import httpx

st.title("🤖 Suki Bot")
st.caption("AI Assistant ของ Sellsuki — ถามได้ทุกเรื่องเกี่ยวกับบริษัท")

AGENT_API_URL = "http://localhost:8000/chat"

if "messages" not in st.session_state:
    st.session_state.messages = []
if "session_id" not in st.session_state:
    st.session_state.session_id = f"streamlit_{id(st.session_state)}"

# แสดงประวัติ
for msg in st.session_state.messages:
    with st.chat_message(msg["role"]):
        st.write(msg["content"])

# รับ input
if prompt := st.chat_input("ถามอะไรก็ได้..."):
    st.session_state.messages.append({"role": "user", "content": prompt})
    with st.chat_message("user"):
        st.write(prompt)
    
    with st.chat_message("assistant"):
        with st.spinner("กำลังค้นหา..."):
            response = httpx.post(
                AGENT_API_URL,
                json={
                    "message": prompt,
                    "session_id": st.session_state.session_id,
                },
                timeout=30.0,
            )
            data = response.json()
            st.write(data["answer"])
            
            if data.get("sources"):
                with st.expander("📎 แหล่งอ้างอิง"):
                    for s in data["sources"]:
                        st.write(f"- {s.get('title', 'N/A')}")
    
    st.session_state.messages.append({"role": "assistant", "content": data["answer"]})

# รัน: streamlit run app_streamlit.py
```

---

## 6. Facebook Messenger

### เหมาะกับ
- Customer support สำหรับลูกค้าที่ติดต่อผ่าน Facebook Page
- Sellsuki อาจมี Facebook Page สำหรับลูกค้าอยู่แล้ว

### สิ่งที่ต้องเตรียม
1. Facebook Page
2. Meta Developer App (developers.facebook.com)
3. Page Access Token
4. Verify Token

### Implementation

```python
# pip install fastapi httpx

from fastapi import FastAPI, Request, Query
import httpx, os, hmac, hashlib

app = FastAPI()

PAGE_ACCESS_TOKEN = os.getenv("FB_PAGE_ACCESS_TOKEN")
VERIFY_TOKEN = os.getenv("FB_VERIFY_TOKEN")
AGENT_API_URL = "http://localhost:8000/chat"

# === Webhook Verification (GET) ===
@app.get("/webhook/messenger")
async def verify(
    hub_mode: str = Query(None, alias="hub.mode"),
    hub_token: str = Query(None, alias="hub.verify_token"),
    hub_challenge: str = Query(None, alias="hub.challenge"),
):
    if hub_mode == "subscribe" and hub_token == VERIFY_TOKEN:
        return int(hub_challenge)
    return {"error": "verification failed"}

# === Receive Messages (POST) ===
@app.post("/webhook/messenger")
async def receive_message(request: Request):
    body = await request.json()
    
    for entry in body.get("entry", []):
        for event in entry.get("messaging", []):
            sender_id = event["sender"]["id"]
            
            if "message" in event and "text" in event["message"]:
                user_text = event["message"]["text"]
                
                # เรียก RAG Agent
                async with httpx.AsyncClient() as client:
                    response = await client.post(
                        AGENT_API_URL,
                        json={
                            "message": user_text,
                            "session_id": f"fb_{sender_id}",
                        },
                        timeout=30.0,
                    )
                    data = response.json()
                
                # ส่งกลับ
                await send_message(sender_id, data["answer"])
    
    return {"status": "ok"}

async def send_message(recipient_id: str, text: str):
    """ส่งข้อความกลับไปทาง Messenger"""
    async with httpx.AsyncClient() as client:
        await client.post(
            "https://graph.facebook.com/v18.0/me/messages",
            params={"access_token": PAGE_ACCESS_TOKEN},
            json={
                "recipient": {"id": recipient_id},
                "message": {"text": text[:2000]},  # FB limit 2000 chars
            }
        )
```

---

## 7. Microsoft Teams

### เหมาะกับ
- องค์กรที่ใช้ Microsoft 365
- Internal bot สำหรับพนักงาน

### วิธีทำ

```
2 วิธีหลัก:

Option A: Microsoft Bot Framework (official)
  - ลงทะเบียนที่ Azure Bot Service
  - ใช้ Bot Framework SDK (Python/Node.js)
  - Deploy บน Azure

Option B: Power Virtual Agents + HTTP connector
  - สร้างผ่าน Power Platform (low-code)
  - เรียก RAG Agent API ผ่าน HTTP action
  - ไม่ต้องเขียนโค้ดมาก
```

```python
# pip install botbuilder-core botbuilder-integration-aiohttp aiohttp httpx

from botbuilder.core import ActivityHandler, TurnContext
from botbuilder.schema import Activity
import httpx

AGENT_API_URL = "http://localhost:8000/chat"

class SukiBotTeams(ActivityHandler):
    async def on_message_activity(self, turn_context: TurnContext):
        user_id = turn_context.activity.from_property.id
        user_text = turn_context.activity.text
        
        # แสดง typing
        await turn_context.send_activity(Activity(type="typing"))
        
        # เรียก RAG Agent
        async with httpx.AsyncClient() as client:
            response = await client.post(
                AGENT_API_URL,
                json={
                    "message": user_text,
                    "session_id": f"teams_{user_id}",
                },
                timeout=30.0,
            )
            data = response.json()
        
        # ตอบกลับด้วย Adaptive Card (สวย)
        card = {
            "type": "AdaptiveCard",
            "version": "1.4",
            "body": [
                {
                    "type": "TextBlock",
                    "text": "🤖 Suki Bot",
                    "weight": "Bolder",
                    "size": "Medium"
                },
                {
                    "type": "TextBlock",
                    "text": data["answer"],
                    "wrap": True
                }
            ]
        }
        
        if data.get("sources"):
            source_text = "\n".join([f"• {s.get('title', '')}" for s in data["sources"][:3]])
            card["body"].append({
                "type": "TextBlock",
                "text": f"📎 อ้างอิง:\n{source_text}",
                "size": "Small",
                "isSubtle": True,
                "wrap": True,
            })
        
        message = Activity(
            type="message",
            attachments=[{
                "contentType": "application/vnd.microsoft.card.adaptive",
                "content": card
            }]
        )
        await turn_context.send_activity(message)
```

---

## 8. Telegram Bot

### เหมาะกับ
- ทีม tech ที่ใช้ Telegram
- ฟรีไม่จำกัด messages
- Bot API ง่ายมาก

### Implementation

```python
# pip install python-telegram-bot httpx

from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler, filters
import httpx

AGENT_API_URL = "http://localhost:8000/chat"
BOT_TOKEN = os.getenv("TELEGRAM_BOT_TOKEN")

# === Handle Messages ===
async def handle_message(update: Update, context):
    user_id = update.effective_user.id
    user_text = update.message.text
    
    # แสดง typing
    await update.effective_chat.send_action("typing")
    
    async with httpx.AsyncClient() as client:
        response = await client.post(
            AGENT_API_URL,
            json={
                "message": user_text,
                "session_id": f"telegram_{user_id}",
            },
            timeout=30.0,
        )
        data = response.json()
    
    answer = data["answer"]
    sources = data.get("sources", [])
    
    if sources:
        source_text = "\n".join([f"• _{s.get('title', 'N/A')}_" for s in sources[:3]])
        answer += f"\n\n📎 *อ้างอิง:*\n{source_text}"
    
    await update.message.reply_text(answer, parse_mode="Markdown")

# === /start Command ===
async def start(update: Update, context):
    await update.message.reply_text(
        "สวัสดีครับ! 🤖 ผมคือ Suki Bot\n"
        "ถามอะไรก็ได้เกี่ยวกับบริษัท Sellsuki ครับ"
    )

# === Run ===
app = Application.builder().token(BOT_TOKEN).build()
app.add_handler(CommandHandler("start", start))
app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
app.run_polling()
```

---

## 9. WhatsApp Business

### เหมาะกับ
- ลูกค้าต่างประเทศ
- ธุรกิจที่ใช้ WhatsApp เป็นหลัก

### วิธีทำ

```
ใช้ผ่าน Meta Cloud API (official):
1. สร้าง Meta Business Account
2. สร้าง WhatsApp Business App
3. ได้ Phone Number ID + Access Token
4. Setup webhook

หรือใช้ผ่าน Twilio (ง่ายกว่า):
1. สมัคร Twilio
2. เปิด WhatsApp Sandbox
3. Webhook URL เหมือนกับ Twilio SMS
```

```python
# ผ่าน Twilio
# pip install twilio fastapi

from twilio.rest import Client as TwilioClient
from twilio.twiml.messaging_response import MessagingResponse
from fastapi import FastAPI, Form
import httpx

AGENT_API_URL = "http://localhost:8000/chat"

app = FastAPI()

@app.post("/webhook/whatsapp")
async def whatsapp_webhook(
    Body: str = Form(...),
    From: str = Form(...),
    To: str = Form(...),
):
    user_phone = From.replace("whatsapp:", "")
    user_text = Body
    
    async with httpx.AsyncClient() as client:
        response = await client.post(
            AGENT_API_URL,
            json={
                "message": user_text,
                "session_id": f"whatsapp_{user_phone}",
            },
            timeout=30.0,
        )
        data = response.json()
    
    resp = MessagingResponse()
    resp.message(data["answer"])
    return str(resp)
```

### ค่าใช้จ่าย WhatsApp

```
Meta Cloud API:
  - Business-initiated: $0.0147-0.1099 / conversation (ขึ้นกับประเทศ)
  - User-initiated: ถูกกว่า
  - 1,000 conversations/เดือน ฟรี

Twilio:
  - $0.005/message (ส่ง) + $0.005/message (รับ)
  - WhatsApp: $0.005 + Meta conversation fee
```

---

## 10. Mobile App

### Option A: React Native

```typescript
// ใช้ react-native-gifted-chat
// npm install react-native-gifted-chat

import React, { useState, useCallback } from 'react';
import { GiftedChat, IMessage } from 'react-native-gifted-chat';

const AGENT_API_URL = 'https://your-agent-api.com/chat';

export default function ChatScreen() {
  const [messages, setMessages] = useState<IMessage[]>([]);
  const [sessionId] = useState(`mobile_${Date.now()}`);

  const onSend = useCallback(async (newMessages: IMessage[]) => {
    setMessages(prev => GiftedChat.append(prev, newMessages));
    
    const userText = newMessages[0].text;
    
    try {
      const res = await fetch(AGENT_API_URL, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          message: userText,
          session_id: sessionId,
        }),
      });
      const data = await res.json();
      
      const botMessage: IMessage = {
        _id: Date.now(),
        text: data.answer,
        createdAt: new Date(),
        user: { _id: 2, name: 'Suki Bot', avatar: '🤖' },
      };
      
      setMessages(prev => GiftedChat.append(prev, [botMessage]));
    } catch {
      // error handling
    }
  }, []);

  return (
    <GiftedChat
      messages={messages}
      onSend={onSend}
      user={{ _id: 1 }}
      placeholder="ถามอะไรก็ได้..."
    />
  );
}
```

### Option B: Flutter

```dart
// pubspec.yaml: dash_chat_2: ^0.0.21, http: ^1.2.0

import 'package:dash_chat_2/dash_chat_2.dart';
import 'package:http/http.dart' as http;
import 'dart:convert';

class ChatScreen extends StatefulWidget { ... }

class _ChatScreenState extends State<ChatScreen> {
  List<ChatMessage> messages = [];
  final user = ChatUser(id: '1', firstName: 'User');
  final bot = ChatUser(id: '2', firstName: 'Suki Bot');

  Future<void> sendMessage(ChatMessage message) async {
    setState(() => messages.insert(0, message));

    final response = await http.post(
      Uri.parse('https://your-agent-api.com/chat'),
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode({
        'message': message.text,
        'session_id': 'flutter_${user.id}',
      }),
    );
    
    final data = jsonDecode(response.body);
    
    setState(() {
      messages.insert(0, ChatMessage(
        user: bot,
        text: data['answer'],
        createdAt: DateTime.now(),
      ));
    });
  }

  @override
  Widget build(BuildContext context) {
    return DashChat(
      currentUser: user,
      onSend: sendMessage,
      messages: messages,
    );
  }
}
```

---

## 11. Internal Dashboard (Next.js)

### เหมาะกับ
- Admin dashboard สำหรับจัดการ knowledge base
- ดู analytics การใช้งาน
- Chat + Document management ในที่เดียว

```typescript
// app/page.tsx (Next.js App Router)
'use client';

import { useState } from 'react';

export default function Dashboard() {
  const [messages, setMessages] = useState<
    { role: string; content: string }[]
  >([]);
  const [input, setInput] = useState('');

  const sendMessage = async () => {
    if (!input.trim()) return;
    
    setMessages(prev => [...prev, { role: 'user', content: input }]);
    setInput('');

    const res = await fetch('/api/chat', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ message: input }),
    });
    
    const data = await res.json();
    setMessages(prev => [...prev, { role: 'assistant', content: data.answer }]);
  };

  return (
    <div className="flex h-screen">
      {/* Sidebar */}
      <div className="w-64 bg-gray-900 text-white p-4">
        <h2 className="text-xl font-bold">Suki Bot Admin</h2>
        <nav className="mt-8 space-y-2">
          <a href="/chat">💬 Chat</a>
          <a href="/documents">📄 Documents</a>
          <a href="/analytics">📊 Analytics</a>
          <a href="/settings">⚙️ Settings</a>
        </nav>
      </div>
      
      {/* Chat Area */}
      <div className="flex-1 flex flex-col">
        <div className="flex-1 overflow-y-auto p-4 space-y-4">
          {messages.map((msg, i) => (
            <div key={i} className={
              msg.role === 'user' ? 'text-right' : 'text-left'
            }>
              <span className={`inline-block p-3 rounded-lg ${
                msg.role === 'user' 
                  ? 'bg-blue-600 text-white' 
                  : 'bg-gray-200'
              }`}>
                {msg.content}
              </span>
            </div>
          ))}
        </div>
        
        <div className="p-4 border-t flex gap-2">
          <input
            value={input}
            onChange={e => setInput(e.target.value)}
            onKeyDown={e => e.key === 'Enter' && sendMessage()}
            className="flex-1 border rounded-lg p-3"
            placeholder="ถามอะไรก็ได้..."
          />
          <button onClick={sendMessage} className="bg-blue-600 text-white px-6 rounded-lg">
            ส่ง
          </button>
        </div>
      </div>
    </div>
  );
}
```

---

## 12. Google Chat

### เหมาะกับ
- องค์กรที่ใช้ Google Workspace

```python
# Google Chat Bot ใช้ Google Cloud Functions

from flask import Flask, request, jsonify
import httpx

app = Flask(__name__)
AGENT_API_URL = "http://localhost:8000/chat"

@app.route("/webhook/google-chat", methods=["POST"])
def google_chat():
    event = request.json
    
    if event.get("type") == "MESSAGE":
        user_text = event["message"]["text"]
        user_id = event["user"]["name"]
        
        with httpx.Client() as client:
            response = client.post(
                AGENT_API_URL,
                json={
                    "message": user_text,
                    "session_id": f"gchat_{user_id}",
                },
                timeout=30.0,
            )
            data = response.json()
        
        return jsonify({
            "text": data["answer"]
        })
    
    return jsonify({"text": "สวัสดีครับ! ถามอะไรก็ได้"})
```

---

## 13. Email (Auto-reply)

### เหมาะกับ
- ตอบ email อัตโนมัติ (ส่งไปที่ ask@sellsuki.com)
- ลูกค้าหรือพนักงานที่ไม่สะดวกใช้ chat

```python
# ใช้ Gmail API หรือ IMAP/SMTP

import imaplib
import smtplib
import email
from email.mime.text import MIMEText
import httpx
import time

AGENT_API_URL = "http://localhost:8000/chat"
EMAIL = "ask@sellsuki.com"
PASSWORD = "app_password"

def check_and_reply():
    # เชื่อมต่อ IMAP
    mail = imaplib.IMAP4_SSL("imap.gmail.com")
    mail.login(EMAIL, PASSWORD)
    mail.select("inbox")
    
    # ค้นหา email ที่ยังไม่อ่าน
    _, messages = mail.search(None, "UNSEEN")
    
    for msg_id in messages[0].split():
        _, msg_data = mail.fetch(msg_id, "(RFC822)")
        msg = email.message_from_bytes(msg_data[0][1])
        
        sender = email.utils.parseaddr(msg["From"])[1]
        subject = msg["Subject"]
        
        # Extract body
        body = ""
        if msg.is_multipart():
            for part in msg.walk():
                if part.get_content_type() == "text/plain":
                    body = part.get_payload(decode=True).decode()
                    break
        else:
            body = msg.get_payload(decode=True).decode()
        
        # เรียก RAG Agent
        with httpx.Client() as client:
            response = client.post(
                AGENT_API_URL,
                json={
                    "message": body,
                    "session_id": f"email_{sender}",
                },
                timeout=30.0,
            )
            data = response.json()
        
        # ส่ง reply
        reply = MIMEText(
            f"สวัสดีครับ,\n\n{data['answer']}\n\n"
            f"---\n🤖 ตอบโดย Suki Bot (AI Assistant)\n"
            f"หากต้องการพูดคุยกับเจ้าหน้าที่ กรุณาตอบกลับอีเมลนี้พร้อมระบุว่า 'ขอคุยกับเจ้าหน้าที่'"
        )
        reply["Subject"] = f"Re: {subject}"
        reply["From"] = EMAIL
        reply["To"] = sender
        
        with smtplib.SMTP_SSL("smtp.gmail.com", 465) as smtp:
            smtp.login(EMAIL, PASSWORD)
            smtp.send_message(reply)
    
    mail.logout()

# รันทุก 30 วินาที
while True:
    check_and_reply()
    time.sleep(30)
```

---

## 14. Voice Assistant (โทรศัพท์)

### เหมาะกับ
- Call center automation
- พนักงานที่อยู่หน้างาน ไม่สะดวกพิมพ์

```python
# ใช้ Twilio Voice + Speech-to-Text + TTS
# pip install twilio fastapi

from twilio.twiml.voice_response import VoiceResponse, Gather
from fastapi import FastAPI, Form
import httpx

app = FastAPI()
AGENT_API_URL = "http://localhost:8000/chat"

@app.post("/webhook/voice/incoming")
async def incoming_call():
    """รับสายโทรเข้า"""
    response = VoiceResponse()
    response.say(
        "สวัสดีครับ ผมคือ Suki Bot ถามอะไรก็ได้เกี่ยวกับบริษัทครับ",
        language="th-TH"
    )
    
    gather = Gather(
        input="speech",
        language="th-TH",
        action="/webhook/voice/process",
        timeout=5,
        speech_timeout="auto",
    )
    gather.say("เชิญถามได้เลยครับ", language="th-TH")
    response.append(gather)
    
    return str(response)

@app.post("/webhook/voice/process")
async def process_speech(SpeechResult: str = Form("")):
    """ประมวลผลเสียงที่ถูกแปลงเป็นข้อความ"""
    response = VoiceResponse()
    
    if not SpeechResult:
        response.say("ขออภัย ไม่ได้ยินคำถาม กรุณาลองใหม่", language="th-TH")
        response.redirect("/webhook/voice/incoming")
        return str(response)
    
    # เรียก RAG Agent
    async with httpx.AsyncClient() as client:
        api_response = await client.post(
            AGENT_API_URL,
            json={
                "message": SpeechResult,
                "session_id": f"voice_{SpeechResult[:10]}",
            },
            timeout=30.0,
        )
        data = api_response.json()
    
    # อ่านคำตอบ
    response.say(data["answer"], language="th-TH")
    
    # ถามว่ามีอะไรอีกมั้ย
    gather = Gather(
        input="speech",
        language="th-TH",
        action="/webhook/voice/process",
        timeout=5,
    )
    gather.say("มีอะไรถามเพิ่มมั้ยครับ?", language="th-TH")
    response.append(gather)
    
    response.say("ขอบคุณครับ สวัสดีครับ", language="th-TH")
    
    return str(response)
```

---

## 15. API สำหรับ Third-party

### เปิด API ให้ระบบอื่นเรียกใช้

```python
# อยู่ใน RAG Agent API อยู่แล้ว (FastAPI)
# เพิ่ม API Key authentication

from fastapi import FastAPI, Depends, HTTPException, Security
from fastapi.security import APIKeyHeader

app = FastAPI()

API_KEY_HEADER = APIKeyHeader(name="X-API-Key")
VALID_API_KEYS = {
    "sk-sellsuki-internal-001": {"name": "Sellsuki Web", "rate_limit": 100},
    "sk-sellsuki-line-002": {"name": "LINE Bot", "rate_limit": 500},
    "sk-sellsuki-slack-003": {"name": "Slack Bot", "rate_limit": 500},
}

async def verify_api_key(api_key: str = Security(API_KEY_HEADER)):
    if api_key not in VALID_API_KEYS:
        raise HTTPException(status_code=403, detail="Invalid API key")
    return VALID_API_KEYS[api_key]

@app.post("/api/v1/chat")
async def chat(
    request: ChatRequest,
    client_info: dict = Depends(verify_api_key)
):
    # ... เรียก RAG Agent เหมือนเดิม
    pass
```

---

## 16. Low-code Platforms

### ไม่อยากเขียนโค้ดเอง? ใช้ platform เหล่านี้ได้

```
┌─────────────────────────────────────────────────────────────────┐
│ Platform       │ ช่องทาง               │ ราคา        │ ความง่าย │
├─────────────────────────────────────────────────────────────────┤
│ Botpress       │ Web, LINE, FB, WA,    │ Free tier   │ ★★★★★  │
│                │ Slack, Teams, Telegram │ มี          │          │
├────────────────┼───────────────────────┼─────────────┼──────────┤
│ Dialogflow CX  │ Web, LINE, FB, WA,   │ Free tier   │ ★★★★☆  │
│ (Google)       │ Telegram, Phone       │ มี          │          │
├────────────────┼───────────────────────┼─────────────┼──────────┤
│ Dify           │ Web (API สำหรับอื่น)   │ Open source │ ★★★★★  │
│                │                       │ / Cloud     │          │
├────────────────┼───────────────────────┼─────────────┼──────────┤
│ Flowise        │ Web (API สำหรับอื่น)   │ Open source │ ★★★★☆  │
├────────────────┼───────────────────────┼─────────────┼──────────┤
│ Voiceflow      │ Web, WA, Alexa       │ Free tier   │ ★★★★★  │
│                │                       │ มี          │          │
├────────────────┼───────────────────────┼─────────────┼──────────┤
│ Typebot        │ Web, WA              │ Open source │ ★★★★☆  │
├────────────────┼───────────────────────┼─────────────┼──────────┤
│ n8n + AI nodes │ ทุกช่องทาง (workflow)  │ Open source │ ★★★★☆  │
└────────────────┴───────────────────────┴─────────────┴──────────┘
```

### Dify (แนะนำสำหรับทดสอบเร็ว)

```bash
# Self-host Dify
git clone https://github.com/langgenius/dify.git
cd dify/docker
cp .env.example .env
docker compose up -d

# เปิด http://localhost:3000
# 1. สร้าง Knowledge Base → อัปโหลดเอกสาร
# 2. สร้าง App → เลือก "Chat App"
# 3. เชื่อม Knowledge Base
# 4. Publish → ได้ API endpoint เอาไปต่อ LINE, Slack ได้
```

### n8n (Workflow Automation)

```bash
# Self-host n8n
docker run -d --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  n8nio/n8n

# เปิด http://localhost:5678
# สร้าง workflow:
# LINE Webhook → AI Agent node → pgvector → Reply to LINE
# ลาก drop ไม่ต้องเขียนโค้ด
```

---

## 17. เปรียบเทียบทุกช่องทาง

```
┌──────────────┬────────┬──────────┬─────────┬────────┬────────────────┐
│ ช่องทาง       │ ค่าใช้  │ ความง่าย  │ UX      │ ไทย    │ เหมาะกับ        │
│              │ จ่าย   │ ในการทำ   │         │ รองรับ  │                │
├──────────────┼────────┼──────────┼─────────┼────────┼────────────────┤
│ LINE         │ ถูก    │ ★★★★☆   │ ★★★★★  │ ★★★★★ │ ไทย, internal  │
│ Slack        │ ฟรี    │ ★★★★★   │ ★★★★☆  │ ★★★☆☆ │ Tech teams     │
│ Discord      │ ฟรี    │ ★★★★★   │ ★★★★☆  │ ★★★☆☆ │ Tech/gaming    │
│ Web Chat     │ ฟรี    │ ★★★☆☆   │ ★★★★★  │ ★★★★★ │ เว็บไซต์/intra │
│ FB Messenger │ ฟรี    │ ★★★★☆   │ ★★★★☆  │ ★★★★☆ │ ลูกค้า FB      │
│ MS Teams     │ กลาง   │ ★★★☆☆   │ ★★★★☆  │ ★★★☆☆ │ องค์กร MS365   │
│ Telegram     │ ฟรี    │ ★★★★★   │ ★★★★☆  │ ★★★☆☆ │ Tech teams     │
│ WhatsApp     │ มีค่า  │ ★★★☆☆   │ ★★★★★  │ ★★★★☆ │ ลูกค้าต่างชาติ  │
│ Mobile App   │ สูง    │ ★★☆☆☆   │ ★★★★★  │ ★★★★★ │ ต้องการ app    │
│ Dashboard    │ กลาง   │ ★★★☆☆   │ ★★★★★  │ ★★★★★ │ Admin/internal │
│ Google Chat  │ ฟรี    │ ★★★★☆   │ ★★★☆☆  │ ★★★☆☆ │ Google WS org  │
│ Email        │ ฟรี    │ ★★★★☆   │ ★★★☆☆  │ ★★★★★ │ Formal, async  │
│ Voice/Phone  │ มีค่า  │ ★★☆☆☆   │ ★★★☆☆  │ ★★★☆☆ │ Call center    │
│ Streamlit    │ ฟรี    │ ★★★★★   │ ★★★☆☆  │ ★★★★★ │ Prototype      │
│ Chainlit     │ ฟรี    │ ★★★★★   │ ★★★★☆  │ ★★★★★ │ Demo/internal  │
└──────────────┴────────┴──────────┴─────────┴────────┴────────────────┘
```

---

## 18. แนะนำสำหรับ Sellsuki

### Phase 1: เริ่มต้น (สัปดาห์ 1-2)

```
1. Streamlit หรือ Chainlit
   → ทำ prototype ทดสอบ RAG quality ก่อน
   → ใช้เวลาทำไม่ถึงชั่วโมง
   → ให้ทีมลองใช้ ปรับ prompt

2. Slack Bot (ถ้าบริษัทใช้ Slack)
   → ทำง่ายที่สุดในบรรดา production channels
   → ทีม dev ใช้ทดสอบได้เลย
```

### Phase 2: Production (สัปดาห์ 3-4)

```
3. LINE Official Account
   → ช่องทางหลักสำหรับพนักงานไทย
   → Rich Menu + Quick Reply ทำให้ UX ดีมาก
   → ทุกคนมี LINE อยู่แล้ว

4. Web Chat Widget
   → ฝังใน intranet/website
   → สำหรับคนที่ไม่สะดวกใช้ LINE
```

### Phase 3: ขยาย (สัปดาห์ 5+)

```
5. เพิ่มช่องทางตามความต้องการ:
   → Facebook Messenger (ถ้ามีลูกค้า FB)
   → Email auto-reply (ถ้ามี email ถามบ่อย)
   → Internal Dashboard (ถ้าต้องการ admin tools)

6. Analytics & Monitoring:
   → เก็บ log ทุก query + response
   → วิเคราะห์คำถามที่ถามบ่อย
   → ปรับปรุง knowledge base ตาม feedback
```

### Architecture แนะนำ

```
                    ┌──────────────────┐
                    │   RAG Agent API  │
                    │   (FastAPI)      │
                    │   + pgvector     │
                    └────────┬─────────┘
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
   ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
   │  LINE Bot   │   │  Slack Bot  │   │  Web Chat   │
   │  (หลัก)     │   │  (internal) │   │  (intranet) │
   └─────────────┘   └─────────────┘   └─────────────┘
   
   ทั้ง 3 ช่องทาง ใช้ RAG Agent API ตัวเดียวกัน
   เพิ่มช่องทางใหม่ = เพิ่มแค่ adapter/webhook handler
```
