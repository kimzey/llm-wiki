---
title: "Step 6: Multi-Agent Systems"
type: source
source_file: raw/notes/LangChain/step6-multi-agent-tutorial.md
tags: [langchain, langgraph, multi-agent, tutorial, patterns]
related: []
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/LangChain/step6-multi-agent-tutorial.md|Original file]]

## สรุป

บทช่วยสอน Multi-Agent Systems สอนวิธีสร้าง "ทีม AI" ที่แต่ละตัวมีความเชี่ยวชาญเฉพาะทาง เนื้อหาครอบคลุม 3 patterns หลัก: Supervisor (มีหัวหน้าสั่งงาน), Handoff (ส่งต่อตามความเชี่ยวชาญ), และ Pipeline (สายพานผลิต)

## ประเด็นสำคัญ

### ทำไมต้องหลาย Agent?
- **Single Agent**:
  - System prompt: "คุณเป็นทั้ง HR expert, IT support, นักเขียน, นักวิจัย, บรรณาธิการ..."
  - ❌ prompt ยาวมาก → สับสน
  - ❌ ยากต่อการ debug
  - ❌ ทำงานทีละอย่าง

- **Multi-Agent**:
  - Agent 1: "คุณเป็น HR expert" ← prompt สั้น ชัดเจน
  - Agent 2: "คุณเป็น IT support" ← แยก role
  - Agent 3: "คุณเป็นหัวหน้าทีม" ← ตัดสินใจว่าส่งใคร
  - ✅ แต่ละตัวเก่งในสิ่งที่ถนัด
  - ✅ debug ง่าย
  - ✅ ทำงานขนานได้

### Pattern 1: Supervisor (มีหัวหน้าสั่งงาน)
- **Supervisor**: หัวหน้าที่ดู state แล้วตัดสินใจว่าส่งให้ใคร
- **Workers**: สมาชิกแต่ละคนทำงานเฉพาะทาง (researcher, writer, reviewer)
- **Flow**: START → supervisor → (ตัดสินใจ) → worker → กลับไป supervisor → วน loop → FINISH

- **ใช้ Structured Output** เพื่อให้ routing แม่นยำ:
  ```python
  class Decision(BaseModel):
      next: str = Field(description="ส่งให้ใคร: researcher, writer, reviewer, FINISH")
      reason: str = Field(description="เหตุผลสั้นๆ")

  structured = boss_model.with_structured_output(Decision)
  ```

### Pattern 2: Handoff (ส่งต่อตามความเชี่ยวชาญ)
- **แนวคิด**: User ถามมา → Triage Agent รับเรื่อง → ส่งต่อให้ Agent ที่เชี่ยวชาญ
- เหมือนโทรไป Call Center แล้วถูกโอนสาย
- **Triage**: รับเรื่อง แล้วจำแนกว่าส่งแผนกไหน (hr, it, general)
- **Handoff agents**: แต่ละแผนกมี Agent เฉพาะทาง

### Pattern 3: Pipeline (สายพานผลิต)
- **แนวคิด**: A → B → C → D (ทำงานต่อเนื่องเป็นขั้นตอน)
- ทุก Node เชื่อมต่อกันเป็นเส้นตรง
- **ตัวอย่าง**: translator → summarizer → formatter

### การใช้ Structured Output สำหรับ Routing
- `with_structured_output(Decision)`: ใช้ structured output เพื่อให้ได้ JSON ที่ parse ได้
- ไม่ใช่ free text ที่อาจ parse ผิด
- ช่วยให้ routing แม่นยำขึ้น

### Flow ของ Supervisor Pattern
```
Supervisor → researcher (ยังไม่มีข้อมูล ต้องค้นหาก่อน)
Researcher → (รวบรวมข้อมูล)
Supervisor → writer (มีข้อมูลแล้ว ให้เขียน)
Writer → (เรียบเรียงเป็นบทความ)
Supervisor → reviewer (เขียนแล้ว ต้องตรวจ)
Reviewer → APPROVED
Supervisor → FINISH (reviewer อนุมัติแล้ว)
```

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- Structured Output ใช้ Base model (`boss_model`) สำหรับตัดสินใจ
- Worker model ใช้ temperature สูงกว่า (0.3 vs 0) เพื่อความสร้างสรรค์
- `Literal["hr_agent", "it_agent", "general_agent"]`: บอกว่า return ได้แค่ค่าเหล่านี้
- Pipeline pattern ง่ายที่สุด → เป็นเส้นตรง ไม่มี conditional edges

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อมูลที่ขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/ai-agent.md|AI Agent]]
- [[wiki/concepts/agentic-rag.md|Agentic RAG]]
