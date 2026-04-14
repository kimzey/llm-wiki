---
title: "Prompt Engineering — ออกแบบคำสั่งให้ LLM"
type: concept
tags: [prompt-engineering, llm, few-shot, chain-of-thought, system-prompt, rag, meta-prompting, apo]
sources: [wiki/sources/rag-complete-knowledge, wiki/sources/step1-langchain-basics, wiki/sources/ai-context-phase8, wiki/sources/ai-engineering/advanced-techniques, wiki/sources/ai-engineering/context-prompt-engineering]
related: [wiki/concepts/llm-large-language-model, wiki/concepts/deterministic-ai-generation, wiki/concepts/ai-agent, wiki/concepts/context-engineering]
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

## Advanced Techniques (เพิ่มเติมจาก 08_advanced_techniques.md)

### Meta-Prompting
ใช้ LLM เขียน prompt ที่ดีที่สุดสำหรับงาน แทนที่จะเขียนเอง:
```python
META_PROMPT = """คุณเป็นผู้เชี่ยวชาญด้าน Prompt Engineering
สร้าง prompt ที่ดีที่สุดสำหรับงานนี้:
งาน: {task_description}
ส่ง prompt เท่านั้น ไม่ต้องอธิบาย"""
```

### Automatic Prompt Optimization (APO)
วนรอบอัตโนมัติ:
1. evaluate prompt บน test set
2. ถ้า score ต่ำ → ให้ LLM วิเคราะห์ failure
3. ให้ LLM แก้ prompt
4. ทดสอบ prompt ใหม่ → ทำซ้ำจนได้ score ที่พอใจ

### Structured Output — 2 วิธี

**JSON Mode**: ระบุ schema ใน prompt → validate ด้วย Pydantic
```python
class ProductReview(BaseModel):
    sentiment: Literal["positive", "negative", "neutral", "mixed"]
    score: int  # 1-10
    pros: list[str]
    cons: list[str]
```

**Tool-forced Extraction**: บังคับให้โมเดลใช้ tool เพื่อส่ง structured output
```python
response = client.messages.create(
    tool_choice={"type": "any"},  # บังคับใช้ tool
    ...
)
```

### Constitutional AI Prompting
```python
# Step 1: สร้างคำตอบ → Step 2: วิจารณ์ตาม constitution → Step 3: แก้ไข
CONSTITUTION = "ตอบอย่างซื่อสัตย์, ไม่สร้างเนื้อหาอันตราย, ยอมรับความไม่แน่นอน"
```

## แหล่งที่มา

- [[wiki/sources/rag/rag-complete-knowledge|Complete RAG & Agent Knowledge Base]]
- [[wiki/sources/langchain/steplangchain-basics|Step 1: LangChain พื้นฐาน]]
- [[wiki/sources/ai-context/ai-context-phase8|AI Context Phase 8: Deterministic README Generator]]
- [[wiki/sources/ai-engineering/advanced-techniques|Advanced Techniques — เทคนิคขั้นสูง AI Engineering]]
- [[wiki/sources/ai-engineering/context-prompt-engineering|Context & Prompt Engineering — คู่มือฉบับสมบูรณ์]]
