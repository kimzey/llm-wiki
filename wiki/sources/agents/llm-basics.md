---
title: "LLM Basics — พื้นฐานก่อนเรียน Agent"
type: source
source_file: raw/notes/ai/agents/00-llm-basics.md
tags: [llm, basics, tokens, context-window, rag, prompting]
related: []
created: 2026-04-14
updated: 2026-04-14
---

> **Full source**: [[../../../raw/notes/ai/agents/00-llm-basics.md|Original file]]

## สรุป

เอกสารพื้นฐานเกี่ยวกับ LLM (Large Language Models) ที่จำเป็นต้องเข้าใจก่อนศึกษาเรื่อง Agents ครอบคลุมตั้งแต่ LLM ทำงานยังไง, tokens, context window, prompting techniques, RAG จนถึงทำไมต้องมี Agent

## ประเด็นสำคัญ

### 1. LLM ทำงานยังไง — อธิบายแบบไม่ใช้ Math

**LLM ทำสิ่งเดียว: รับข้อความ → ทำนายคำต่อไปที่น่าจะตามมา → ทำซ้ำจนครบ**

- เป็น **autoregressive generation** — ทำนายทีละคำแล้วใช้ผลลัพธ์เป็น input ต่อไป
- LLM **ไม่ได้ "รู้" หรือ "เข้าใจ"** — มันแค่ทำนายความน่าจะเป็น
- ทำนองเดียวกับการจำรูปแบบจาก training data

> [!example] ตัวอย่าง
> Input: "กรุงเทพเป็นเมืองหลวงของ"
> LLM: ทำนาย → "ประเทศ" (prob 80%) → "ไทย" (prob 95%)
> Output: "กรุงเทพเป็นเมืองหลวงของประเทศไทย"

### 2. Tokens — หน่วยที่ LLM ใช้จริง

**Token คือ chunk ของข้อความ** ที่พบบ่อยพอที่จะมีในพจนานุกรม

- ภาษาไทยใช้ tokens มากกว่าภาษาอังกฤษ **~2-3×** ต่อความหมายเดียวกัน
- ราคาคิดตาม tokens (input + output)
- 1 token ≈ 4 ตัวอักษรอังกฤษ หรือ 1-2 คำไทย

> [!example] ตัวอย่าง tokens
> - "Hello world" → ["Hello", " world"] = 2 tokens
> - "กรุงเทพมหานคร" → ["กรุง", "เทพ", "มหา", "นคร"] ≈ 4 tokens
> - "สวัสดีครับ" → ["ส", "วัส", "ดี", "ครับ"] ≈ 4 tokens

### 3. Context Window — หน่วยความจำชั่วคราว

**Context window = "กระดาษ A4" ที่ LLM ใช้ทำงาน**

- ทุกอย่างที่ LLM "เห็น" ต้องอยู่บนกระดาษนี้:
  - System prompt
  - Conversation history
  - Tool definitions
  - Tool results
  - Output ที่กำลังเขียนอยู่
- claude-sonnet-4-6 = 200,000 tokens ≈ หนังสือ 600 หน้า

**ปัญหาเมื่อ context เต็ม:**
1. **Overflow**: token เกิน limit → API error หรือตัดอัตโนมัติ
2. **Lost in the middle**: LLM ให้ความสนใจต้น/ปลาย มากกว่าตรงกลาง
3. **Cost**: ยิ่งยาว ยิ่งแพง
4. **Latency**: ยิ่งยาว ยิ่งช้า

> [!warning] Stateless — สิ่งที่คนสับสนบ่อย
> LLM เป็น **stateless** — ไม่มี "ความจำ" ของตัวเอง
> - แต่ละ API call เป็นอิสระต่อกัน
> - ต้องส่ง history ทั้งหมดทุกครั้ง
> - ถ้าไม่ส่ง → LLM ลืมทุกอย่าง

### 4. วิธีสื่อสารกับ LLM — Prompt Anatomy

**3 Roles ใน messages:**

```python
messages = [
    {"role": "system", "content": "คำสั่งเริ่มต้น — LLM เป็นใคร ทำอะไรได้"},
    {"role": "user", "content": "ข้อความจาก user หรือ tool result"},
    {"role": "assistant", "content": "ข้อความจาก LLM (history)"}
]
```

**System prompt คือสิ่งที่ควบคุม LLM ที่ทรงพลังที่สุด** — ถ้าเขียนดี agent จะทำงานตามที่ต้องการ

### 5. Prompting Techniques

**Chain-of-Thought (CoT)** — บังคับให้ LLM เขียน reasoning steps ก่อนตอบ

> [!example] CoT ใช้ยังไง
> Prompt: "Let's think step by step."
>
> ผลลัพธ์: LLM generate intermediate steps ก่อนถึงคำตอบ
> - ทำให้แม่นขึ้นในงาน multi-step reasoning
> - ทำไมได้ผล: LLM generate ทีละ token → การเขียน intermediate steps บังคับให้แต่ละ token "มีพื้นที่คิด"

**Temperature** — ควบคุมความ "สร้างสรรค์"
- 0 → deterministic, เลือก token ที่ probability สูงสุดเสมอ (code gen, factual Q&A)
- 0.7 → balanced, default (general conversation)
- 1.0+ → creative, หลากหลาย แต่อาจ inconsistent (creative writing, brainstorming)

### 6. RAG — ทำไมต้องมี และทำงานยังไง

**ปัญหาของ LLM ล้วนๆ:**
1. Knowledge cutoff: ไม่รู้ข้อมูลหลัง cutoff
2. Private knowledge: ไม่รู้นโยบายบริษัทของเรา
3. Hallucination: ถ้าถามเรื่องที่ไม่รู้ → ตอบมั่ว

**RAG = Retrieval-Augmented Generation** — ก่อนตอบ → ดึงข้อมูลที่เกี่ยวข้องมาให้ LLM อ่านก่อน

**2 Phase:**

**Phase 1: INDEXING (ทำครั้งเดียวล่วงหน้า)**
1. Chunking — แบ่งเป็น chunks เล็กๆ
2. Embedding — แปลงข้อความเป็น vector
3. Vector DB — เก็บ vectors + metadata

**Phase 2: RETRIEVAL + GENERATION (ทำทุกครั้ง)**
1. Embed Query → vector
2. Vector Search — หา chunks ที่ vector ใกล้เคียงสุด
3. LLM + Retrieved Chunks → Answer

**Embedding คืออะไร:**
- การแปลงข้อความเป็น vector ของตัวเลข
- "ความหมายคล้ายกัน" จะให้ vector ที่ใกล้กัน
- ใช้สำหรับ **semantic search** (ความหมาย ไม่ใช่ keyword matching)

### 7. ทำไม RAG ไม่พอ → ทำไมต้องมี Agent

**RAG ทำได้แค่:**
- Query → Retrieve → Generate (1 ครั้ง, แหล่งเดียว, ไม่มี loop)

**RAG ไม่พอสำหรับ:**
1. คำถามซับซ้อนที่ต้องดึงหลาย sources
2. Multi-step tasks (ดึงข้อมูล → คำนวณ → เขียน email → ส่ง)
3. งานที่ต้องตัดสินใจว่าจะทำอะไรต่อ (debug code อาจต้อง search → run → search อีก)
4. Calculation และ real-time data

**Agent แก้โดยเพิ่ม 3 อย่างบน RAG:**
1. **Tools**: ไม่ใช่แค่ retrieve — รัน code ได้, call API ได้, ส่ง email ได้
2. **Loop**: ไม่ใช่แค่ 1 ครั้ง — ทำซ้ำจนงานเสร็จ
3. **Planning**: ตัดสินใจเองว่าจะทำอะไร step ถัดไป

**Evolution:**
- Level 1: LLM ล้วนๆ (ตอบจาก training data เท่านั้น)
- Level 2: RAG (ดึงข้อมูล private/real-time)
- Level 3: Agent (ใช้ tools หลายอย่าง, วนซ้ำได้)
- Level 4: Multi-Agent (แบ่งงานให้ specialists)

## ข้อมูล / หลักฐาน ที่น่าสนใจ

### Token Pricing ตัวอย่าง

claude-sonnet-4-6:
- Input: $3 per 1M tokens ≈ 750,000 คำภาษาอังกฤษ
- Output: $15 per 1M tokens

ตัวอย่าง:
- "ช่วยเขียน email สั้นๆ" + response ~300 words ≈ 500 tokens ≈ $0.006
- Agent 10 iterations × 1000 tokens = 10,000 tokens ≈ $0.18
- แต่ถ้า 1000 users/day = **$180/day**

### Vector Search vs Keyword Search

- **Keyword Search**: ค้นหา "dog" → หา docs ที่มีคำว่า "dog" → ไม่เจอถ้าถามว่า "สัตว์เลี้ยง"
- **Vector Search**: ค้นหา "สัตว์เลี้ยง" → embed → หา docs ที่ vector ใกล้ → เจอ dog, cat, rabbit แม้ไม่มีคำว่า "สัตว์เลี้ยง"

## Concepts ที่เกี่ยวข้อง

- [[LLM]] - Large Language Models
- [[Tokens]] - หน่วยที่ LLM ใช้
- [[Context Window]] - ขีดจำกัดความจำของ LLM
- [[Prompt Engineering]] - เทคนิคการเขียน prompt
- [[Chain-of-Thought]] - CoT prompting
- [[RAG]] - Retrieval-Augmented Generation
- [[Embeddings]] - Vector representations
- [[Vector Database]] - เก็บ embeddings
- [[Agent]] - LLM + Tools + Loop
