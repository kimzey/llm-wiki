---
title: "AI Dispatch System — ระบบตัดสินใจของ Claude"
type: concept
tags: [claude-code, dispatch, decision-making, non-deterministic, ai-behavior]
sources:
  - ai-context-phase4.md
  - ai-context-phase6.md
related:
  - wiki/concepts/ai-agents-system.md
  - wiki/concepts/commands-and-skills.md
  - wiki/concepts/claude-code-ai-cli.md
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

ไม่มี routing table หรือ mapping ใดๆ ระหว่าง Command, Skill, และ Agent — Claude ใช้ "ความเข้าใจภาษา" อ่าน description แล้วตัดสินใจเอง มีเพียงบางส่วนที่ hardcoded (ชื่อไฟล์ match) ที่เหลือเป็น AI reasoning ที่ไม่ deterministic

## อธิบาย

**Decision Tree:**
```
User ส่งข้อความ
├── เริ่มด้วย / ?
│   └── ใช่ → HARDCODED: โหลด commands/<name>.md → ทำใน main session
│
└── ไม่ → AI Decision Layer:
    ├── 1) อ่าน skill descriptions ทั้งหมด
    │      ├── match? → เรียก Skill tool → ทำใน main session
    │      └── ไม่ match → ข้อ 2
    ├── 2) งานซับซ้อน? ต้องการ specific role?
    │      ├── ใช่ → อ่าน agent descriptions → Task(subagent_type=...)
    │      └── ไม่ → ข้อ 3
    └── 3) ทำเองใน main session ด้วย tools โดยตรง
```

**สิ่งที่ hardcoded (แน่นอน 100%):**

| Trigger | กลไก |
|---|---|
| `/new-feature` | ชื่อไฟล์ตรงกับ command name |
| `Skill("new-feature")` | ชื่อไฟล์ตรงกับ skill name |
| `Task(subagent_type="Developer")` | YAML `name:` field ตรงกัน |
| `$ARGUMENTS` | Template substitution ของ Claude Code |
| Agent tools list | YAML `tools:` บังคับ ไม่สามารถใช้ tools อื่นได้ |

## ประเด็นสำคัญ

**ระดับความมั่นใจ:**
| วิธีสั่ง | ขั้นตอน | Agent | ความมั่นใจ |
|---|---|---|---|
| `/new-feature widget` | 100% | ไม่ใช้ | สูง |
| "เพิ่ม feature widget" | ~90% | ไม่ใช้ | ปานกลาง |
| "ให้ Dev agent ทำ /new-feature" | ~95% | 100% | สูง |
| `/dev-new-feature widget` (command spawn agent) | 100% | 100% | สูงมาก |

**ข้อดีของระบบ "ไม่ผูก":**
- ยืดหยุ่น — เพิ่ม agent/skill ใหม่ได้ไม่ต้องแก้ mapping
- Natural language — สั่งภาษาธรรมชาติได้
- Graceful degradation — ถ้าเลือกผิดก็ยังทำงานได้ (แค่ไม่ optimal)

**ข้อเสียของระบบ "ไม่ผูก":**
- ไม่ deterministic — สั่งเหมือนกัน 2 ครั้งอาจได้ path ต่างกัน
- Command ไม่ได้ Agent workflow — `/new-feature` ไม่ได้ Jira/Outline โดยอัตโนมัติ
- ต้องเขียน description ดี — ถ้า description กำกวม AI เลือกผิด

## ตัวอย่าง / กรณีศึกษา

**วิธีรับประกันว่าจะได้ Agent:**
```markdown
<!-- .claude/commands/dev-new-feature.md -->
Scaffold a complete new feature using the Developer agent.

Domain: $ARGUMENTS

Use the Task tool to spawn a Developer agent:
"Scaffold payment domain with entity, repository, use case, HTTP handler.
After done: Update Jira ticket to In Progress."

Use subagent_type: "Developer"
```
→ `/dev-new-feature payment` **รับประกัน** ว่าจะ spawn Developer agent

**Best practice เลือกวิธีสั่ง:**
- ต้องการ "แค่โค้ด" → `/new-feature payment`
- ต้องการ "โค้ด + Jira + Outline" → "ให้ Developer agent scaffold..."
- ต้องการ deterministic + agent → เขียน command ที่ spawn agent

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/ai-agents-system|AI Agents System]] — Agent ถูกเลือกผ่าน dispatch
- [[wiki/concepts/commands-and-skills|Commands and Skills]] — Command/Skill เป็น entry points ที่มีระดับ determinism ต่างกัน

## แหล่งที่มา

- [[wiki/sources/ai-context/ai-context-phase4|Phase 4: Bindings]]
- [[wiki/sources/ai-context/ai-context-phase6|Phase 6: Dispatch System]]
