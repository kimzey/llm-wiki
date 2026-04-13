---
title: "RAG Agent Integration — Deploy แล้วต่อกับอะไรได้บ้าง"
type: source
source_file: raw/notes/rag-knowledge/03-rag-integration-guide.md
tags: [rag, integration, api, mcp, ide, deployment]
related: [wiki/concepts/rag-retrieval-augmented-generation, wiki/concepts/mcp-model-context-protocol]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/rag/03-rag-integration-guide.md|Original file]]

## สรุป

คู่มืออธิบายว่า Deploy RAG Agent แล้วได้อะไรมา และต่อกับอะไรได้บ้าง — ครอบคลุม API Endpoints, MCP (Model Context Protocol) สำหรับเชื่อมกับ Claude Code/Cursor/IDE, Open Source Code Tools, และ Use Cases ที่หลากหลาย

## ประเด็นสำคัญ

### สิ่งที่ได้หลัง Deploy

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  1. API Endpoint (URL)                                           │
│     https://agent.sellsuki.com/api/v1/                           │
│                                                                  │
│  2. API Key(s)                                                   │
│     sk-sellsuki-xxxxxxxxxxxx                                     │
│                                                                  │
│  3. Documentation (Swagger/OpenAPI)                              │
│     https://agent.sellsuki.com/docs                              │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### API Endpoints ที่ได้

- **POST /api/v1/chat**: ถาม-ตอบ
- **POST /api/v1/search**: ค้นหา documents อย่างเดียว (ไม่ต้องให้ LLM ตอบ)
- **POST /api/v1/ingest**: เพิ่มเอกสารใหม่
- **GET /api/v1/health**: Health Check

### สิ่งที่เอาไปทำต่อได้

URL + API Key ที่ได้ เอาไปใส่ใน:

#### Chat Interfaces
- LINE Bot webhook
- Slack Bot
- Web Chat widget
- Mobile App

#### Developer Tools
- **Claude Code** (ผ่าน MCP)
- **Cursor IDE** (ผ่าน MCP)
- VS Code extension
- Continue.dev (ผ่าน MCP)
- Terminal CLI tool

#### Automation & Workflow
- n8n / Make.com / Zapier
- GitHub Actions
- CI/CD pipelines
- Cron jobs

#### Applications
- Internal dashboard
- Auto-fill forms
- Report generation
- Email auto-reply
- Search engine

#### Other AI Tools
- LangChain agents (เป็น tool)
- AutoGen agents
- Custom GPTs (OpenAI)
- Gemini Extensions

### ต่อกับ Claude Code / Cursor / IDE — ผ่าน MCP

MCP (Model Context Protocol) คือโปรโตคอลที่ Anthropic สร้างขึ้น — ทำให้ AI tools เข้าถึง data sources ภายนอกได้

#### MCP Server สำหรับ RAG Agent

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent
import httpx

server = Server("sellsuki-knowledge")

@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="search_company_knowledge",
            description="ค้นหาข้อมูลบริษัท Sellsuki เช่น นโยบาย, SOP, คู่มือ",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "คำถามหรือสิ่งที่ต้องการค้นหา"},
                    "category": {"type": "string", "enum": ["hr", "product", "policy", "general"]}
                },
                "required": ["query"]
            }
        ),
        Tool(
            name="ask_company_agent",
            description="ถามคำถามเกี่ยวกับบริษัท Sellsuki แล้วได้คำตอบพร้อมแหล่งอ้างอิง",
            inputSchema={
                "type": "object",
                "properties": {
                    "question": {"type": "string", "description": "คำถามที่ต้องการถาม"}
                },
                "required": ["question"]
            }
        )
    ]
```

#### Config Claude Code

```jsonc
// ~/.claude/claude_desktop_config.json
{
  "mcpServers": {
    "sellsuki-knowledge": {
      "command": "python",
      "args": ["/path/to/mcp_server.py"],
      "env": {
        "AGENT_API_URL": "https://agent.sellsuki.com/api/v1",
        "API_KEY": "sk-sellsuki-xxx"
      }
    }
  }
}
```

#### ใช้งานใน Claude Code
```
> นโยบายลาพักร้อนของ Sellsuki เป็นยังไง?
# Claude Code จะเรียก MCP tool "ask_company_agent"
# → ได้คำตอบจาก RAG Agent พร้อมอ้างอิง

> เขียน function สำหรับ calculate leave ตามนโยบาย Sellsuki
# Claude Code จะค้นหานโยบายลาจาก RAG → เขียนโค้ดให้ถูกต้อง
```

### ใช้ได้มากกว่าแค่แชท — 14 Use Cases

```
┌──────────────────────────────────────────────────────────────────┐
│                    รูปแบบการใช้งาน RAG Agent                      │
├──────────────────────────────────────────────────────────────────┤
│  1. 💬 Chat / Q&A           — ถาม-ตอบ (ที่รู้จักกัน)             │
│  2. 🔍 Semantic Search      — ค้นหาเอกสารอัจฉริยะ               │
│  3. 📝 Auto-fill / Form     — กรอกฟอร์มอัตโนมัติ                │
│  4. 📊 Report Generation    — สร้างรายงานจากข้อมูล               │
│  5. 📧 Email Drafting       — ร่างอีเมลจากข้อมูลบริษัท           │
│  6. ✅ Auto-approval        — ตรวจสอบตามนโยบายอัตโนมัติ          │
│  7. 🔔 Proactive Alert      — แจ้งเตือนอัจฉริยะ                  │
│  8. 📋 Summarization        — สรุปเอกสารยาวๆ                    │
│  9. 🔄 Data Enrichment      — เติมข้อมูลให้สมบูรณ์                │
│ 10. 🧪 Validation          — ตรวจสอบความถูกต้อง                 │
│ 11. 📚 Training Assistant   — ช่วยสอนพนักงานใหม่                │
│ 12. 🤖 Workflow Automation  — อัตโนมัติ workflow ทั้งหมด          │
│ 13. 💻 Code Generation     — เขียนโค้ดตาม business rules        │
│ 14. 🔌 API / Webhook       — ให้ระบบอื่นเรียกใช้                 │
└──────────────────────────────────────────────────────────────────┘
```

### ตัวอย่าง Use Cases

#### Semantic Search (ค้นหาอัจฉริยะ)
```python
@app.post("/api/v1/search")
async def semantic_search(request: SearchRequest):
    """ค้นหาเอกสารโดยไม่ต้องผ่าน LLM — เร็วกว่า ถูกกว่า"""
    results = search_similar(
        query=request.query,
        top_k=request.top_k or 10,
        category=request.category,
    )
    return {"results": results, "total": len(results)}
```

#### Auto-fill / Form
```python
# ส่ง form data → RAG หาข้อมูล → เติมให้อัตโนมัติ
@app.post("/api/v1/auto-fill")
async def auto_fill_form(request: AutoFillRequest):
    # เช่น กรอกใบลา → หานโยบายการลา → เติมข้อมูลให้อัตโนมัติ
```

#### Report Generation
```python
# สรุปข้อมูลจากหลาย source → สร้าง report
@app.post("/api/v1/report")
async def generate_report(request: ReportRequest):
    # เช่น "สรุปนโยบายการลาทั้งหมด" → ค้นหา → สรุปเป็นตาราง
```

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG — Retrieval-Augmented Generation]]
- [[wiki/concepts/mcp-model-context-protocol|MCP — Model Context Protocol]]
- [[wiki/concepts/api-design|API Design]]
