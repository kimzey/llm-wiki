---
title: "LangGraph Deep Dive — ควบคุม Agent Flow อย่างละเอียด"
type: source
source_file: raw/notes/LangChain/langgraph-deep-dive.md
tags: [langgraph, deep-dive, state, nodes, edges, multi-agent]
related: []
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/LangChain/langgraph-deep-dive.md|Original file]]

## สรุป

เอกสาร Deep Dive สำหรับ LangGraph สอนทุกแนวคิดพื้นฐาน การใช้งาน และ best practices สำหรับสร้าง stateful, multi-actor applications ด้วย LLM โดยใช้แนวคิดของ Graph (Nodes + Edges)

## ประเด็นสำคัญ

### LangGraph vs LangChain Agent

```
LangChain create_agent:
  User → [LLM ↔ Tools loop] → Answer
  (ควบคุมน้อย, ใช้ง่าย, เหมาะงานตรงไปตรงมา)

LangGraph:
  User → [Node A] → (เงื่อนไข?) → [Node B] → [Node C] → Answer
                                  → [Node D] → [Node B] → ...
  (ควบคุมทุกจุด, ยืดหยุ่นสูง, เหมาะงานซับซ้อน)
```

### องค์ประกอบหลัก

```python
# 1. State  = ข้อมูลที่ไหลผ่าน Graph (เหมือน "กระดาษจดบันทึก")
# 2. Node   = ฟังก์ชันที่ทำงานบางอย่าง
# 3. Edge   = เส้นเชื่อมระหว่าง Node
# 4. Graph  = โครงสร้างรวมที่ compile แล้วพร้อมรัน
```

### State

**State คืออะไร**
- TypedDict ที่กำหนดโครงสร้างข้อมูลที่จะไหลผ่านทุก Node
- ทุก Node จะรับ State เข้ามา ทำงาน แล้ว return State ที่อัปเดตกลับไป

**Reducer คืออะไร**
- กำหนดว่าเมื่อ Node return ค่าใหม่ จะรวมกับค่าเก่าอย่างไร
- `add_messages` reducer → APPEND messages ใหม่เข้าไป
- ถ้าไม่มี reducer → REPLACE (ค่าใหม่ทับค่าเก่า)

**ประเภท Reducer**
- ไม่มี reducer → REPLACE
- `add_messages` → APPEND messages
- `operator.add` → รวม list เข้าด้วยกัน
- Custom reducer → lambda old, new: old + new

### หัวข้อที่ครอบคลุม

1. **Core Concepts** — แนวคิดพื้นฐาน
2. **State** — หัวใจของ LangGraph
3. **Nodes** — ขั้นตอนการทำงาน
4. **Edges** — เส้นทางการเชื่อมต่อ
5. **Prebuilt Components** — ของสำเร็จรูป
6. **Checkpointing** — บันทึก State
7. **Human-in-the-Loop** — คนอนุมัติ
8. **Subgraphs** — Graph ซ้อน Graph
9. **Multi-Agent Systems**
10. **Streaming** — แสดงผลแบบ Real-time
11. **ตัวอย่าง: Customer Support Bot สมบูรณ์**
12. **ตัวอย่าง: Research Agent**
13. **Deployment**

### ติดตั้ง

```bash
pip install -U langgraph "langchain[google-genai]"
```

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- **State** เป็นหัวใจของ LangGraph — ทุก Node อ่าน/เขียน State ได้
- **Reducer** คือสิ่งที่ทำให้ LangGraph แตกต่างจาก framework อื่น
- **Checkpointing** ทำให้ Agent จำบทสนทนาข้าม invoke ได้

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อมูลที่ขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/ai-agent.md|AI Agent]]
- [[wiki/concepts/agentic-rag.md|Agentic RAG]]
