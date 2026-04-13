# RAG Agent — Deploy แล้วได้อะไร, ต่อกับอะไรได้บ้าง, ใช้ได้มากกว่าแชท

## สารบัญ

1. [Deploy แล้วได้อะไรมา](#1-deploy-แล้วได้อะไรมา)
2. [ต่อกับ Claude Code / Cursor / IDE ได้มั้ย](#2-ต่อกับ-claude-code--cursor--ide)
3. [ต่อกับ Open Source Code Tools](#3-ต่อกับ-open-source-code-tools)
4. [ใช้ได้มากกว่าแค่แชท — ทุกรูปแบบที่เป็นไปได้](#4-ใช้ได้มากกว่าแค่แชท)
5. [MCP (Model Context Protocol) — ทำให้ทุก AI Tool เข้าถึง RAG ได้](#5-mcp-model-context-protocol)
6. [API Endpoint ที่ควรสร้าง](#6-api-endpoints-ทั้งหมดที่ควรมี)
7. [ตัวอย่าง Use Cases จริง](#7-ตัวอย่าง-use-cases-จริง)
8. [Architecture แบบเต็ม](#8-architecture-แบบเต็ม)

---

## 1. Deploy แล้วได้อะไรมา

### สิ่งที่ได้หลัง Deploy

```
เมื่อ deploy RAG Agent เสร็จ คุณจะได้:

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

แค่มี 3 สิ่งนี้ ก็ต่อเข้ากับอะไรก็ได้ที่ส่ง HTTP request ได้
```

### API Endpoints ที่ได้

```bash
# =======================================
# ทดสอบเรียกได้เลยจาก Terminal
# =======================================

# 1. Chat (ถาม-ตอบ)
curl -X POST https://agent.sellsuki.com/api/v1/chat \
  -H "Content-Type: application/json" \
  -H "X-API-Key: sk-sellsuki-xxx" \
  -d '{
    "message": "ลาพักร้อนกี่วัน?",
    "session_id": "user_123"
  }'

# Response:
{
  "answer": "พนักงานประจำมีสิทธิ์ลาพักร้อนปีละ 10 วัน...",
  "sources": [
    {"title": "นโยบายการลา", "source": "hr_policy.pdf", "similarity": 0.92}
  ],
  "session_id": "user_123"
}

# 2. Search (ค้นหา documents อย่างเดียว ไม่ต้องให้ LLM ตอบ)
curl -X POST https://agent.sellsuki.com/api/v1/search \
  -H "X-API-Key: sk-sellsuki-xxx" \
  -d '{"query": "สวัสดิการ", "top_k": 5}'

# 3. Ingest (เพิ่มเอกสารใหม่)
curl -X POST https://agent.sellsuki.com/api/v1/ingest \
  -H "X-API-Key: sk-sellsuki-xxx" \
  -F "file=@new_policy.pdf" \
  -F "category=hr"

# 4. Health Check
curl https://agent.sellsuki.com/api/v1/health
```

### สิ่งที่เอาไปทำต่อได้

```
URL + API Key ที่ได้ เอาไปใส่ใน:

├── Chat Interfaces
│   ├── LINE Bot webhook
│   ├── Slack Bot
│   ├── Web Chat widget
│   └── Mobile App
│
├── Developer Tools
│   ├── Claude Code (ผ่าน MCP)
│   ├── Cursor IDE (ผ่าน MCP)
│   ├── VS Code extension
│   ├── Continue.dev (ผ่าน MCP)
│   └── Terminal CLI tool
│
├── Automation & Workflow
│   ├── n8n / Make.com / Zapier
│   ├── GitHub Actions
│   ├── CI/CD pipelines
│   └── Cron jobs
│
├── Applications
│   ├── Internal dashboard
│   ├── Auto-fill forms
│   ├── Report generation
│   ├── Email auto-reply
│   └── Search engine
│
├── Other AI Tools
│   ├── LangChain agents (เป็น tool)
│   ├── AutoGen agents
│   ├── Custom GPTs (OpenAI)
│   └── Gemini Extensions
│
└── Any HTTP client
    ├── Python (requests/httpx)
    ├── JavaScript (fetch/axios)
    ├── Go, Rust, Java, etc.
    └── Postman / Insomnia
```

---

## 2. ต่อกับ Claude Code / Cursor / IDE

### คำตอบสั้นๆ: ได้! ผ่าน MCP (Model Context Protocol)

```
MCP คือโปรโตคอลที่ Anthropic สร้างขึ้น
ทำให้ AI tools (Claude Code, Cursor, etc.) เข้าถึง data sources ภายนอกได้

RAG Agent ของคุณ → สร้างเป็น MCP Server
Claude Code / Cursor  → เป็น MCP Client อยู่แล้ว

ผลลัพธ์: พิมพ์ใน Claude Code ว่า
  "นโยบายลาพักร้อนของบริษัทเป็นยังไง?"
  Claude Code จะเรียก RAG Agent ของคุณ แล้วตอบได้เลย
```

### 2.1 สร้าง MCP Server สำหรับ RAG Agent

```python
# pip install mcp httpx

# mcp_server.py
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent
import httpx
import json

AGENT_API_URL = "https://agent.sellsuki.com/api/v1"
API_KEY = "sk-sellsuki-xxx"

# === สร้าง MCP Server ===
server = Server("sellsuki-knowledge")

# === Tool 1: ค้นหาข้อมูลบริษัท ===
@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="search_company_knowledge",
            description="""ค้นหาข้อมูลบริษัท Sellsuki เช่น:
            - นโยบายบริษัท (ลา, สวัสดิการ, กฎระเบียบ)
            - ข้อมูลผลิตภัณฑ์ Sellsuki
            - SOP / ขั้นตอนการทำงาน
            - ข้อมูล HR
            - คู่มือต่างๆ""",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "คำถามหรือสิ่งที่ต้องการค้นหา"
                    },
                    "category": {
                        "type": "string",
                        "description": "หมวดหมู่ (hr, product, policy, general)",
                        "enum": ["hr", "product", "policy", "general", ""],
                        "default": ""
                    }
                },
                "required": ["query"]
            }
        ),
        Tool(
            name="ask_company_agent",
            description="""ถามคำถามเกี่ยวกับบริษัท Sellsuki แล้วได้คำตอบ
            พร้อมแหล่งอ้างอิง ใช้เมื่อต้องการคำตอบสรุป ไม่ใช่แค่ค้นหา""",
            inputSchema={
                "type": "object",
                "properties": {
                    "question": {
                        "type": "string",
                        "description": "คำถามที่ต้องการถาม"
                    }
                },
                "required": ["question"]
            }
        ),
        Tool(
            name="list_company_documents",
            description="ดูรายการเอกสารทั้งหมดในระบบ knowledge base",
            inputSchema={
                "type": "object",
                "properties": {
                    "category": {
                        "type": "string",
                        "description": "filter ตามหมวดหมู่",
                        "default": ""
                    }
                }
            }
        ),
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    headers = {
        "Content-Type": "application/json",
        "X-API-Key": API_KEY,
    }
    
    async with httpx.AsyncClient(timeout=30.0) as client:
        
        if name == "search_company_knowledge":
            response = await client.post(
                f"{AGENT_API_URL}/search",
                headers=headers,
                json={
                    "query": arguments["query"],
                    "category": arguments.get("category", ""),
                    "top_k": 5,
                }
            )
            data = response.json()
            results = data.get("results", [])
            
            text = f"พบ {len(results)} ผลลัพธ์:\n\n"
            for i, r in enumerate(results, 1):
                text += f"[{i}] {r['title']} (similarity: {r['similarity']})\n"
                text += f"    {r['content'][:300]}...\n\n"
            
            return [TextContent(type="text", text=text)]
        
        elif name == "ask_company_agent":
            response = await client.post(
                f"{AGENT_API_URL}/chat",
                headers=headers,
                json={
                    "message": arguments["question"],
                    "session_id": "mcp_session",
                }
            )
            data = response.json()
            
            text = f"{data['answer']}\n\n"
            if data.get("sources"):
                text += "📎 แหล่งอ้างอิง:\n"
                for s in data["sources"]:
                    text += f"  - {s.get('title', 'N/A')} ({s.get('source', '')})\n"
            
            return [TextContent(type="text", text=text)]
        
        elif name == "list_company_documents":
            response = await client.get(
                f"{AGENT_API_URL}/documents",
                headers=headers,
                params={"category": arguments.get("category", "")},
            )
            data = response.json()
            
            text = "เอกสารในระบบ:\n\n"
            for doc in data.get("documents", []):
                text += f"- [{doc['category']}] {doc['title']} ({doc['source']})\n"
            
            return [TextContent(type="text", text=text)]

# === Run ===
async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(read_stream, write_stream)

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

### 2.2 ต่อกับ Claude Code

```jsonc
// ~/.claude/claude_desktop_config.json (Claude Desktop)
// หรือ claude code settings

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

```bash
# ใน Claude Code พิมพ์:
> นโยบายลาพักร้อนของ Sellsuki เป็นยังไง?

# Claude Code จะ:
# 1. เรียก MCP tool "ask_company_agent"
# 2. ได้คำตอบจาก RAG Agent
# 3. แสดงผลพร้อมอ้างอิง

# หรือใช้ขณะเขียนโค้ด:
> เขียน function สำหรับ calculate leave ตามนโยบาย Sellsuki
# Claude Code จะค้นหานโยบายลาจาก RAG → เขียนโค้ดให้ถูกต้องตามนโยบาย
```

### 2.3 ต่อกับ Cursor IDE

```jsonc
// Cursor Settings → MCP Servers
// .cursor/mcp.json (ในโปรเจค)

{
  "mcpServers": {
    "sellsuki-knowledge": {
      "command": "python",
      "args": ["./mcp_server.py"],
      "env": {
        "AGENT_API_URL": "https://agent.sellsuki.com/api/v1",
        "API_KEY": "sk-sellsuki-xxx"
      }
    }
  }
}
```

```
ใน Cursor Chat พิมพ์:
> @sellsuki-knowledge ขั้นตอนการสร้าง order ใน Sellsuki มียังไงบ้าง?
  ช่วยเขียน API endpoint สำหรับสร้าง order ตาม flow นี้

Cursor จะ:
1. ค้นหาข้อมูลจาก RAG (SOP การสร้าง order)
2. ใช้ข้อมูลนั้นเขียนโค้ดให้
3. โค้ดจะถูกต้องตาม business logic จริง
```

### 2.4 ต่อกับ VS Code (Continue.dev)

```jsonc
// .continue/config.json

{
  "experimental": {
    "mcpServers": [
      {
        "name": "sellsuki-knowledge",
        "command": "python",
        "args": ["./mcp_server.py"]
      }
    ]
  }
}

// หรือต่อตรงผ่าน context provider:
{
  "contextProviders": [
    {
      "name": "http",
      "params": {
        "url": "https://agent.sellsuki.com/api/v1/search",
        "title": "Sellsuki Knowledge",
        "description": "ค้นหาข้อมูล Sellsuki",
        "headers": {
          "X-API-Key": "sk-sellsuki-xxx"
        }
      }
    }
  ]
}
```

---

## 3. ต่อกับ Open Source Code Tools

### 3.1 Aider (AI Pair Programming)

```bash
# Aider สามารถอ่าน context จากไฟล์ได้
# สร้าง script ที่ดึงข้อมูลจาก RAG แล้วใส่ใน context file

# fetch_context.py
import httpx, sys

query = sys.argv[1]
response = httpx.post(
    "https://agent.sellsuki.com/api/v1/search",
    headers={"X-API-Key": "sk-sellsuki-xxx"},
    json={"query": query, "top_k": 5}
)

results = response.json()["results"]
with open(".aider.context.md", "w") as f:
    f.write("# Company Knowledge Context\n\n")
    for r in results:
        f.write(f"## {r['title']}\n{r['content']}\n\n")

# ใช้งาน:
python fetch_context.py "order creation flow"
aider --read .aider.context.md app/orders/api.py
```

### 3.2 Open Interpreter

```python
# Open Interpreter สามารถเรียก API ได้โดยตรง

# ใน Open Interpreter:
> ค้นหาข้อมูลจาก Sellsuki knowledge base เรื่องนโยบายลา
  แล้วสร้าง Python script สำหรับคำนวณวันลาคงเหลือ

# Open Interpreter จะ:
# 1. เขียนโค้ดเรียก RAG API
# 2. อ่านผลลัพธ์
# 3. เขียน script ตามข้อมูลที่ได้
```

### 3.3 GitHub Copilot (ผ่าน custom instructions)

```markdown
<!-- .github/copilot-instructions.md -->

## Sellsuki Business Rules

เมื่อเขียนโค้ดที่เกี่ยวกับ Sellsuki ให้อ้างอิงจาก:
- API docs: https://agent.sellsuki.com/docs
- ใช้ Sellsuki Knowledge API สำหรับ business rules
- Endpoint: POST /api/v1/search

<!-- Copilot จะใช้เป็น context เวลา suggest โค้ด -->
```

---

## 4. ใช้ได้มากกว่าแค่แชท

### คำตอบ: ไม่ใช่แค่แชท! RAG Agent ใช้ได้หลายรูปแบบมาก

```
┌──────────────────────────────────────────────────────────────────┐
│                    รูปแบบการใช้งาน RAG Agent                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 💬 Chat / Q&A           — ถาม-ตอบ (ที่รู้จักกัน)             │
│  2. 🔍 Semantic Search      — ค้นหาเอกสารอัจฉริยะ               │
│  3. 📝 Auto-fill / Form     — กรอกฟอร์มอัตโนมัติ                │
│  4. 📊 Report Generation    — สร้างรายงานจากข้อมูล               │
│  5. 📧 Email Drafting       — ร่างอีเมลจากข้อมูลบริษัท           │
│  6. ✅ Auto-approval        — ตรวจสอบตามนโยบายอัตโนมัติ          │
│  7. 🔔 Proactive Alert      — แจ้งเตือนอัจฉริยะ                  │
│  8. 📋 Summarization        — สรุปเอกสารยาวๆ                    │
│  9. 🔄 Data Enrichment      — เติมข้อมูลให้สมบูรณ์                │
│  10. 🧪 Validation          — ตรวจสอบความถูกต้อง                 │
│  11. 📚 Training Assistant   — ช่วยสอนพนักงานใหม่                │
│  12. 🤖 Workflow Automation  — อัตโนมัติ workflow ทั้งหมด          │
│  13. 💻 Code Generation     — เขียนโค้ดตาม business rules        │
│  14. 🔌 API / Webhook       — ให้ระบบอื่นเรียกใช้                 │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 4.1 Semantic Search (ค้นหาอัจฉริยะ)

```
ต่างจากแชทยังไง?
- แชท: ถาม → ได้คำตอบ (LLM สรุปให้)
- Search: ถาม → ได้เอกสารที่เกี่ยวข้อง (ไม่ต้องผ่าน LLM)
           เร็วกว่า ถูกกว่า เหมาะกับ search bar
```

```python
# === Backend API ===
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

```html
<!-- ฝัง Search Bar ในเว็บ Intranet -->
<div id="sellsuki-search">
  <input 
    type="text" 
    placeholder="ค้นหาข้อมูลบริษัท..."
    onkeyup="handleSearch(this.value)"
  />
  <div id="results"></div>
</div>

<script>
let debounceTimer;
async function handleSearch(query) {
  clearTimeout(debounceTimer);
  debounceTimer = setTimeout(async () => {
    if (query.length < 2) return;
    
    const res = await fetch('https://agent.sellsuki.com/api/v1/search', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-API-Key': 'sk-sellsuki-xxx'
      },
      body: JSON.stringify({ query, top_k: 5 })
    });
    const data = await res.json();
    
    document.getElementById('results').innerHTML = data.results
      .map(r => `
        <div class="result-item">
          <h3>${r.title}</h3>
          <p>${r.content.substring(0, 200)}...</p>
          <span class="badge">${r.category}</span>
          <span class="score">${(r.similarity * 100).toFixed(0)}% match</span>
        </div>
      `).join('');
  }, 300); // debounce 300ms
}
</script>
```

### 4.2 Auto-fill Form (กรอกฟอร์มอัตโนมัติ)

```
ตัวอย่าง: พนักงานกรอกใบลา
→ RAG ค้นหานโยบายลา
→ auto-fill จำนวนวันลาคงเหลือ, ชื่อหัวหน้า, ข้อจำกัด
```

```python
# === API Endpoint ===
@app.post("/api/v1/autofill")
async def autofill(request: AutofillRequest):
    """กรอกข้อมูลอัตโนมัติจาก knowledge base"""
    
    # ค้นหาข้อมูลที่เกี่ยวข้อง
    context = search_similar(request.context_query, top_k=3)
    context_text = "\n".join([r["content"] for r in context])
    
    # ใช้ LLM extract ข้อมูลที่ต้องการ
    response = openai_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": """Extract structured data from context.
             Return ONLY valid JSON matching the requested fields."""},
            {"role": "user", "content": f"""
Context: {context_text}

Extract these fields: {json.dumps(request.fields)}

Return JSON only."""}
        ],
        response_format={"type": "json_object"},
    )
    
    return json.loads(response.choices[0].message.content)
```

```javascript
// Frontend — ใบลาที่ auto-fill
async function autoFillLeaveForm() {
  const res = await fetch('/api/v1/autofill', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'X-API-Key': 'sk-xxx' },
    body: JSON.stringify({
      context_query: "นโยบายลาพักร้อน สิทธิ์การลา",
      fields: {
        max_vacation_days: "จำนวนวันลาพักร้อนสูงสุดต่อปี",
        advance_notice_days: "ต้องแจ้งล่วงหน้ากี่วัน",
        approval_required_from: "ต้องขออนุมัติจากใคร",
        sick_leave_certificate: "ลาป่วยกี่วันต้องมีใบรับรองแพทย์",
      }
    })
  });
  
  const data = await res.json();
  // Response:
  // {
  //   "max_vacation_days": 10,
  //   "advance_notice_days": 3,
  //   "approval_required_from": "หัวหน้างานโดยตรง",
  //   "sick_leave_certificate": 3
  // }
  
  // Auto-fill ลงฟอร์ม
  document.getElementById('max_days').value = data.max_vacation_days;
  document.getElementById('notice').value = data.advance_notice_days;
}
```

### 4.3 Report Generation (สร้างรายงาน)

```python
# === API Endpoint ===
@app.post("/api/v1/generate-report")
async def generate_report(request: ReportRequest):
    """สร้างรายงานจากข้อมูลใน knowledge base"""
    
    # ค้นหาข้อมูลทั้งหมดที่เกี่ยวข้อง
    results = search_similar(request.topic, top_k=15)
    context = "\n\n".join([
        f"[{r['title']}]\n{r['content']}" for r in results
    ])
    
    # สร้างรายงาน
    response = openai_client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": f"""สร้างรายงานในรูปแบบ {request.format}
             ใช้ข้อมูลจาก context เท่านั้น ห้ามเดา
             ภาษา: {request.language}"""},
            {"role": "user", "content": f"""
หัวข้อ: {request.topic}
รูปแบบ: {request.format}
ความยาว: {request.length}

ข้อมูลอ้างอิง:
{context}"""}
        ]
    )
    
    report = response.choices[0].message.content
    
    return {
        "report": report,
        "format": request.format,
        "sources_used": len(results),
    }
```

```bash
# ตัวอย่างเรียกใช้:
curl -X POST https://agent.sellsuki.com/api/v1/generate-report \
  -H "X-API-Key: sk-xxx" \
  -d '{
    "topic": "สรุปสวัสดิการพนักงาน Sellsuki ทั้งหมด",
    "format": "markdown",
    "length": "detailed",
    "language": "th"
  }'

# ได้รายงาน Markdown กลับมาเลย พร้อมนำไปใส่ใน Notion, Confluence, etc.
```

### 4.4 Auto-approval / Policy Check (ตรวจสอบนโยบาย)

```python
# === ตรวจสอบว่าคำขอสอดคล้องกับนโยบายหรือไม่ ===

@app.post("/api/v1/policy-check")
async def policy_check(request: PolicyCheckRequest):
    """ตรวจสอบคำขอกับนโยบายบริษัท"""
    
    # ค้นหานโยบายที่เกี่ยวข้อง
    policies = search_similar(
        f"นโยบาย กฎระเบียบ {request.request_type}",
        top_k=5,
        category="policy"
    )
    policy_text = "\n".join([r["content"] for r in policies])
    
    response = openai_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": """ตรวจสอบว่าคำขอสอดคล้องกับนโยบายหรือไม่
            Return JSON:
            {
              "approved": true/false,
              "reason": "เหตุผล",
              "policy_reference": "อ้างอิงนโยบายข้อไหน",
              "conditions": ["เงื่อนไขที่ต้องปฏิบัติ"]
            }"""},
            {"role": "user", "content": f"""
นโยบายบริษัท:
{policy_text}

คำขอ:
ประเภท: {request.request_type}
รายละเอียด: {request.details}
ผู้ขอ: {request.requester}"""}
        ],
        response_format={"type": "json_object"},
    )
    
    return json.loads(response.choices[0].message.content)
```

```javascript
// ตัวอย่าง: ระบบลาอัตโนมัติ
const checkResult = await fetch('/api/v1/policy-check', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', 'X-API-Key': 'sk-xxx' },
  body: JSON.stringify({
    request_type: "ลาพักร้อน",
    details: "ขอลา 5 วัน วันที่ 1-5 มกราคม แจ้งล่วงหน้า 1 วัน",
    requester: "สมชาย"
  })
});

// Response:
// {
//   "approved": false,
//   "reason": "แจ้งล่วงหน้าไม่ครบ ลาเกิน 3 วันต้องแจ้งล่วงหน้า 1 สัปดาห์",
//   "policy_reference": "นโยบายการลา ข้อ 2.1",
//   "conditions": ["ต้องแจ้งล่วงหน้าอย่างน้อย 7 วันทำการ"]
// }
```

### 4.5 Email Drafting (ร่างอีเมล)

```python
@app.post("/api/v1/draft-email")
async def draft_email(request: EmailDraftRequest):
    """ร่างอีเมลโดยใช้ข้อมูลจาก knowledge base"""
    
    context = search_similar(request.topic, top_k=5)
    context_text = "\n".join([r["content"] for r in context])
    
    response = openai_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": """ร่างอีเมลภาษาไทย สุภาพ เป็นทางการ
            ใช้ข้อมูลจาก context ให้ถูกต้อง"""},
            {"role": "user", "content": f"""
ร่างอีเมลเรื่อง: {request.topic}
ผู้รับ: {request.recipient}
โทน: {request.tone}

ข้อมูลอ้างอิง:
{context_text}"""}
        ]
    )
    
    return {
        "subject": f"เรื่อง: {request.topic}",
        "body": response.choices[0].message.content,
    }
```

### 4.6 Proactive Alert (แจ้งเตือนอัจฉริยะ)

```python
# === Cron Job ที่ตรวจสอบและแจ้งเตือนอัตโนมัติ ===

import schedule
import time

def check_policy_updates():
    """ตรวจสอบว่ามีนโยบายที่ใกล้หมดอายุหรือต้อง review"""
    
    results = search_similar("นโยบาย review date หมดอายุ", top_k=20)
    
    for doc in results:
        last_updated = doc.get("last_updated")
        if last_updated and is_older_than_months(last_updated, 12):
            send_slack_notification(
                channel="#hr-team",
                message=f"⚠️ เอกสาร '{doc['title']}' ไม่ได้อัปเดตมากกว่า 12 เดือน กรุณา review"
            )

def daily_knowledge_digest():
    """สรุปคำถามที่ถูกถามบ่อยประจำวัน"""
    
    # ดึง log คำถามวันนี้
    today_queries = get_today_queries()
    
    if not today_queries:
        return
    
    # ใช้ LLM สรุป
    summary = openai_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "สรุปคำถามที่ถูกถามบ่อย จัดกลุ่มตามหมวดหมู่"},
            {"role": "user", "content": f"คำถามวันนี้:\n" + "\n".join(today_queries)}
        ]
    )
    
    send_slack_notification(
        channel="#agent-analytics",
        message=f"📊 สรุปคำถามวันนี้:\n{summary.choices[0].message.content}"
    )

# รันทุกวัน
schedule.every().day.at("09:00").do(check_policy_updates)
schedule.every().day.at("18:00").do(daily_knowledge_digest)
```

### 4.7 Training Assistant (ช่วยสอนพนักงานใหม่)

```python
@app.post("/api/v1/onboarding")
async def onboarding_assistant(request: OnboardingRequest):
    """สร้าง onboarding checklist + quiz สำหรับพนักงานใหม่"""
    
    # ค้นหาข้อมูล onboarding ทั้งหมด
    context = search_similar(
        "onboarding พนักงานใหม่ ปฐมนิเทศ สิ่งที่ต้องรู้",
        top_k=15
    )
    context_text = "\n".join([r["content"] for r in context])
    
    if request.mode == "checklist":
        prompt = f"""สร้าง onboarding checklist สำหรับพนักงานใหม่
        ตำแหน่ง: {request.position}
        แผนก: {request.department}
        
        ข้อมูล:
        {context_text}
        
        Return JSON: {{"checklist": [{{"task": "...", "deadline": "...", "category": "..."}}]}}"""
    
    elif request.mode == "quiz":
        prompt = f"""สร้างแบบทดสอบความรู้เกี่ยวกับบริษัท 10 ข้อ
        พร้อมเฉลย สำหรับพนักงานใหม่
        
        ข้อมูล:
        {context_text}
        
        Return JSON: {{"quiz": [{{"question": "...", "options": [...], "answer": 0}}]}}"""
    
    response = openai_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "สร้างเนื้อหา onboarding ภาษาไทย"},
            {"role": "user", "content": prompt}
        ],
        response_format={"type": "json_object"},
    )
    
    return json.loads(response.choices[0].message.content)
```

### 4.8 Workflow Automation (n8n / Make.com / Zapier)

```
ตัวอย่าง Workflows ที่ทำได้:

1. พนักงานส่งใบลาใน Google Form
   → n8n เรียก RAG API /policy-check
   → ผ่าน → ส่ง approval ไป Slack
   → ไม่ผ่าน → แจ้งเหตุผลกลับทาง email

2. มีเอกสารใหม่ใน Google Drive
   → n8n detect ไฟล์ใหม่
   → เรียก RAG API /ingest
   → แจ้ง Slack ว่า "เพิ่มเอกสารใหม่แล้ว"

3. ลูกค้าส่ง email ถาม
   → n8n อ่าน email
   → เรียก RAG API /chat
   → ร่าง draft reply
   → ส่งให้ staff review ก่อน send

4. ทุกวันจันทร์เช้า
   → n8n เรียก /generate-report
   → สรุป FAQ ประจำสัปดาห์
   → ส่งไป Slack #weekly-digest
```

```javascript
// n8n HTTP Request Node Config:
{
  "method": "POST",
  "url": "https://agent.sellsuki.com/api/v1/chat",
  "headers": {
    "X-API-Key": "sk-sellsuki-xxx",
    "Content-Type": "application/json"
  },
  "body": {
    "message": "{{ $json.email_body }}",
    "session_id": "workflow_{{ $json.email_from }}"
  }
}
```

### 4.9 Code Generation (เขียนโค้ดตาม Business Rules)

```python
@app.post("/api/v1/generate-code")
async def generate_code(request: CodeGenRequest):
    """เขียนโค้ดโดยอ้างอิง business rules จาก knowledge base"""
    
    # ค้นหา business rules ที่เกี่ยวข้อง
    rules = search_similar(
        f"business rule logic {request.feature_description}",
        top_k=10
    )
    rules_text = "\n".join([r["content"] for r in rules])
    
    response = openai_client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": f"""คุณเป็น developer ที่เข้าใจ business rules ของ Sellsuki
            เขียนโค้ดภาษา {request.language} ตาม rules ที่ให้มา"""},
            {"role": "user", "content": f"""
Feature: {request.feature_description}
Language: {request.language}
Framework: {request.framework}

Business Rules:
{rules_text}

เขียนโค้ดที่ implement ตาม business rules เหล่านี้"""}
        ]
    )
    
    return {
        "code": response.choices[0].message.content,
        "rules_referenced": [r["title"] for r in rules],
    }
```

---

## 5. MCP (Model Context Protocol)

### MCP คืออะไร — อธิบายง่ายๆ

```
MCP = มาตรฐานที่ให้ AI tools ต่างๆ เข้าถึง data sources ภายนอกได้

เหมือน USB ที่เป็นมาตรฐาน:
- USB → เสียบอุปกรณ์อะไรก็ได้กับคอมพิวเตอร์ไหนก็ได้
- MCP → ต่อ data source อะไรก็ได้กับ AI tool ไหนก็ได้

สร้าง MCP Server 1 ตัว → ใช้ได้กับ:
  ✅ Claude Desktop / Claude Code
  ✅ Cursor IDE  
  ✅ VS Code (Continue.dev)
  ✅ Windsurf
  ✅ Cline
  ✅ Zed Editor
  ✅ AI tools อื่นๆ ที่ support MCP

ไม่ต้องเขียน integration แยกสำหรับแต่ละ tool!
```

### MCP Architecture

```
┌─────────────────────────────────────────────────┐
│              MCP Clients (AI Tools)              │
│                                                  │
│  Claude Code  │  Cursor  │  VS Code  │  Windsurf │
│       │            │          │            │      │
│       └────────────┼──────────┼────────────┘      │
│                    │   MCP Protocol               │
│                    ▼                              │
│            ┌──────────────┐                       │
│            │  MCP Server  │  ← คุณสร้างตัวนี้      │
│            │  (sellsuki)  │                       │
│            └──────┬───────┘                       │
│                   │                               │
│                   ▼                               │
│            ┌──────────────┐                       │
│            │  RAG Agent   │                       │
│            │  API         │                       │
│            └──────────────┘                       │
└─────────────────────────────────────────────────┘
```

### MCP Server แบบ HTTP (Remote)

```python
# mcp_server_http.py
# สำหรับ deploy เป็น remote server (ไม่ต้องติดตั้งในเครื่อง client)

# pip install mcp[http] httpx

from mcp.server import Server
from mcp.server.sse import SseServerTransport
from starlette.applications import Starlette
from starlette.routing import Route, Mount
from mcp.types import Tool, TextContent
import httpx
import uvicorn

AGENT_API_URL = "https://agent.sellsuki.com/api/v1"
API_KEY = "sk-sellsuki-xxx"

server = Server("sellsuki-knowledge")

@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="ask_sellsuki",
            description="ถามคำถามเกี่ยวกับบริษัท Sellsuki",
            inputSchema={
                "type": "object",
                "properties": {
                    "question": {"type": "string", "description": "คำถาม"}
                },
                "required": ["question"]
            }
        ),
        Tool(
            name="search_sellsuki_docs",
            description="ค้นหาเอกสาร Sellsuki",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {"type": "string"},
                    "category": {"type": "string", "default": ""}
                },
                "required": ["query"]
            }
        ),
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    headers = {"X-API-Key": API_KEY, "Content-Type": "application/json"}
    
    async with httpx.AsyncClient(timeout=30.0) as client:
        if name == "ask_sellsuki":
            resp = await client.post(
                f"{AGENT_API_URL}/chat",
                headers=headers,
                json={"message": arguments["question"], "session_id": "mcp"}
            )
            data = resp.json()
            return [TextContent(type="text", text=data["answer"])]
        
        elif name == "search_sellsuki_docs":
            resp = await client.post(
                f"{AGENT_API_URL}/search",
                headers=headers,
                json={"query": arguments["query"], "category": arguments.get("category", "")}
            )
            data = resp.json()
            text = "\n\n".join([
                f"**{r['title']}**\n{r['content'][:300]}" 
                for r in data.get("results", [])
            ])
            return [TextContent(type="text", text=text or "ไม่พบข้อมูล")]

# === HTTP SSE Transport ===
sse = SseServerTransport("/messages/")

async def handle_sse(request):
    async with sse.connect_sse(request.scope, request.receive, request._send) as streams:
        await server.run(streams[0], streams[1], server.create_initialization_options())

app = Starlette(
    routes=[
        Route("/sse", endpoint=handle_sse),
        Mount("/messages/", app=sse.handle_post_message),
    ]
)

# รัน: uvicorn mcp_server_http:app --host 0.0.0.0 --port 3001
# URL: https://mcp.sellsuki.com/sse
```

### Config สำหรับแต่ละ AI Tool

```jsonc
// === Claude Desktop ===
// ~/Library/Application Support/Claude/claude_desktop_config.json  (Mac)
// %APPDATA%\Claude\claude_desktop_config.json  (Windows)

{
  "mcpServers": {
    // Option A: Local (stdio)
    "sellsuki-local": {
      "command": "python",
      "args": ["/path/to/mcp_server.py"]
    },
    // Option B: Remote (SSE)
    "sellsuki-remote": {
      "url": "https://mcp.sellsuki.com/sse"
    }
  }
}


// === Cursor IDE ===
// .cursor/mcp.json (ในโปรเจค) หรือ global settings

{
  "mcpServers": {
    "sellsuki": {
      "url": "https://mcp.sellsuki.com/sse"
    }
  }
}


// === Claude Code (CLI) ===
// ~/.claude.json หรือ project .claude.json

{
  "mcpServers": {
    "sellsuki": {
      "command": "python",
      "args": ["./mcp_server.py"]
    }
  }
}
// หรือ:
// claude mcp add sellsuki -- python ./mcp_server.py


// === Continue.dev (VS Code) ===
// .continue/config.json

{
  "experimental": {
    "mcpServers": [
      {
        "name": "sellsuki",
        "command": "python",
        "args": ["./mcp_server.py"]
      }
    ]
  }
}


// === Windsurf ===
// ~/.windsurf/mcp.json

{
  "mcpServers": {
    "sellsuki": {
      "command": "python",
      "args": ["./mcp_server.py"]
    }
  }
}
```

---

## 6. API Endpoints ทั้งหมดที่ควรมี

```python
# === FastAPI — Complete API ===

from fastapi import FastAPI, UploadFile, File, Depends
from pydantic import BaseModel

app = FastAPI(title="Sellsuki RAG Agent API", version="1.0")

# ──────────────────────────────
# Core Endpoints
# ──────────────────────────────

# 1. Chat (ถาม-ตอบ ผ่าน LLM)
@app.post("/api/v1/chat")
async def chat(message: str, session_id: str = "default"):
    """ถาม-ตอบ — ค้นหา + LLM สรุปคำตอบ"""
    pass

# 2. Search (ค้นหา documents อย่างเดียว ไม่ผ่าน LLM)
@app.post("/api/v1/search")
async def search(query: str, top_k: int = 5, category: str = ""):
    """Semantic search — เร็ว ถูก ไม่ผ่าน LLM"""
    pass

# 3. Streaming Chat (ตอบทีละคำ real-time)
@app.post("/api/v1/chat/stream")
async def chat_stream(message: str, session_id: str = "default"):
    """Server-Sent Events — ตอบทีละ token เหมือน ChatGPT"""
    pass

# ──────────────────────────────
# Document Management
# ──────────────────────────────

# 4. Upload & Ingest Document
@app.post("/api/v1/documents/ingest")
async def ingest(file: UploadFile, category: str = "general"):
    """อัปโหลดเอกสารใหม่ → chunk → embed → store"""
    pass

# 5. List Documents
@app.get("/api/v1/documents")
async def list_documents(category: str = ""):
    """ดูรายการเอกสารทั้งหมด"""
    pass

# 6. Delete Document
@app.delete("/api/v1/documents/{doc_id}")
async def delete_document(doc_id: int):
    """ลบเอกสาร"""
    pass

# 7. Re-index (สร้าง embeddings ใหม่)
@app.post("/api/v1/documents/reindex")
async def reindex():
    """สร้าง embeddings ใหม่ทั้งหมด"""
    pass

# ──────────────────────────────
# Advanced Features
# ──────────────────────────────

# 8. Auto-fill
@app.post("/api/v1/autofill")
async def autofill(context_query: str, fields: dict):
    """ดึงข้อมูล structured จาก knowledge base"""
    pass

# 9. Report Generation
@app.post("/api/v1/generate-report")
async def generate_report(topic: str, format: str = "markdown"):
    """สร้างรายงาน"""
    pass

# 10. Policy Check
@app.post("/api/v1/policy-check")
async def policy_check(request_type: str, details: str):
    """ตรวจสอบคำขอกับนโยบาย"""
    pass

# 11. Email Draft
@app.post("/api/v1/draft-email")
async def draft_email(topic: str, recipient: str, tone: str = "formal"):
    """ร่างอีเมล"""
    pass

# 12. Summarize Document
@app.post("/api/v1/summarize")
async def summarize(doc_id: int = None, text: str = None):
    """สรุปเอกสาร"""
    pass

# ──────────────────────────────
# Analytics & Admin
# ──────────────────────────────

# 13. Query Logs
@app.get("/api/v1/analytics/queries")
async def get_query_logs(date_from: str, date_to: str):
    """ดู log คำถามทั้งหมด"""
    pass

# 14. Popular Questions
@app.get("/api/v1/analytics/popular")
async def popular_questions(limit: int = 10):
    """คำถามที่ถูกถามบ่อย"""
    pass

# 15. Unanswered Questions
@app.get("/api/v1/analytics/unanswered")
async def unanswered_questions():
    """คำถามที่ bot ตอบไม่ได้"""
    pass

# 16. Health
@app.get("/api/v1/health")
async def health():
    """ตรวจสอบสถานะระบบ"""
    pass
```

---

## 7. ตัวอย่าง Use Cases จริง

### Use Case 1: พนักงานใหม่ Onboarding

```
วันแรกของพนักงานใหม่:

สมชาย (พนักงานใหม่) เปิด LINE → พิมพ์ถาม Suki Bot:

สมชาย: "ผมเป็นพนักงานใหม่ ต้องทำอะไรบ้าง?"
Bot:    "ยินดีต้อนรับค่ะ! สิ่งที่ต้องทำในสัปดาห์แรก:
         1. ส่งเอกสาร HR (สำเนาบัตรประชาชน, สมุดบัญชี)
         2. รับ laptop จาก IT — ติดต่อคุณวิทย์ ชั้น 3
         3. เข้า orientation วันพุธ 10:00 ห้อง Meeting A
         4. ตั้งค่า email บริษัท (คู่มือ: link)
         📎 อ้างอิง: Onboarding Checklist 2024"

สมชาย: "wifi บริษัทรหัสอะไร?"
Bot:    "WiFi: Sellsuki-Office
         Password: xxxxxxxx
         📎 อ้างอิง: IT Setup Guide"

สมชาย: "ร้านอาหารแถวนี้แนะนำอะไรบ้าง?"
Bot:    "ขออภัยค่ะ ไม่มีข้อมูลเรื่องนี้ในระบบ
         แนะนำถามเพื่อนร่วมงาน หรือดู Google Maps ค่ะ 😊"
```

### Use Case 2: Developer ใช้ผ่าน Cursor IDE

```
Developer กำลังเขียนระบบ Order Management:

(ใน Cursor Chat)
Dev: @sellsuki-knowledge ขั้นตอนการสร้าง order ใน Sellsuki มียังไงบ้าง?
     ต้อง validate อะไรบ้าง?

Cursor: (เรียก MCP → RAG API)
    ตาม SOP ของ Sellsuki การสร้าง order ต้อง:
    1. ตรวจสอบ stock availability
    2. Validate shipping address
    3. คำนวณค่าส่ง (ตาม zone)
    4. ตรวจ credit limit ลูกค้า
    5. สร้าง order number (format: ORD-YYYYMMDD-XXXX)
    
    📎 อ้างอิง: SOP Order Management v2.3

Dev: เขียน validation function ตาม rules เหล่านี้ให้หน่อย

Cursor: (เขียนโค้ดให้ โดยอิง business rules จริง)
    ```python
    def validate_order(order: OrderCreate) -> ValidationResult:
        errors = []
        
        # Rule 1: Stock check
        for item in order.items:
            stock = get_stock(item.product_id)
            if stock < item.quantity:
                errors.append(f"สินค้า {item.product_id} คงเหลือ {stock}")
        
        # Rule 2: Shipping address
        if not is_valid_thai_address(order.shipping_address):
            errors.append("ที่อยู่จัดส่งไม่ถูกต้อง")
        
        # ... (ครบทุก rule)
    ```
```

### Use Case 3: HR ใช้ตรวจสอบนโยบาย

```
HR Manager ใช้ผ่าน Slack:

HR: @suki-bot พนักงานขอลาพักร้อน 15 วันติดต่อกัน ทำได้มั้ย?

Bot: ❌ ไม่สามารถอนุมัติได้ตามนโยบายปัจจุบัน
     
     เหตุผล:
     - นโยบายกำหนดลาพักร้อนต่อเนื่องได้ไม่เกิน 5 วันทำการ
     - หากต้องการลาเกิน 5 วัน ต้องได้รับอนุมัติจาก VP ขึ้นไป
     
     ทางเลือก:
     1. แบ่งลาเป็น 2 ช่วง (ห่างกันอย่างน้อย 2 สัปดาห์)
     2. ขอ special approval จาก VP
     
     📎 อ้างอิง: นโยบายการลา ข้อ 3.2.1
```

### Use Case 4: Automation — Weekly Digest

```
ทุกวันศุกร์ 17:00 ระบบจะ:

1. รวบรวมคำถามทั้งสัปดาห์
2. วิเคราะห์ patterns
3. ส่ง digest ไป Slack

#weekly-suki-digest:
━━━━━━━━━━━━━━━━━━━━━━━━━
📊 Suki Bot Weekly Digest
สัปดาห์ที่ 10/2026

คำถามทั้งหมด: 156 ครั้ง
ตอบได้: 142 (91%)
ตอบไม่ได้: 14 (9%)

🔥 หมวดยอดนิยม:
1. การลา (34 ครั้ง)
2. สวัสดิการ (28 ครั้ง)  
3. IT Support (22 ครั้ง)

⚠️ คำถามที่ตอบไม่ได้ (ควรเพิ่มข้อมูล):
- "ปฏิทินวันหยุดปี 2026" (5 ครั้ง)
- "ขั้นตอนเบิก laptop ใหม่" (4 ครั้ง)
- "นโยบาย remote work" (3 ครั้ง)
━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 8. Architecture แบบเต็ม

```
┌─────────────────────────────────────────────────────────────────────┐
│                        SELLSUKI RAG AGENT                          │
│                      Complete Architecture                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─── AI / Dev Tools ───┐  ┌─── Chat Channels ──┐  ┌── Web ──┐    │
│  │ Claude Code          │  │ LINE Bot           │  │ Web Chat│    │
│  │ Cursor IDE           │  │ Slack Bot          │  │ Intranet│    │
│  │ VS Code / Continue   │  │ Discord Bot        │  │ Search  │    │
│  │ Windsurf / Cline     │  │ Teams Bot          │  │ Dashboard│   │
│  └──────────┬───────────┘  └────────┬───────────┘  └────┬────┘    │
│             │ MCP                    │ Webhook            │ HTTP    │
│             │                        │                    │         │
│  ┌──────────▼────────────────────────▼────────────────────▼──────┐  │
│  │                                                               │  │
│  │                    API Gateway (FastAPI)                       │  │
│  │                                                               │  │
│  │  /chat  /search  /ingest  /autofill  /policy-check           │  │
│  │  /generate-report  /draft-email  /summarize  /stream          │  │
│  │                                                               │  │
│  └──────────┬─────────────────────────────────┬─────────────────┘  │
│             │                                  │                    │
│  ┌──────────▼──────────┐          ┌───────────▼───────────┐       │
│  │   RAG Engine        │          │   Document Pipeline   │       │
│  │                     │          │                       │       │
│  │ 1. Query → Embed    │          │ 1. Extract (PDF/Doc)  │       │
│  │ 2. Vector Search    │          │ 2. Clean              │       │
│  │ 3. Re-rank          │          │ 3. Chunk              │       │
│  │ 4. LLM Generate     │          │ 4. Embed              │       │
│  │                     │          │ 5. Store              │       │
│  └──────────┬──────────┘          └───────────┬───────────┘       │
│             │                                  │                    │
│  ┌──────────▼──────────────────────────────────▼─────────────────┐  │
│  │                                                               │  │
│  │              PostgreSQL + pgvector                             │  │
│  │              (Vector Database)                                 │  │
│  │                                                               │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌───────────────────┐  ┌──────────────┐  ┌────────────────────┐   │
│  │ LLM Provider      │  │ Embedding    │  │ Monitoring         │   │
│  │ OpenAI / Gemini / │  │ OpenAI /     │  │ Logs / Analytics / │   │
│  │ Claude API        │  │ Cohere       │  │ Alerts             │   │
│  └───────────────────┘  └──────────────┘  └────────────────────┘   │
│                                                                     │
│  ┌── Automation ──────────────────────────────────────────────┐     │
│  │ n8n / Make.com / Cron Jobs                                 │     │
│  │ → Weekly digest, Policy alerts, Auto-ingest new docs       │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### สรุป: ทำอะไรได้บ้างทั้งหมด

```
Deploy เสร็จ คุณจะได้ระบบที่:

✅ ถาม-ตอบผ่าน Chat (LINE, Slack, Web, etc.)
✅ ค้นหาเอกสารอัจฉริยะ (Semantic Search)
✅ ใช้ใน IDE ขณะเขียนโค้ด (Claude Code, Cursor)
✅ กรอกฟอร์มอัตโนมัติ (Auto-fill)
✅ สร้างรายงาน (Report Generation)
✅ ตรวจสอบนโยบาย (Policy Check)
✅ ร่างอีเมล (Email Drafting)
✅ แจ้งเตือนอัจฉริยะ (Proactive Alerts)
✅ ช่วย Onboarding พนักงานใหม่
✅ เขียนโค้ดตาม Business Rules
✅ Workflow Automation ทุกรูปแบบ
✅ API สำหรับระบบอื่นเรียกใช้
✅ Analytics & Insights

ทั้งหมดนี้ใช้ RAG Agent API ตัวเดียวกัน
แค่สร้าง endpoint ที่เหมาะกับแต่ละ use case
```
