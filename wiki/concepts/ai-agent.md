---
title: "AI Agent — ระบบ AI ที่ตัดสินใจเอง"
type: concept
tags: [agent, ai, llm, autonomous, decision-making]
sources: [wiki/sources/rag-complete-knowledge, wiki/sources/rag-vs-agent-explained, step2-tools-agents.md, step4-langgraph-tutorial.md, step6-multi-agent-tutorial.md, langgraph-deep-dive.md, multi-agent-deep-dive.md]
related: [wiki/concepts/llm-large-language-model, wiki/concepts/rag-retrieval-augmented-generation, wiki/concepts/agentic-rag]
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

AI Agent คือระบบ AI ที่สามารถตัดสินใจเองได้ — ไม่ใช่แค่ตอบคำถาม แต่สามารถเลือกว่าจะทำอะไร, เลือก tools, วางแผน และลงมือทำ (execute) ได้

## อธิบาย

### Agent คืออะไร

```
Agent = ระบบ AI ที่ตัดสินใจเอง เลือกว่าจะทำอะไร

RAG ≠ Agent
RAG = เทคนิค (technique) ในการดึงข้อมูลมาช่วย LLM ตอบ
Agent = ระบบ AI ที่ตัดสินใจเอง

เปรียบเทียบ:
  RAG   = สูตรอาหาร (ขั้นตอน 1-2-3 ตายตัว)
  Agent = พ่อครัว (ตัดสินใจเองว่าจะทำเมนูอะไร ใช้วัตถุดิบอะไร)

  สูตรอาหาร ≠ พ่อครัว
  แต่พ่อครัวอาจ "ใช้" สูตรอาหารเป็นส่วนหนึ่งของการทำงาน

เช่นเดียวกัน:
  RAG ≠ Agent
  แต่ Agent อาจ "ใช้" RAG เป็นเครื่องมือหนึ่ง
```

### ทำไมต้อง Agent

**ปัญหาของ RAG**:
- RAG มี flow ตายตัว: search → stuff → LLM → ตอบ
- ไม่มีการ "ตัดสินใจ" — ทำเหมือนกันทุกครั้ง
- เสียเงิน search ทุก request (แม้คำถามแค่ "สวัสดี")

**Agent แก้ปัญหา**:
- LLM เลือกว่าจะ search หรือไม่ (decision making)
- LLM เลือกว่า search อะไร (tool selection)
- LLM ตรวจผลลัพธ์ว่าพอมั้ย (self-evaluation)
- LLM ทำซ้ำได้ถ้าไม่พอ (loop)

### Agent Loop

```
1. LLM ตัดสินใจ: "ต้อง search หรือไม่?"
   ↓
2. Search → ได้ผลลัพธ์
   ↓
3. LLM ประเมิน: "พอมั้ย?"
   ↓
4a. ถ้าไม่พอ → กลับไป (2)
   ↓
4b. ถ้าพอแล้ว → ตอบ
```

### Agentic Patterns

#### ReAct (Reasoning + Acting)
```
คิด → ทำ → observe → คิดใหม่

1. Thought: คิดว่าจะทำอะไร
2. Action: ทำ (เรียก tool)
3. Observation: ดูผลลัพธ์
4. Repeat: จนกว่าจะเสร็จ
```

#### Plan-and-Execute
```
1. วางแผนก่อน (generate plan)
2. Execute ทีละขั้นตอน
3. ปรับเปลี่ยนแผนถ้าจำเป็น
```

#### Self-Reflection
```
LLM ตรวจสอบผลลัพธ์ของตัวเอง:
- ตอบตรงประเด็นมั้ย?
- มีข้อมูลพอมั้ย?
- ควรทำอะไรต่อ?
```

## ประเด็นสำคัญ

### RAG vs Agent vs Agentic RAG

| คุณสมบัติ | RAG | Agent | Agentic RAG |
|-----------|-----|-------|-------------|
| มี LLM | ✅ | ✅ | ✅ |
| มี Database | ✅ | ❌/✅ | ✅ |
| Search | ✅ ตายตัว | ❌/✅ เลือกเอง | ✅ เลือกเอง |
| ตัดสินใจ | ❌ | ✅ | ✅ |
| Loop | ❌ | ✅ | ✅ |
| หลาย Tools | ❌ (search 1) | ✅ (actions) | ✅ (search หลาย) |
| วางแผน | ❌ | ✅ | บางส่วน |
| ลงมือทำ | ❌ | ✅ | ❌ |

### Full Agent vs Agentic RAG

**Full Agent**:
```
Tools:
├── Knowledge tools (RAG)
│   ├── search_hr
│   └── search_product
├── Action tools
│   ├── send_email
│   ├── create_ticket
│   ├── book_meeting
│   └── calculate_leave
└── External tools
    ├── web_search
    └── calendar

ตัวอย่าง: "ช่วยลาพักร้อนให้ด้วย"
→ ค้นนโยบาย (RAG)
→ คำนวณวันลาเหลือ
→ สร้างใบลา
→ แจ้งหัวหน้า
```

**Agentic RAG**:
```
Tools:
├── search_hr
├── search_product
└── search_general

ตัวอย่าง: "เปรียบเทียบนโยบายลาทุกประเภท"
→ search หลายรอบ
→ สรุปเปรียบเทียบ
```

### สเปกตรัมทั้งหมด — 5 Levels

```
Level 0          Level 1          Level 2          Level 3          Level 4
LLM เปล่า ──────▶ RAG ───────────▶ Advanced RAG ──▶ Agentic RAG ──▶ Full Agent

Level 0: ไม่มี DB ไม่มี search ตอบจากความรู้เดิม
Level 1: มี DB + search ตายตัว
Level 2: มี DB + search ดีขึ้น (hybrid, rerank)
Level 3: มี DB + search + ตัดสินใจเอง (Agentic RAG)
Level 4: มี DB + tools เยอะ + วางแผน + ลงมือทำ (Full Agent)
```

## ตัวอย่าง / กรณีศึกษา

### Use Cases

#### 1. Documentation Search (RAG พอ)
```
Q: "ลาพักร้อนกี่วัน?"
→ search → ตอบ
```

#### 2. Complex Comparison (ต้อง Agentic RAG)
```
Q: "เปรียบเทียบนโยบายลาทุกประเภท"
→ Iteration 1: search "ลาพักร้อน"
→ Iteration 2: search "ลาป่วย"
→ Iteration 3: search "ลากิจ"
→ Iteration 4: search "ลาคลอด"
→ สรุปเปรียบเทียบ
```

#### 3. Auto Leave Request (ต้อง Full Agent)
```
Q: "ช่วยลาพักร้อนให้ด้วย"
→ ค้นนโยบาย (RAG)
→ คำนวณวันลาเหลือ
→ ตรวจว่าตรงนโยบายมั้ย
→ สร้างใบลา
→ ส่งอนุมัติหัวหน้า
→ แจ้งผล
```

### สำหรับ Sellsuki — ทำแค่ไหนดี

```
Phase 1: RAG (สัปดาห์ 1-4)
✅ ตอบคำถามเกี่ยวกับบริษัทได้
✅ ง่าย debug ง่าย
✅ Cost ต่ำ
✅ พอสำหรับ 80% ของ use cases

Phase 2: Agentic RAG (สัปดาห์ 5-6)
เพิ่ม agency:
- ให้ LLM เลือกว่าจะ search หรือไม่
- ให้เลือก category (HR, Product, IT)
- ให้ loop ได้ถ้าข้อมูลไม่พอ

Phase 3: Full Agent (อนาคต)
เพิ่ม action tools:
- ส่ง email
- สร้าง ticket
- จอง meeting
- คำนวณ
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/llm-large-language-model|LLM]] — Agent ใช้ LLM เป็น "สมอง"
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — Agent อาจใช้ RAG เป็น tool หนึ่ง
- [[wiki/concepts/agentic-rag|Agentic RAG]] — Agent ที่ใช้ RAG เป็น tool หลัก
- [[wiki/concepts/tool-calling|Tool Calling]] — กลไกที่ Agent ใช้เรียก external functions

## แหล่งที่มา

- Complete RAG & Agent Knowledge Base
- RAG vs Agent — Explained
