---
title: "RAG Agent — ช่องทางการเชื่อมต่อทั้งหมด"
type: source
source_file: raw/notes/rag-knowledge/05-rag-client-channels.md
tags: [rag, integration, channels, line, slack, api, bot]
related: [wiki/concepts/rag-retrieval-augmented-generation]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/rag-knowledge/05-rag-client-channels.md|Original file]]

## สรุป

คู่มือครบวิธีเชื่อมต่อ RAG Agent กับทุกช่องทาง — ครอบคลุม 14 ช่องทาง (LINE, Slack, Discord, Web Chat, Messenger, Teams, Telegram, WhatsApp, Mobile App, Dashboard, Google Chat, Email, Voice, API) พร้อม code ตัวอย่าง

## ประเด็นสำคัญ

### Architecture หลัก

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
   ┌─────────┼─────────┐
   │         │         │
┌──▼───┐ ┌──▼───┐ ┌──▼───┐
│ LINE │ │Slack │ │ Web  │ ...
└──────┘ └──────┘ └──────┘
```

**หลักการ**: ทุกช่องทางเชื่อมต่อกับ **RAG Agent API ตัวเดียวกัน** ผ่าน HTTP POST

### LINE Official Account

#### สิ่งที่ต้องเตรียม
1. LINE Official Account
2. Messaging API
3. Channel Access Token & Channel Secret

#### Architecture
```
User (LINE App) → LINE Platform → Your Server (Webhook)
→ RAG Agent API (/chat) → Your Server → LINE Platform → User
```

#### Implementation (สำคัญ)
```python
from fastapi import FastAPI, Request
from linebot.v3 import WebhookHandler
from linebot.v3.messaging import ApiClient, MessagingApi, ReplyMessageRequest

@app.post("/webhook/line")
async def line_webhook(request: Request):
    signature = request.headers.get("X-Line-Signature", "")
    body = (await request.body()).decode("utf-8")
    handler.handle(body, signature)
    return {"status": "ok"}

@handler.add(MessageEvent, message=TextMessageContent)
def handle_text_message(event):
    user_message = event.message.text

    # เรียก RAG Agent
    response = httpx.post(AGENT_API_URL, json={
        "message": user_message,
        "session_id": f"line_{event.source.user_id}",
    })

    data = response.json()
    answer = data["answer"]
    sources = data.get("sources", [])

    # สร้าง reply
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
```

#### ค่าใช้จ่าย LINE
```
Free    : 200 messages/เดือน → ฟรี (ทดสอบ)
Light   : 5,000 messages/เดือน → ~500 บาท/เดือน
Standard: 25,000 messages/เดือน → ~1,500 บาท/เดือน
```

### Slack Bot

#### สิ่งที่ต้องเตรียม
1. สร้าง Slack App ที่ api.slack.com/apps
2. เปิด Event Subscriptions + Bot Token Scopes
3. Bot Token (xoxb-xxx)

#### Implementation
```python
from slack_bolt import App
from slack_bolt.adapter.fastapi import SlackRequestHandler

slack_app = App(
    token=os.getenv("SLACK_BOT_TOKEN"),
    signing_secret=os.getenv("SLACK_SIGNING_SECRET"),
)

@slack_app.event("message")
def handle_message(event, say, client):
    # แสดง typing indicator
    client.reactions_add(channel=event["channel"],
                         name="hourglass_flowing_sand",
                         timestamp=event["ts"])

    # เรียก RAG Agent
    response = httpx.post(AGENT_API_URL, json={
        "message": event.get("text", ""),
        "session_id": f"slack_{event['user']}_{event['channel']}",
    })

    data = response.json()
    answer = data["answer"]
    sources = data.get("sources", [])

    # ตอบกลับ
    say(blocks=[{"type": "section", "text": {"type": "mrkdwn", "text": answer}}])
```

#### ค่าใช้จ่าย Slack
```
Slack App: ฟรี (ไม่มีค่า bot)
Slack Workspace:
  Free   : ฟรี (จำกัด history 90 วัน)
  Pro    : $8.75/user/เดือน
```

### ช่องทางอื่นๆ

| ช่องทาง | เหมาะกับ | ความยาก | ค่าใช้จ่าย |
|---------|----------|---------|------------|
| **Discord Bot** | Tech teams, Gaming communities | ง่าย | ฟรี |
| **Web Chat Widget** | Website internal/external | ง่าย | ฟรี |
| **Facebook Messenger** | External customers | ง่าย | ฟรี |
| **Microsoft Teams** | Enterprise internal | ปานกลาง | ต้องมี license |
| **Telegram Bot** | Global users | ง่าย | ฟรี |
| **WhatsApp Business** | Mobile-first markets | ปานกลาง | มีค่าใช้จ่าย |
| **Mobile App** | Native experience | ยาก | ตาม platform |
| **Internal Dashboard** | Admin/Analytics | ง่าย | ฟรี |
| **Google Chat** | Google Workspace | ปานกลาง | ฟรี (สำหรับ workspace) |
| **Email Auto-reply** | Async communication | ง่าย | ฟรี |
| **Voice Assistant** | Phone/Voice | ยาก | มีค่าใช้จ่าย |
| **API** | Third-party integration | ง่าย | ฟรี |

### เปรียบเทียบทุกช่องทาง

| ช่องทาง | Latency | UX | Cost | Implementation | แนะนำสำหรับ |
|---------|---------|-----|------|----------------|---------------|
| LINE | ต่ำ | ดีมาก | ต่ำ | ง่าย | **ไทย — แนะนำอย่างยิ่ง** |
| Slack | ต่ำ | ดี | ฟรี/ต่ำ | ง่าย | Tech companies |
| Web Chat | ต่ำ | ดี | ฟรี | ง่าย | Website |
| Discord | ต่ำ | ดี | ฟรี | ง่าย | Tech teams |
| Teams | ปานกลาง | ดี | สูง | ปานกลาง | Enterprise |
| Mobile App | ต่ำ | ดีที่สุด | สูง | ยาก | ถ้ามี resources |

### แนะนำสำหรับ Sellsuki

```
Priority 1 (ทันที):
  ✅ LINE Official Account — คนไทยใช้มากที่สุด

Priority 2 (สัปดาห์ 2-4):
  ✅ Slack Bot — สำหรับ internal team
  ✅ Web Chat Widget — สำหรับ website/knowledge base

Priority 3 (ถ้ามีเวลา):
  ⏸️ Mobile App — ถ้ามี resources
  ⏸️ Email Auto-reply — สำหรับ async
```

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG — Retrieval-Augmented Generation]]
- [[wiki/concepts/api-integration|API Integration]]
