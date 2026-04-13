---
title: "API Integration — เชื่อมต่อ RAG Agent กับ Channel ต่างๆ"
type: concept
tags: [api-integration, line, slack, webhook, rest-api, rag, channels]
sources: [wiki/sources/rag-client-channels-complete, wiki/sources/rag-integration-guide-complete, wiki/sources/openrag-integration-scenarios]
related: [wiki/concepts/api-design, wiki/concepts/rag-retrieval-augmented-generation, wiki/concepts/mcp-model-context-protocol]
created: 2026-04-14
updated: 2026-04-14
---

## สรุปสั้น

API Integration คือกระบวนการเชื่อมต่อ RAG Agent เข้ากับ channel ต่างๆ ที่ user ใช้งาน — ไม่ว่าจะเป็น LINE, Slack, Web Chat, Mobile App หรือ Internal Dashboard โดยทุก channel ส่ง request มาที่ `/chat` API เดียวกัน

## อธิบาย

### Architecture: 1 Agent, N Channels

```
LINE Bot ──────┐
Slack Bot ─────┤
Web Chat ──────┤──→  POST /chat  →  RAG Agent  →  Vector DB
Mobile App ────┤                        ↓
Dashboard ─────┘              อ่านเอกสาร, ตอบ user
```

ข้อดีของ pattern นี้: เพิ่ม channel ใหม่ไม่กระทบ Agent logic — แค่ wrap webhook ใหม่

## ประเด็นสำคัญ

### Channel Integration Patterns

**LINE Messaging API**
```python
@app.post("/webhook/line")
async def line_webhook(request: Request):
    body = await request.body()
    handler.handle(body.decode(), request.headers.get("x-line-signature"))

@handler.add(MessageEvent, message=TextMessageContent)
def handle_text(event):
    reply = agent.invoke({"input": event.message.text})["output"]
    line_bot_api.reply_message(event.reply_token, TextSendMessage(text=reply))
```

**Slack Slash Command**
```python
@app.post("/webhook/slack")
async def slack_command(text: str = Form(), user_id: str = Form()):
    result = await agent.ainvoke({"input": text, "user_id": user_id})
    return {"text": result["output"], "response_type": "ephemeral"}
```

**Web Chat (WebSocket / SSE)**
```python
@app.websocket("/ws/chat")
async def websocket_chat(ws: WebSocket):
    await ws.accept()
    while True:
        message = await ws.receive_text()
        async for chunk in agent.astream({"input": message}):
            await ws.send_text(chunk["output"])
```

### 14 Channels ที่ RAG Agent รองรับได้

| Channel | Protocol | ใช้สำหรับ |
|---------|---------|---------|
| LINE | Webhook + Reply Token | ลูกค้า/พนักงาน บน mobile |
| Slack | Slash Command + Events | Internal team |
| Web Chat | REST / SSE / WebSocket | เว็บไซต์ |
| Discord | Bot + Webhook | Community |
| Teams | Bot Framework | Enterprise |
| Telegram | Bot API | Group/Channel |
| Mobile App | REST API | iOS/Android native |
| Email | Inbound Parsing | Email-to-answer |
| Voice | STT → API → TTS | Call center |
| Claude Code | MCP Server | Developer tools |
| Dashboard | REST + WebSocket | Admin monitoring |
| WhatsApp | Cloud API | International |
| Google Chat | App webhook | G-Suite users |
| Messenger | Webhook | Facebook pages |

### MCP Integration (สำหรับ Developer Tools)

```json
{
  "mcpServers": {
    "sellsuki-kb": {
      "command": "node",
      "args": ["./mcp-server/index.js"],
      "env": {"RAG_API_URL": "http://localhost:8000"}
    }
  }
}
```

ทำให้ Claude Code หรือ Cursor สามารถ query knowledge base ของ Sellsuki ได้โดยตรง

## ตัวอย่าง / กรณีศึกษา

**Sellsuki HR Bot:** deploy เป็น LINE Bot ให้พนักงาน 500+ คนถามเรื่อง HR Policy ผ่าน LINE ที่ใช้อยู่แล้ว — ลด ticket ที่ส่งมา HR ลง ไม่ต้องเรียนรู้ระบบใหม่

**OpenRAG Integration:** รองรับ Google Drive Connector ที่ sync อัตโนมัติ — เอกสารใหม่ใน Drive จะถูก ingest โดย webhook trigger

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/api-design|API Design]] — ต้อง design API ให้ดีก่อน แล้ว integrate channel
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — RAG Agent คือ backend ที่ทุก channel เชื่อมมา
- [[wiki/concepts/mcp-model-context-protocol|MCP]] — MCP เป็น channel พิเศษสำหรับ developer tools (Claude Code, Cursor)

## แหล่งที่มา

- [[wiki/sources/rag-client-channels-complete|RAG Agent — ช่องทางเชื่อมต่อทั้งหมด]]
- [[wiki/sources/rag-integration-guide-complete|RAG Agent Integration — Deploy แล้วต่ออะไรได้บ้าง]]
- [[wiki/sources/openrag-integration-scenarios|OpenRAG — Integration Scenarios & Full Stack]]
