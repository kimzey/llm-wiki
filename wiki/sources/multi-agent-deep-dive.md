---
title: "Multi-Agent Systems Deep Dive"
type: source
source_file: raw/notes/LangChain/multi-agent-deep-dive.md
tags: [langgraph, multi-agent, patterns, architecture]
related: []
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/LangChain/multi-agent-deep-dive.md|Original file]]

## สรุป

เอกสาร Deep Dive สำหรับ Multi-Agent Systems สอนวิธีสร้าง Agent หลายตัวทำงานร่วมกันผ่าน LangGraph ครอบคลุม Architecture Patterns ทั้งหมด ได้แก่ Supervisor, Handoff, Swarm, Pipeline, และ Debate

## ประเด็นสำคัญ

### ทำไมต้อง Multi-Agent

**Single Agent vs Multi-Agent**
```
Single Agent:
  ❌ System prompt ยาวมาก (หลาย role ในตัวเดียว)
  ❌ เปลี่ยน model ไม่ได้ (บาง task ต้องการ model ที่ต่างกัน)
  ❌ ยากต่อการ debug (ทุกอย่างรวมกันหมด)
  ❌ ไม่สามารถทำงานขนานได้

Multi-Agent:
  ✅ แต่ละ Agent มี role ชัดเจน prompt สั้นกระชับ
  ✅ ใช้ model ที่เหมาะสมกับแต่ละ task
  ✅ Debug ง่าย (แยก trace ตาม Agent)
  ✅ ทำงานขนานได้ (Parallel execution)
  ✅ เพิ่ม/ลด Agent ได้ง่าย (Modular)
```

**เมื่อไหร่ควรใช้ Multi-Agent**
```
✅ ใช้เมื่อ:
  - งานมีหลายขั้นตอนที่ต้องใช้ความเชี่ยวชาญต่างกัน
  - ต้องการ quality control (Agent ตรวจ Agent)
  - ต้องการ parallel processing
  - ระบบซับซ้อนเกินกว่า single agent จะจัดการได้

❌ ไม่จำเป็นเมื่อ:
  - งาน simple ที่ Agent เดียวทำได้
  - ไม่ต้องการ role แยกชัดเจน
  - ต้นทุน/latency เป็นปัจจัยหลัก
```

### Architecture Patterns

```
1. Supervisor     2. Handoff      3. Swarm
  ┌───┐            A ──→ B         A ↔ B ↔ C
  │ S │            ↓     ↓           ↕   ↕
  └┬┬┬┘            C ──→ D         D ↔ E ↔ F
   ││└→ C
   │└→ B
   └→ A

4. Pipeline       5. Debate
  A → B → C → D    A ←→ B
  (สายพาน)         Judge ↓
                   (ตัดสิน)
```

### 5 Patterns หลัก

**1. Supervisor Pattern**
- มี "Supervisor" Agent เป็นหัวหน้า
- คอยตัดสินใจว่าจะสั่งให้ Agent ไหนทำงาน
- Flow: `User → Supervisor → "ให้ Researcher ไปค้นข้อมูล" → "ให้ Writer เขียนบทความ" → "ให้ Reviewer ตรวจสอบ" → "เสร็จแล้ว ส่งผลลัพธ์"`

**2. Handoff Pattern**
- User ถามมา → Triage Agent รับเรื่อง → ส่งต่อให้ Agent ที่เชี่ยวชาญ
- เหมือนโทรไป Call Center แล้วถูกโอนสาย

**3. Swarm Pattern**
- ทุก Agent ทำงานขนานกัน พร้อมกัน
- มีการสื่อสารระหว่าง Agent

**4. Pipeline Pattern**
- A → B → C → D (ทำงานต่อเนื่องเป็นขั้นตอน)
- ทุก Node เชื่อมต่อกันเป็นเส้นตรง

**5. Debate Pattern**
- Agent ถกเถียงกัน → Judge ตัดสิน
- ใช้สำหรับเลือก solution ที่ดีที่สุด

### หัวข้อที่ครอบคลุม

1. **ทำไมต้อง Multi-Agent**
2. **Architecture Patterns**
3. **Pattern 1: Supervisor (มีหัวหน้า)**
4. **Pattern 2: Handoff (ส่งต่อ)**
5. **Pattern 3: Swarm (ฝูง)**
6. **Pattern 4: Pipeline (สายพาน)**
7. **Pattern 5: Debate (ถกเถียง)**
8. **Shared State & Communication**
9. **ตัวอย่าง: Content Creation Team**
10. **ตัวอย่าง: Software Development Team**
11. **ตัวอย่าง: Customer Support Escalation**
12. **Error Handling & Safeguards**
13. **Best Practices**

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- **Modular design** = เพิ่ม/ลด Agent ได้ง่าย
- **Parallel execution** = ประหยัดเวลา
- **Quality control** = Agent ตรวจ Agent ได้

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อมูลที่ขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/ai-agent.md|AI Agent]]
- [[wiki/concepts/agentic-rag.md|Agentic RAG]]
