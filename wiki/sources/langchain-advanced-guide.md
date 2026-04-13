---
title: "LangChain Advanced Guide — LangGraph, LangSmith & RAG"
type: source
source_file: raw/notes/LangChain/langchain-advanced-guide.md
tags: [langchain, langgraph, langsmith, rag, advanced]
related: []
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/LangChain/langchain-advanced-guide.md|Original file]]

## สรุป

เอกสาร Advanced Guide สำหรับ LangChain ครอบคลุมหัวข้อขั้นสูง ได้แก่ LangGraph (ควบคุม Flow ของ Agent), LangSmith (ตรวจสอบและประเมินผล Agent), RAG (Retrieval-Augmented Generation), และการรวมทุกอย่างเข้าด้วยกัน

## ประเด็นสำคัญ

### 1. LangGraph

**LangGraph คืออะไร**
- Library ที่สร้างขึ้นมาบน LangChain เพื่อให้ควบคุม Flow ของ Agent ระดับ Low-level ได้อย่างละเอียด
- ใช้แนวคิดของ **Graph** (กราฟ) ที่ประกอบด้วย:
  - **Nodes (โหนด)**: แต่ละ Node คือ "ขั้นตอนการทำงาน" เช่น เรียก LLM, เรียก Tool, ตรวจสอบเงื่อนไข
  - **Edges (เส้นเชื่อม)**: กำหนดว่าจาก Node หนึ่งจะไปยัง Node ใดต่อ
  - **Conditional Edges**: เส้นเชื่อมแบบมีเงื่อนไข
  - **State (สถานะ)**: ข้อมูลที่ถูกส่งต่อระหว่าง Node

**ทำไมต้องใช้ LangGraph**
- Chatbot ง่ายๆ ถาม-ตอบ → LangChain (create_agent) เหมาะ
- Agent + Tools พื้นฐาน → LangChain เหมาะ
- **Branching Logic ซับซ้อน** → LangGraph เหมาะมาก
- **Human-in-the-loop** → LangGraph รองรับ
- **Multi-Agent** → LangGraph ออกแบบมาเพื่อสิ่งนี้
- **Retry / Error handling ละเอียด** → LangGraph ควบคุมได้ทุกจุด

**การติดตั้ง**
```bash
pip install -U langgraph langchain[google-genai]
```

**State และ Reducer**
- State = ข้อมูลที่จะถูกส่งต่อระหว่าง Node
- Reducer = ฟังก์ชันที่กำหนดว่าจะรวมข้อมูลใหม่เข้ากับข้อมูลเก่าอย่างไร
- `add_messages` reducer = APPEND messages ใหม่เข้าไป
- ถ้าไม่มี reducer = REPLACE (ค่าใหม่ทับค่าเก่า)

### 2. LangSmith

**LangSmith คืออะไร**
- Platform สำหรับ Tracing, Debugging, Testing, และ Monitoring LLM Applications
- ช่วยให้รู้ว่า Agent ทำอะไร ทำไมถึงตอบแบบนั้น และจะปรับปรุงอย่างไร

**ความสามารถ**
- **Tracing**: ดู trace ทุก step แบบ real-time
- **Evaluation**: วัดคุณภาพด้วย dataset + evaluator
- **Monitoring**: dashboard สำหรับ production
- **Prompt Management**: จัดการ prompt version

### 3. RAG (Retrieval-Augmented Generation)

**หัวข้อที่ครอบคลุม**
- Document Loaders
- Text Splitters
- Embeddings & Vector Stores
- Retrieval Strategies
- Generation
- Advanced Techniques

### 4. รวมทุกอย่างเข้าด้วยกัน — RAG Agent with LangGraph

- การนำ RAG มาใช้กับ LangGraph
- สร้าง Agent ที่สามารถค้นหาข้อมูลและตอบคำถามได้

## ข้อมูล / หลักฐาน ที่น่าสนใจ

**LangGraph vs LangChain Agent Comparison Table**:
- Chatbot ง่ายๆ: LangChain ✅ เหมาะ, LangGraph ❌ เกินความจำเป็น
- Branching Logic ซับซ้อน: LangChain ❌ ทำได้ยาก, LangGraph ✅ เหมาะมาก
- Human-in-the-loop: LangChain ❌ ไม่รองรับ, LangGraph ✅ รองรับ
- Multi-Agent: LangChain ❌ ทำได้ยาก, LangGraph ✅ ออกแบบมาเพื่อสิ่งนี้

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อมูลที่ขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/ai-agent|AI Agent]]
- [[wiki/concepts/agentic-rag|Agentic RAG]]
