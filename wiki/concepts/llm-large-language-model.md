---
title: "LLM — Large Language Model"
type: concept
tags: [llm, ai, machine-learning, nlp, transformer, tokenization, training]
sources: [wiki/sources/rag-complete-knowledge, wiki/sources/rag-glossary-data-prep, wiki/sources/ai-engineering/llm-fundamentals]
related: [wiki/concepts/rag-retrieval-augmented-generation, wiki/concepts/embedding, wiki/concepts/fine-tuning, wiki/concepts/context-engineering]
created: 2026-04-13
updated: 2026-04-14
---

## สรุปสั้น

LLM (Large Language Model) คือ โมเดล AI ที่เรียนรู้จากข้อมูลข้อความมหาศาล และสามารถเข้าใจและสร้างภาษาได้ — ทำงานโดยการทำนาย "คำถัดไป" จาก patterns ที่เรียนรู้มา

## อธิบาย

### LLM ทำงานยังไง

LLM = เครื่องทำนาย "คำถัดไป"

```
Input:  "แมวนั่งอยู่บน"
LLM คิด: คำถัดไปน่าจะเป็น...
  "โต๊ะ"    → 35% probability
  "เก้าอี้"  → 25% probability
  "พื้น"    → 20% probability
  ...

เลือก "โต๊ะ" → แล้วต่อ: "แมวนั่งอยู่บนโต๊ะ"
แล้วทำนายคำถัดไปอีก: "ในครัว"
```

### ทำไมมัน "ฉลาด" ได้

- Train จากข้อมูลมหาศาล (หนังสือ, เว็บไซต์, โค้ด)
- เรียนรู้ patterns ของภาษาและความรู้ทั้งหมด
- ไม่ได้แค่ทำนายคำ แต่เข้าใจ "ความหมาย" ระดับหนึ่ง

### สิ่งสำคัญของ LLM

#### Token
- **คือ**: หน่วยเล็กสุดที่ LLM อ่าน/สร้าง
- **ภาษาอังกฤษ**: 1 คำ ≈ 1-1.5 tokens ("hello" = 1 token)
- **ภาษาไทย**: 1 คำ ≈ 2-4 tokens ("สวัสดี" ≈ 3 tokens)
- **ทำไมสำคัญ**: ทุกอย่างคิดเงินเป็น token
  - GPT-4o-mini: $0.15 / 1 ล้าน input tokens

#### Context Window
- **คือ**: จำนวน tokens สูงสุดที่ LLM เห็นใน 1 ครั้ง
- **ตัวอย่าง**:
  - GPT-4o: 128,000 tokens (~300 หน้า)
  - Claude: 200,000 tokens (~500 หน้า)
  - Gemini 1.5: 1,000,000 tokens (~2,500 หน้า)
- **สำคัญกับ RAG**:
  - ถ้า context window ใหญ่ → ยัด documents เข้าไปได้เยอะ
  - แต่ยิ่งเยอะ → ยิ่งแพง และ LLM อาจ "หลง"

#### Temperature
- **คือ**: ความสุ่มของคำตอบ
- **ค่าแนะนำ**:
  - **temperature = 0**: ตอบเหมือนเดิมทุกครั้ง (deterministic) — **เหมาะ: RAG, fact-based Q&A**
  - **temperature = 0.7**: สุ่มปานกลาง (creative) — เหมาะ: เขียนบทความ
  - **temperature = 1.0**: สุ่มมาก — เหมาะ: brainstorming

#### System Prompt
- **คือ**: คำสั่งที่บอก LLM ว่า "คุณคือใคร ทำอะไร"
- **ตัวอย่าง**:
```
คุณเป็น AI Assistant ของบริษัท Sellsuki
ตอบเฉพาะข้อมูลที่ได้รับ ห้ามเดา
ตอบเป็นภาษาไทย สุภาพ
```

### Completion vs Chat

#### Completion API (เก่า)
```
Input:  "เมืองหลวงของไทยคือ"
Output: "กรุงเทพมหานคร"
→ ใส่ข้อความ → ได้ข้อความต่อ
```

#### Chat API (ปัจจุบัน)
```
Input: [
  {"role": "system", "content": "คุณเป็น AI ตอบคำถามภาษาไทย"},
  {"role": "user", "content": "เมืองหลวงของไทยคืออะไร?"},
]
Output: {"role": "assistant", "content": "เมืองหลวงของไทยคือกรุงเทพมหานคร"}
→ เป็นบทสนทนา มี roles
```

#### Roles
- **system**: คำสัั่งลับที่ user ไม่เห็น (บอก LLM ว่าเป็นใคร)
- **user**: ข้อความจาก user
- **assistant**: คำตอบจาก LLM
- **tool**: ผลลัพธ์จาก tool call

## ประเด็นสำคัญ

### Hallucination (LLM เดาเอง)

**คือ**: LLM สร้างข้อมูลที่ไม่จริงขึ้นมาเอง

**ตัวอย่าง**:
```
Q: "CEO ของ Sellsuki คือใคร?"
LLM (ไม่มี context): "CEO ของ Sellsuki คือคุณสมชาย ใจดี
                      ก่อตั้งเมื่อปี 2015"
← เดามาหมด! อาจไม่จริง!
```

**วิธีลด Hallucination ใน RAG**:
1. Prompt ชัดเจน: "ตอบเฉพาะจาก context ห้ามเดา"
2. Temperature = 0 (ไม่สุ่ม)
3. ให้ LLM บอก confidence level
4. ให้ LLM อ้างอิง source
5. ถ้าไม่มีข้อมูล → บอกว่าไม่รู้ (ดีกว่าเดา)

### Grounding

**คือ**: การ "ยึด" คำตอบ LLM กับข้อมูลจริง

RAG เป็น grounding technique อย่างหนึ่ง — เพราะบังคับให้ LLM ตอบจาก context ที่ค้นมา

## ตัวอย่าง / กรณีศึกษา

### ใช้ใน RAG System

```python
# RAG System Prompt ที่ดี
RAG_SYSTEM_PROMPT = """คุณเป็น "Suki Bot" AI Assistant ของบริษัท Sellsuki

## กฎการตอบ:
1. ตอบ **เฉพาะ** จากข้อมูลใน [Context] เท่านั้น
2. ถ้าไม่มีข้อมูลเพียงพอ → บอกตรงๆ ว่า "ไม่พบข้อมูลเรื่องนี้"
3. ห้ามเดา ห้ามสร้างข้อมูลเอง
4. อ้างอิงแหล่งที่มาเสมอ

## รูปแบบการตอบ:
- ภาษาไทย สุภาพ เป็นกันเอง
- ตอบกระชับ ตรงประเด็น
- ถ้ามีตัวเลขสำคัญ ใส่ให้ชัดเจน
- ลงท้ายด้วยแหล่งอ้างอิง"""
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — LLM เป็นส่วนหนึ่งของ RAG System
- [[wiki/concepts/embedding|Embedding]] — การแปลงข้อความเป็นตัวเลขสำหรับ LLM
- [[wiki/concepts/ai-agent|AI Agent]] — LLM เป็น "สมอง" ของ Agent
- [[wiki/concepts/prompt-engineering|Prompt Engineering]] — วิธีคุยกับ LLM ให้ตอบตามต้องการ

## Transformer Architecture (เพิ่มเติม)

สถาปัตยกรรม Transformer ที่ LLM ทุกตัวใช้ (สร้างปี 2017):

```
Input Text → [Tokenizer] → [Embedding Layer] → [Attention Layers × N] → [Feed-Forward] → Output Token
```

**Self-Attention**: กลไกที่ทำให้โมเดลรู้ว่าคำไหนเกี่ยวข้องกับคำไหน
```
"แมวกินปลา เพราะมันหิว"
คำว่า "มัน" → Attention weight สูงกับ "แมว" (0.85) — รู้ว่า "มัน" หมายถึงแมว
```

**Flash Attention**: เทคนิคเพิ่มประสิทธิภาพ เร็วขึ้น 2-4x โดยไม่เสียความแม่นยำ

## Sampling Parameters

| Parameter | ค่า | ผล |
|-----------|-----|-----|
| **Temperature** | 0.0 | Deterministic ทุกครั้ง |
| | 0.5 | สมดุล (แนะนำงานทั่วไป) |
| | 1.0 | Default |
| | 2.0 | สร้างสรรค์มาก ไม่ consistent |
| **Top-P** | 0.9 | เลือกจาก tokens ที่รวม prob ถึง 90% |
| **Top-K** | 50 | เลือกจาก 50 tokens prob สูงสุด |

## Training Stages

1. **Pre-training**: Internet ทั้งหมด (Trillion tokens) → Base Model (รู้ภาษา แต่ตอบคำถามไม่เก่ง)
2. **SFT (Supervised Fine-Tuning)**: คู่ (คำถาม, คำตอบดี) จากมนุษย์ → Instruct Model
3. **RLHF**: มนุษย์จัดอันดับ "A ดีกว่า B" → ใช้ PPO Algorithm → Assistant Model (Claude, GPT-4)
4. **Constitutional AI** (Anthropic): โมเดลวิจารณ์ตัวเองตาม "Constitution" ลดพึ่งมนุษย์

## ข้อจำกัดสำคัญ

- **Lost in the Middle**: โมเดลจำข้อมูลต้น-ท้าย context ดีกว่าตรงกลาง → ใส่ข้อมูลสำคัญที่ปลาย
- **Positional Bias**: มีแนวโน้มเลือกตัวเลือกแรกหรือสุดท้ายใน multiple choice
- **Sycophancy**: ตอบเพื่อเอาใจ user ไม่ใช่ตอบความจริง → แก้ด้วย system prompt
- **Context Poisoning**: RAG ดึงเอกสารผิด → โมเดลตอบผิดตาม ("ขยะเข้า ขยะออก")

## แหล่งที่มา

- RAG Agent Complete Knowledge Base
- RAG Glossary & Data Preparation Guide
- [[wiki/sources/ai-engineering/llm-fundamentals|LLM Fundamentals — พื้นฐาน Large Language Models]]
