---
title: "Agentic RAG — Agent ที่ใช้ RAG"
type: concept
tags: [rag, agent, agentic-rag, decision-making, llm]
sources: [wiki/sources/rag-complete-knowledge, wiki/sources/rag-vs-agent-explained, step3-rag-tutorial.md, step5-advanced-rag-tutorial.md, advanced-rag-deep-dive.md]
related: [wiki/concepts/rag-retrieval-augmented-generation, wiki/concepts/ai-agent]
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

Agentic RAG คือ RAG System ที่ LLM มี "agency" (ความสามารถตัดสินใจเอง) — ไม่ใช่ flow ตายตัว แต่ LLM เลือกเองว่าจะ search หรือไม่, search อะไร, และ search กี่รอบ

## อธิบาย

### คืออะไร

```
AI + Vector Database + fixed pipeline  = RAG
AI + Vector Database + ตัดสินใจเอง     = Agentic RAG

Agentic RAG = Agent ที่ใช้ RAG เป็น tool หลัก
```

### Spectrum

```
LLM เปล่า ──────▶ RAG ──────▶ Agentic RAG ──────▶ Full Agent

ไม่มี DB          มี DB             มี DB +            มี DB +
ไม่มี search      search ตายตัว     search ดีขึ้น      ตัดสินใจเอง      tools เยอะ
ตอบจากความรู้เดิม  flow ตายตัว       flow ตายตัว       เลือก tool        วางแผน
                                       loop ได้          multi-step
```

### จุดสำคัญของ Agentic RAG

#### 1. Decision Making
LLM ตัดสินใจเองว่า:
- ต้อง search หรือไม่?
- ถ้า search → search อะไร? (query selection)
- search กี่รอบ? (iteration)

#### 2. Tool Selection
```python
# LLM เลือก tool เอง
tools = [
    search_hr_tool,
    search_product_tool,
    search_general_tool,
]

# LLM คิด: "คำถามนี้เกี่ยวกับ HR" → เลือก search_hr_tool
```

#### 3. Loop
```python
for i in range(5):  # loop ได้สูงสุด 5 รอบ
    response = llm.chat(messages, tools=tools)

    if response.has_tool_calls:
        # execute tool → ดูผลลัพธ์ → loop กลับไป
    else:
        # LLM ตัดสินใจว่า "พอแล้ว ตอบได้"
        return response.content
```

## ประเด็นสำคัญ

### RAG vs Agentic RAG

#### RAG Flow (ไม่มี agency)
```
User: "ลาพักร้อนกี่วัน?"
│
▼ [Embed คำถาม] (ขั้นตอนที่ 1 ตายตัว)
▼ [Search vector DB → top 5 chunks] ← ALWAYS search
▼ [Stuff chunks เข้า prompt] ← ALWAYS stuff
▼ [LLM Generate answer] ← ALWAYS ส่งให้ LLM
│
▼
"ลาพักร้อนได้ 10 วัน/ปี"

ข้อดี:  เร็ว, คาดเดาได้, debug ง่าย
ข้อเสีย: ไม่ยืดหยุ่น, เสียเงิน search ทุก request
```

#### Agentic RAG Flow (มี agency)
```
User: "เปรียบเทียบนโยบายลาทุกประเภท"
│
▼ [LLM THINK] ← จุดตัดสินใจ #1
"คำถามนี้กว้าง ต้อง search หลายรอบ เริ่มจากลาพักร้อนก่อน"
│
▼ [search_hr("ลาพักร้อน")] → ได้ข้อมูล
▼ [LLM THINK] ← จุดตัดสินใจ #2
"ได้ลาพักร้อนแล้ว ยังไม่ครบ ต้องหาลาป่วย"
│
▼ [search_hr("ลาป่วย")] → ได้ข้อมูล
▼ [LLM THINK] ← จุดตัดสินใจ #3
"ยังขาดลากิจ ลาคลอด ลาบวช"
│
▼ [search_hr("ลากิจ ลาคลอด ลาบวช")] → ได้ข้อมูล
▼ [LLM THINK] ← จุดตัดสินใจ #4
"ข้อมูลครบแล้ว สรุปได้เลย"
│
▼ [LLM Generate] → ตารางเปรียบเทียบครบทุกประเภท ✅

มีจุดตัดสินใจ 4 จุด → นี่คือ "agency"
```

### ตัวอย่าง Comparison

#### คำถามง่าย
```
Agentic RAG: "สวัสดี"
→ LLM คิด: "คำถามนี้แค่ทักทาย ไม่ต้อง search"
→ ตอบเลย: "สวัสดีครับ! มีอะไรให้ช่วยมั้ยครับ?" ✅
→ ไม่ search เลย (ประหยัด!)
```

#### คำถามเฉพาะเจาะจง
```
Agentic RAG: "ลาพักร้อนกี่วัน?"
→ LLM คิด: "ต้อง search เรื่อง HR การลา"
→ tool_call: search_hr("ลาพักร้อน สิทธิ์การลา")
→ ได้ผลลัพธ์ → LLM: "ข้อมูลพอแล้ว"
→ ตอบ: "10 วัน/ปี ต้องแจ้ง 3 วัน" ✅
```

#### คำถามซับซ้อน
```
Agentic RAG: "เปรียบเทียบนโยบายลาทุกประเภท"
Iteration 1: LLM → search_hr("ลาพักร้อน")     → ได้ข้อมูล
Iteration 2: LLM → search_hr("ลาป่วย")         → ได้ข้อมูล
Iteration 3: LLM → search_hr("ลากิจ ลาคลอด")   → ได้ข้อมูล
Iteration 4: LLM → "ข้อมูลครบแล้ว" → ตอบเปรียบเทียบ ✅
→ search 3 รอบ เอง ตัดสินใจเอง!
```

## ตัวอย่าง / กรณีศึกษา

### Implementation

```python
def agentic_rag(question: str) -> str:
    messages = [
        {"role": "system", "content": "คุณเป็น Suki Bot มี tools ค้นหาข้อมูล"},
        {"role": "user", "content": question},
    ]

    for i in range(5):  # loop ได้สูงสุด 5 รอบ
        response = llm.chat(messages, tools=[
            search_hr_tool,
            search_product_tool,
            search_general_tool,
        ])

        # ────── จุดสำคัญ: LLM ตัดสินใจเอง ──────

        if response.has_tool_calls:
            # LLM เลือกว่าจะใช้ tool ไหน ด้วย query อะไร
            for tool_call in response.tool_calls:
                result = execute_tool(tool_call)
                messages.append(tool_result(result))
            # loop กลับไปให้ LLM ดูผลลัพธ์
        else:
            # LLM ตัดสินใจว่า "พอแล้ว ตอบได้"
            return response.content

    return "ขอโทษ หาข้อมูลไม่ได้"
```

### สำหรับ Sellsuki — ทำแค่ไหนดี

```
Phase 1: RAG (สัปดาห์ 1-4)
→ ตอบคำถามพื้นฐานได้

Phase 2: Agentic RAG (สัปดาห์ 5-6)
→ เพิ่ม agency:
  - ให้ LLM เลือกว่าจะ search หรือไม่
  - ให้เลือก category (HR, Product, IT)
  - ให้ loop ได้ถ้าข้อมูลไม่พอ

เหตุผล:
- ประหยัด cost (ไม่ search ทุก request)
- ตอบคำถามซับซ้อนได้
- ยังไม่ซับซ้อนเกินไป
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — Agentic RAG คือ RAG + agency
- [[wiki/concepts/ai-agent|AI Agent]] — Agentic RAG คือ Agent ที่ใช้ RAG เป็น tool หลัก
- [[wiki/concepts/tool-calling|Tool Calling]] — กลไกที่ Agentic RAG ใช้เลือกและเรียก tools

## แหล่งที่มา

- Complete RAG & Agent Knowledge Base
- RAG vs Agent — Explained

## จาก RAG Deep Dive (LangChain)

### RAG Agent Pattern ด้วย LangGraph

```python
# Pattern: StateGraph + domain-specific tools
@tool
def search_hr_docs(query: str) -> str:
    """ค้นหาข้อมูลจากเอกสาร HR เช่น นโยบายลา สวัสดิการ"""
    # filter ตาม department
    ...

@tool
def search_it_docs(query: str) -> str:
    """ค้นหาข้อมูลจากเอกสาร IT"""
    ...

@tool
def calculate(expression: str) -> str:
    """คำนวณตัวเลข"""
    ...

# StateGraph: agent → tools → agent (loop)
builder = StateGraph(RAGAgentState)
builder.add_conditional_edges("agent", tools_condition)
```

Agent เลือก tool ตามประเภทคำถาม (HR, IT, คำนวณ) โดยอัตโนมัติ และยังสามารถ combine หลาย tools เช่น ค้นหา HR ข้อมูลค่าอาหาร แล้ว calculate รวม

### ทำไม Tool-per-Domain ดีกว่า Single Search Tool

- Single retriever ค้นทั้งหมด → อาจได้ chunks ที่ไม่เกี่ยวข้อง
- แยก HR tool, IT tool → LLM เลือก scope ที่ถูกต้อง → precision สูงขึ้น
- ทำได้โดย metadata filter: `filter={"department": "HR"}`

- [[wiki/sources/rag/rag-deep-dive-langchain|RAG Deep Dive — LangChain]]
