---
title: "Prompt Engineering — ออกแบบคำสั่งให้ LLM"
type: concept
tags: [prompt-engineering, llm, few-shot, chain-of-thought, system-prompt, rag]
sources: [wiki/sources/rag-complete-knowledge, wiki/sources/step1-langchain-basics, wiki/sources/ai-context-phase8]
related: [wiki/concepts/llm-large-language-model, wiki/concepts/deterministic-ai-generation, wiki/concepts/ai-agent]
created: 2026-04-14
updated: 2026-04-14
---

## สรุปสั้น

Prompt Engineering คือศาสตร์ของการเขียนคำสั่ง (prompts) ให้ LLM เพื่อให้ได้ผลลัพธ์ที่ต้องการ — ทั้งในแง่ความถูกต้อง รูปแบบ ภาษา และความสม่ำเสมอ ถือเป็นทักษะสำคัญสำหรับนักพัฒนา AI

## อธิบาย

### โครงสร้าง Prompt ที่ดี

```
System Prompt: บอก LLM ว่าเป็นใคร บทบาทคืออะไร ข้อจำกัดอะไรบ้าง
User Message: คำถามหรือ task จริงๆ
Context (RAG): ข้อมูลที่ดึงมา inject เข้าไป
Examples: ตัวอย่าง input/output ที่ต้องการ
```

### Prompt Patterns หลัก

**1. Zero-shot** — ไม่ให้ตัวอย่าง
```
"สรุปบทความนี้เป็นภาษาไทยใน 3 ประโยค: {article}"
```

**2. Few-shot** — ให้ตัวอย่าง 2-5 ชุด
```
"จัดหมวดหมู่ feedback:
Positive: 'บริการดีมาก' → positive
Negative: 'ช้ามาก' → negative
คำถาม: '{customer_feedback}' → "
```

**3. Chain-of-Thought (CoT)** — ให้คิดก่อนตอบ
```
"คิดทีละขั้น แล้วตอบ: {complex_question}
Let's think step by step:"
```

**4. Role Prompting** — กำหนดบทบาท
```
"คุณเป็น HR Specialist ของ Sellsuki ที่เชี่ยวชาญเรื่อง labor law ไทย
ตอบคำถามพนักงานด้วยข้อมูลที่ถูกต้องและกระชับ"
```

## ประเด็นสำคัญ

### Prompt สำหรับ RAG

```python
from langchain_core.prompts import ChatPromptTemplate

template = ChatPromptTemplate.from_messages([
    ("system", """คุณเป็น AI Assistant ของ Sellsuki
ตอบคำถามโดยใช้ข้อมูลที่ให้มาเท่านั้น
ถ้าไม่มีข้อมูลในเอกสาร ให้บอกว่า "ไม่มีข้อมูลในระบบ"
ห้ามตอบจากความรู้ตัวเอง"""),
    ("human", """เอกสารอ้างอิง:
{context}

คำถาม: {question}""")
])
```

### Constraint Prompting — ได้ผลลัพธ์ที่ predictable

สำหรับ structured output ที่ต้องการ format ตายตัว:

```python
template = """ตอบเป็น JSON เท่านั้น ตาม schema นี้:
{
  "answer": "คำตอบ",
  "confidence": 0.0-1.0,
  "sources": ["source1", "source2"]
}

คำถาม: {question}
ข้อมูล: {context}"""
```

### Output Format Control

```
"ตอบเป็น bullet points ไม่เกิน 5 ข้อ ภาษาไทย"
"ตอบด้วยตาราง markdown 3 columns: ข้อดี | ข้อเสีย | แนะนำ"
"ตอบภาษาไทย แต่ใช้ English สำหรับ technical terms"
```

## ตัวอย่าง / กรณีศึกษา

**Sellsuki HR Bot:** System prompt กำหนดให้ตอบภาษาไทยสุภาพ อ้างอิงเฉพาะจากเอกสาร HR ที่ inject เข้ามา และบอกแหล่งที่มาทุกครั้ง — ป้องกัน hallucination และให้ trace ได้ว่าข้อมูลมาจากไหน

**Deterministic README Generator:** ใช้ Fixed Template + Data Extraction + Constraint Prompting ทำให้ output เหมือนกันทุกครั้งที่รันกับ codebase เดียวกัน

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/llm-large-language-model|LLM]] — Prompt Engineering คือวิธีสื่อสารกับ LLM ให้มีประสิทธิภาพ
- [[wiki/concepts/deterministic-ai-generation|Deterministic AI Generation]] — Constraint Prompting เป็นหนึ่งใน 3 เทคนิคหลัก
- [[wiki/concepts/ai-agent|AI Agent]] — System prompt ใน agent กำหนดบทบาท, tools ที่ใช้, และ stopping conditions

## แหล่งที่มา

- [[wiki/sources/rag-complete-knowledge|Complete RAG & Agent Knowledge Base]]
- [[wiki/sources/step1-langchain-basics|Step 1: LangChain พื้นฐาน]]
- [[wiki/sources/ai-context-phase8|AI Context Phase 8: Deterministic README Generator]]
