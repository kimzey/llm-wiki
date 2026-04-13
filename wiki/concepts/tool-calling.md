---
title: "Tool Calling — กลไกให้ LLM เรียก Function ภายนอก"
type: concept
tags: [tool-calling, function-calling, agent, llm, openai, langchain]
sources: [wiki/sources/rag-vs-agent-explained, wiki/sources/step2-tools-agents, wiki/sources/langgraph-deep-dive, wiki/sources/multi-agent-deep-dive]
related: [wiki/concepts/ai-agent, wiki/concepts/agentic-rag, wiki/concepts/llm-large-language-model]
created: 2026-04-14
updated: 2026-04-14
---

## สรุปสั้น

Tool Calling (หรือ Function Calling) คือกลไกที่ทำให้ LLM สามารถ "เรียกใช้ function ภายนอก" ได้ — LLM ไม่ได้รันโค้ดเอง แต่ **output โครงสร้าง JSON** บอกว่าอยากเรียก function อะไรด้วย arguments อะไร แล้วระบบจะรันให้และส่งผลลัพธ์กลับ

## อธิบาย

### ขั้นตอน Tool Calling

```
1. Developer กำหนด Tools (functions + schema ให้ LLM รู้จัก)
         ↓
2. User ส่ง message มา
         ↓  
3. LLM ตัดสินใจ: "ต้องเรียก tool ไหนมั้ย?"
         ↓
4a. ถ้าต้องเรียก → LLM output JSON: {"tool": "search", "args": {"query": "..."}}
         ↓
5. Code เรียก function จริง (ค้น DB, call API ฯลฯ)
         ↓
6. ผลลัพธ์ส่งกลับให้ LLM
         ↓
7. LLM ตอบ user ด้วยข้อมูลจาก tool
         ↓
4b. ถ้าไม่ต้องเรียก → LLM ตอบตรงๆ เลย
```

### ทำไม Tool Calling สำคัญ

LLM มีข้อจำกัด:
- **ความรู้หมดอายุ** — training cutoff
- **ไม่รู้ข้อมูล real-time** — ราคาหุ้น, อากาศ
- **ทำ action ไม่ได้** — ส่ง email, อัปเดต DB

Tool Calling แก้ทั้งหมดนี้ โดยให้ LLM "ยื่นมือออกไป" เรียกข้อมูลหรือทำ action ผ่าน code จริง

## ประเด็นสำคัญ

### Tool Schema (OpenAI format)

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "search_documents",
            "description": "ค้นหาเอกสารใน knowledge base ตาม query",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "คำค้นหา"},
                    "category": {"type": "string", "enum": ["hr", "tech", "finance"]}
                },
                "required": ["query"]
            }
        }
    }
]
```

### LangChain @tool decorator

```python
from langchain_core.tools import tool

@tool
def search_kb(query: str, category: str = None) -> str:
    """ค้นหาข้อมูลใน Sellsuki Knowledge Base"""
    results = vector_db.similarity_search(query, filter={"category": category})
    return "\n".join([r.page_content for r in results])

# bind tool กับ LLM
llm_with_tools = llm.bind_tools([search_kb, get_employee_info])
```

### LangGraph ToolNode

```python
from langgraph.prebuilt import ToolNode, tools_condition

# สร้าง graph ที่รัน tools อัตโนมัติ
graph_builder.add_node("tools", ToolNode(tools=[search_kb]))
graph_builder.add_conditional_edges("llm", tools_condition)
# tools_condition = ถ้า LLM เรียก tool → ไป tools node
#                   ถ้าไม่ → END
```

### Parallel Tool Calls

LLM ทันสมัยสามารถเรียกหลาย tools พร้อมกัน:

```
User: "เปรียบเทียบราคาหุ้น AAPL กับ GOOG"
LLM → เรียก get_price("AAPL") และ get_price("GOOG") พร้อมกัน
     → รวมผลลัพธ์ → ตอบ
```

## ตัวอย่าง / กรณีศึกษา

**Arona Tool Call Agent:** ใช้ Vercel AI SDK กับ `tools` option — LLM เลือกเรียก `searchDocumentation(query)` เมื่อต้องการข้อมูลจาก Elysia docs, มี stop conditions ป้องกัน infinite loop (max 5 iterations)

**Sellsuki Agent:** Tools ที่เตรียมให้ agent มีเช่น `search_policy`, `get_employee_profile`, `create_leave_request` — Agent เลือก tool ตามคำถาม HR ที่รับมา

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/ai-agent|AI Agent]] — Tool Calling คือกลไกหลักที่ทำให้ Agent "ทำได้มากกว่าแค่ตอบ"
- [[wiki/concepts/agentic-rag|Agentic RAG]] — Agentic RAG ใช้ Tool Calling ให้ LLM เลือก search เอง แทนการ search ทุก request
- [[wiki/concepts/llm-large-language-model|LLM]] — LLM เป็นผู้ตัดสินใจว่าจะเรียก tool ไหน ด้วย parameters อะไร

## แหล่งที่มา

- [[wiki/sources/rag-vs-agent-explained|RAG vs Agent — มันคืออะไร ต่างกันยังไง]]
- [[wiki/sources/step2-tools-agents|Step 2: Tools & Agents]]
- [[wiki/sources/langgraph-deep-dive|LangGraph Deep Dive]]
- [[wiki/sources/multi-agent-deep-dive|Multi-Agent Systems Deep Dive]]
