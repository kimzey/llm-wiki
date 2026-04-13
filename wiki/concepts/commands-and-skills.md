---
title: "Commands and Skills — คำสั่งลัดและทักษะ AI"
type: concept
tags: [claude-code, commands, skills, slash-commands, ai-invocation]
sources:
  - ai-context-phase2.md
  - ai-context-phase4.md
  - ai-context-phase6.md
related:
  - wiki/concepts/claude-code-ai-cli.md
  - wiki/concepts/ai-agents-system.md
  - wiki/concepts/ai-dispatch-system.md
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

Commands และ Skills คือระบบ "recipe" สำเร็จรูปที่กำหนดขั้นตอนการทำงานไว้ล่วงหน้า ทั้งสองมี**เนื้อหาเหมือนกันทุกตัวอักษร** ต่างกันแค่ trigger: Command เรียกโดย user พิมพ์ `/xxx`, Skill เรียกโดย AI ที่ detect จาก description อัตโนมัติ

## อธิบาย

**Command** (`/.claude/commands/<name>.md`) — ไม่มี YAML front matter, เรียกด้วย `/command-name [arguments]`

**Skill** (`.claude/skills/<name>.md`) — มี YAML front matter ที่มี `description` field:
```yaml
---
description: Scaffold a complete new feature for the domain named $ARGUMENTS.
             Use when the user asks to add a new domain, entity, or feature.
---
```

**`$ARGUMENTS`** — ถูกแทนที่ด้วยสิ่งที่พิมพ์หลังชื่อคำสั่ง (เช่น `/new-feature payment` → `$ARGUMENTS = "payment"`)

**9 คู่ Command/Skill ในระบบ boilerplate:**

| ชื่อ | หน้าที่ |
|---|---|
| `new-feature` | Scaffold feature ใหม่ end-to-end (entity → HTTP handler) |
| `add-usecase` | เพิ่ม use case method ใน domain ที่มีอยู่แล้ว |
| `add-http-route` | Implement HTTP handler สำหรับ TypeSpec-generated route |
| `add-error` | เพิ่ม domain error sentinel + HTTP mapping |
| `gen-http` | Regenerate HTTP interface จาก TypeSpec |
| `gen-grpc` | Regenerate gRPC stubs จาก .proto |
| `unit-test` | รัน unit tests + รายงาน |
| `coverage` | เช็ค coverage threshold (>=70%) |
| `review-arch` | ตรวจ architecture compliance |

## ประเด็นสำคัญ

**ทำงานใน main session** — ต่างจาก Agent ที่ทำงานใน subprocess:
- เห็น conversation history
- ใช้ tools ทั้งหมดของ main session (รวมถึง Task tool → สามารถ spawn Agent ได้)
- ไม่มีการจำกัด tools

**Command vs Skill — ทำไมต้องมีทั้งคู่:**
- **Command** = entry point สำหรับคน → พิมพ์ `/new-feature widget` → แน่นอน 100%
- **Skill** = entry point สำหรับ AI → Claude ตัดสินใจเรียกเองจาก description → flexible

**ข้อจำกัดสำคัญ:**
- Command ทำงานใน main session → **ไม่ได้ Jira/Outline workflow** โดยอัตโนมัติ
- ถ้าต้องการ Agent workflow → ต้องระบุ agent ในคำสั่ง หรือเขียน command ที่ spawn agent ด้วย prompt

## ตัวอย่าง / กรณีศึกษา

**User พิมพ์ `/new-feature payment`:**
1. Hardcoded: โหลด `commands/new-feature.md`
2. แทนค่า `$ARGUMENTS = "payment"`
3. Inject prompt เข้า main session
4. Claude ทำตาม steps ทีละขั้นตอน (ไม่มี agent เข้ามาเกี่ยวข้อง)

**User พิมพ์ "เพิ่ม feature payment":**
1. Claude อ่าน skill descriptions ทุกตัว
2. match `new-feature.md` description: "Scaffold a complete new feature..."
3. เรียก Skill tool: `skill="new-feature", args="payment"`
4. ผลเหมือนกับ Command แต่ trigger ต่างกัน

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/ai-dispatch-system|AI Dispatch System]] — กลไกที่ Claude ตัดสินใจว่าจะเรียก Skill หรือ Command หรือ Agent
- [[wiki/concepts/ai-agents-system|AI Agents System]] — Skill สามารถ spawn Agent ได้ผ่าน Task tool, แต่ Agent ไม่สามารถเรียก Skill ได้

## แหล่งที่มา

- [[wiki/sources/ai-context-phase2|Phase 2: Agents, Commands, Skills]]
- [[wiki/sources/ai-context-phase4|Phase 4: Bindings]]
- [[wiki/sources/ai-context-phase6|Phase 6: Dispatch System]]
