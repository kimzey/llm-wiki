---
title: "Step 2: Tools & Agents"
type: source
source_file: raw/notes/LangChain/step2-tools-agents.md
tags: [langchain, tools, agents, tutorial]
related: []
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/langchain/step2-tools-agents.md|Original file]]

## สรุป

บทช่วยสอนเรื่อง Tools และ Agents ใน LangChain สอนวิธีสร้างเครื่องมือให้ AI ใช้และสร้าง Agent ที่สามารถคิดและตัดสินใจเองได้ เนื้อหาครอบคลุมการสร้าง Tool ด้วย `@tool` decorator, การ bind tools เข้ากับ model, ReAct Loop, และการสร้าง Agent ด้วย `create_agent`

## ประเด็นสำคัญ

### ทำไม AI ต้องมี Tools?
- ไม่มี Tools: AI ตอบได้แค่จากความรู้ที่มี (knowledge cutoff)
- มี Tools: AI สามารถ ค้นข้อมูล, คำนวณ, เรียก API, ลงมือทำได้จริง

### การสร้าง Tool ด้วย @tool decorator
```python
@tool
def calculate(expression: str) -> str:
    """คำนวณนิพจน์ทางคณิตศาสตร์"""
    return f"{expression} = {eval(expression)}"
```
- **Type hints สำคัญมาก**: AI รู้ว่าต้องใส่อะไร
- **Docstring สำคัญมาก**: AI อ่านเพื่อตัดสินใจว่าจะใช้ tool นี้เมื่อไหรี
- Tool ที่ดีต้องมีทั้ง type hints และ docstring

### Tool Calling Flow
1. **Reasoning**: AI คิดว่าต้องใช้ tool ไหน
2. **Acting**: AI ตอบด้วย `tool_calls` (ยังไม่ตอบจริง)
3. **Execution**: เรารัน tool ด้วยตัวเอง หรือให้ Agent รัน
4. **Observation**: ได้ผลลัพธ์จาก tool
5. **Repeating**: AI ดูผล → คิดต่อ → เรียก tool อีก หรือตอบ

### bind_tools()
- `model.bind_tools(tools)` = "สอน" ให้ model รู้ว่ามี tools อะไรบ้าง
- Model จะอ่าน name + description + parameters ของแต่ละ tool
- แล้วตัดสินใจเองว่าจะเรียก tool ไหน

### Agent และ ReAct Loop
- **Agent** = ระบบที่:
  1. รับคำถาม
  2. คิดว่าต้องใช้ tool ไหน (Reasoning)
  3. เรียก tool อัตโนมัติ (Acting)
  4. ดูผล → คิดต่อ → เรียก tool อื่น (ถ้าจำเป็น)
  5. สรุปคำตอบสุดท้าย

- **ReAct** = **Re**asoning + **Act**ing (วน loop)
- ใช้ `create_agent(model, tools, system_prompt)`

### สร้าง Chatbot ที่จำบทสนทนาได้
- เก็บประวัติใน list: `history = []`
- ทุกครั้งที่คุย: เพิ่ม user message → ส่งทั้ง history → เพิ่ม AI response
- AI จำได้เพราะเราส่ง history ไปทุกครั้ง

### Temperature สำหรับ Agent
- **ใช้ `temperature=0`** สำหรับ Agent!
- ลดความ "สุ่ม" ให้ตัดสินใจแม่นยำ

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- **Manual Execution**: ต้องเขียน code รัน tool เอง → ยุ่งมาก ถ้า AI เรียก tool หลายตัวต่อเนื่อง
- **Agent**: จัดการให้หมดอัตโนมัติ → AI ตัดสินใจเรียก tool เอง → รัน → ดูผล → วน loop
- **Tool map pattern**: `tool_map = {t.name: t for t in tools}` ใช้หา tool จากชื่อ
- **tool_calls**: เป็น list ที่มี `name`, `args`, `id`

### Example: Multi-step Reasoning
User: "ค่าอาหาร 22 วัน รวมค่าเดินทาง เดือนนึงเท่าไหร่?"
1. Reasoning: "ต้องรู้ค่าอาหารก่อน" → Call `search_company_info("ค่าอาหาร")`
2. Result: "100 บาท/วัน"
3. Reasoning: "ต้องรู้ค่าเดินทางด้วย" → Call `search_company_info("ค่าเดินทาง")`
4. Result: "2,500 บาท/เดือน"
5. Reasoning: "ต้องคำนวณ 100*22 + 2500" → Call `calculate("100 * 22 + 2500")`
6. Result: "4700"
7. Reasoning: "ได้ข้อมูลครบแล้ว สรุปคำตอบ" → ไม่เรียก tool → ตอบ user

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อมูลที่ขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/ai-agent|AI Agent]]
- [[wiki/concepts/agentic-rag|Agentic RAG]]
