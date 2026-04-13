---
title: "RAG vs Agent — มันคืออะไร ต่างกันยังไง"
type: source
source_file: raw/notes/rag-knowledge/08-rag-vs-agent-explained.md
tags: [rag, agent, comparison, architecture, decision-making]
related: [wiki/concepts/rag-retrieval-augmented-generation, wiki/concepts/ai-agent]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/rag-knowledge/08-rag-vs-agent-explained.md|Original file]]

## สรุป

คู่มืออธิบายความแตกต่างระหว่าง RAG และ Agent อย่างละเอียด — พร้อมตัวอย่าง code, flow diagram, spectrum ทั้งหมด (5 levels), และแนะนำสำหรับ Sellsuki

## ประเด็นสำคัญ

### คำตอบสั้นๆ

- **"RAG เป็น Agent ตัวนึงมั้ย?"** → ไม่ — RAG ไม่ใช่ Agent
  - RAG = เทคนิค (technique) ในการดึงข้อมูลมาช่วย LLM ตอบ
  - Agent = ระบบ AI ที่ตัดสินใจเอง เลือกว่าจะทำอะไร
  - RAG เป็นแค่ "ท่อน้ำ" | Agent เป็น "คนที่คิดเอง"

- **"AI + Vector Database ถือว่าเป็น RAG หรือ Agent?"** → ขึ้นกับว่า "ทำงานยังไง"
  - Flow: คำถาม → search vector DB → ส่ง context → LLM → ตอบ = **RAG**
  - Flow: คำถาม → LLM ตัดสินใจว่าจะ search อะไร → search → LLM ดูผลว่าพอมั้ย → ถ้าไม่พอ search อีก → ตอบ = **Agentic RAG**

- **"ทำ RAG เสร็จถือว่าเป็น Agent มั้ย?"** → ไม่ครับ
  - RAG ที่ทำเสร็จ = ระบบที่ search + LLM ตอบ ตาม flow ตายตัว
  - ถ้าอยากให้เป็น Agent ต้องเพิ่ม: decision making, tool selection, self-evaluation, loop

### Spectrum ทั้งหมด — 5 Levels

```
Level 0          Level 1          Level 2          Level 3          Level 4
LLM เปล่า ──────▶ RAG ───────────▶ Advanced RAG ──▶ Agentic RAG ──▶ Full Agent
```

#### Level 0: LLM เปล่า (ไม่ใช่ RAG ไม่ใช่ Agent)
- ไม่มี database, ไม่มี search, ไม่มีการตัดสินใจ
- ตอบจากความรู้ที่ train มาเท่านั้น

#### Level 1: RAG พื้นฐาน (เป็น RAG ไม่ใช่ Agent)
- มี database + search + LLM
- Flow ตายตัว: search → stuff → LLM
- ไม่มีการตัดสินใจ

#### Level 2: Advanced RAG (ยังเป็น RAG ไม่ใช่ Agent)
- Query rewriting, Hybrid search, Re-rank, Context compression
- ยังเป็น fixed pipeline

#### Level 3: Agentic RAG (เป็นทั้ง RAG และ Agent)
- LLM ตัดสินใจเอง: "ต้อง search หรือไม่?", "search อะไร?", "พอมั้ย?"
- เลือก tool เอง, Loop ได้

#### Level 4: Full Agent (Agent เต็มรูปแบบ)
- Multiple tools (search + actions)
- วางแผน, Memory, ลงมือทำ (send email, book meeting)

### ตารางเปรียบเทียบ

| คุณสมบัติ | LLM เปล่า | RAG | Agentic RAG | Full Agent |
|-----------|-----------|-----|-------------|------------|
| มี LLM | ✅ | ✅ | ✅ | ✅ |
| มี Database | ❌ | ✅ | ✅ | ✅ (optional) |
| Search | ❌ | ✅ ตายตัว | ✅ เลือกเอง | ✅ เลือกเอง |
| ตัดสินใจ | ❌ | ❌ | ✅ | ✅ |
| Loop | ❌ | ❌ | ✅ | ✅ |
| หลาย Tools | ❌ | ❌ (search 1) | ✅ (search หลาย) | ✅ (search + actions) |
| วางแผน | ❌ | ❌ | บางส่วน | ✅ |
| ลงมือทำ | ❌ | ❌ | ❌ | ✅ |
| Memory | ❌ | ❌/บางส่วน | ✅ | ✅ |

### ความสัมพันธ์ระหว่าง RAG กับ Agent

```
RAG และ Agent ไม่ได้ตรงข้ามกัน — มันอยู่คนละแกน:

แกน X: เทคนิคการดึงข้อมูล
  ไม่มี ──────── RAG ──────── Advanced RAG

แกน Y: ระดับ Agency (ตัดสินใจเอง)
  Fixed pipeline ──────── Agentic ──────── Full Agent

สรุป:
  RAG ที่ไม่มี agency         = RAG ธรรมดา (Level 1-2)
  RAG ที่มี agency            = Agentic RAG (Level 3)
  Agent ที่ไม่ใช้ RAG          = Agent (เช่น coding agent)
  Agent ที่ใช้ RAG ด้วย       = Agent + RAG (Level 4)
```

### Flow จริงเทียบกัน

#### RAG Flow (ไม่มี agency)
```
User: "ลาพักร้อนกี่วัน?"
│
▼ [Embed คำถาม] (ขั้นตอนที่ 1 ตายตัว)
▼ [Search vector DB → top 5 chunks] (ALWAYS search)
▼ [Stuff chunks เข้า prompt] (ALWAYS stuff ทุก chunks)
▼ [LLM Generate answer] (ALWAYS ส่งให้ LLM)
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

## สำหรับ Sellsuki — ทำแค่ไหนดี

```
คำตอบ: เริ่มจาก RAG แล้วค่อยขยับเป็น Agentic RAG

Phase 1: RAG (สัปดาห์ 1-4)
  ✅ ตอบคำถามเกี่ยวกับบริษัทได้
  ✅ ง่าย debug ง่าย
  ✅ Cost ต่ำ (fixed pipeline)
  ✅ พอสำหรับ 80% ของ use cases

Phase 2: Agentic RAG (สัปดาห์ 5-6)
  เพิ่ม agency:
  - ให้ LLM เลือกว่าจะ search หรือไม่
  - ให้เลือก category (HR, Product, IT)
  - ให้ loop ได้ถ้าข้อมูลไม่พอ

Phase 3: Full Agent (อนาคต)
  เพิ่ม action tools:
  - ส่ง email, สร้าง ticket, จอง meeting, คำนวณ

ไม่ต้องกระโดดไป Full Agent ทันที
เริ่มจาก RAG → ใช้งานได้แล้ว → ค่อยเพิ่ม agency
```

## อุปมาให้เข้าใจง่าย

```
LLM เปล่า = คนฉลาดที่ถูกปิดตา
  ฉลาดมาก แต่ไม่เห็นข้อมูลบริษัท

RAG = คนฉลาดที่มีหนังสือวางข้างๆ
  ถาม → เปิดหนังสือหน้าที่เกี่ยวข้อง → อ่าน → ตอบ
  ทำเหมือนกันทุกครั้ง (fixed pipeline)

Agentic RAG = คนฉลาดที่มีห้องสมุด
  ถาม → คิดก่อนว่า "ต้องหาอะไร?"
  → เลือกหนังสือที่เหมาะ → อ่าน → ยังไม่พอ? → เลือกเล่มอื่น → ตอบ

Full Agent = พนักงานเต็มตัว
  ถาม → คิด → ค้นข้อมูล → ส่ง email → จอง meeting → ทำรายงาน
  ← ลงมือทำให้จริงๆ!
```

## สรุปสุดท้าย

```
┌─────────────────────────────────────┐
│ RAG ≠ Agent                          │
│ RAG = เทคนิคการดึงข้อมูล         │
│ Agent = ระบบที่คิดเอง              │
│                                     │
│ RAG อาจเป็นส่วนหนึ่งของ Agent ได้ │
│                                     │
│ ทำ RAG เสร็จ ≠ ได้ Agent           │
│ ทำ RAG เสร็จ = ได้ RAG             │
│                                     │
│ AI + VectorDB                        │
│ + fixed flow = RAG                   │
│ + ตัดสินใจเอง = Agentic RAG        │
└─────────────────────────────────────┘
```

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG — Retrieval-Augmented Generation]]
- [[wiki/concepts/ai-agent|AI Agent]]
- [[wiki/concepts/agentic-rag|Agentic RAG]]
