---
title: "Learn Phase 6: Dispatch — ใครเลือก? ใครตัดสินใจ? ระบบ Dispatch ที่แท้จริง"
type: source
source_file: raw/notes/ai-context/LearnPhase6-Dispatch ใครเลือก ใครตัดสินใจ — ระบบ Dispatch ที่แท้จริง.md
url: ""
published: 2026-04-13
tags: [claude-code, dispatch, decision-making, ai-behavior, non-deterministic]
related:
  - wiki/concepts/ai-dispatch-system.md
  - wiki/concepts/ai-agents-system.md
  - wiki/concepts/commands-and-skills.md
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/ai-context/LearnPhase6-Dispatch ใครเลือก ใครตัดสินใจ — ระบบ Dispatch ที่แท้จริง.md|Original file]]

## สรุป

Phase 6 ตอบคำถาม "ใครเลือก?" เมื่อสั่งงาน — ไม่มี routing table ใดๆ Claude ใช้ "ความเข้าใจภาษา" อ่าน description แล้วเลือกเอง มีเพียงบางอย่างที่ hardcoded (ชื่อไฟล์ match) ที่เหลือเป็น AI reasoning

## ประเด็นสำคัญ

**สิ่งที่ hardcoded (แน่นอน 100%):**
- `/new-feature` → โหลด `commands/new-feature.md` (ชื่อไฟล์ตรงกัน)
- `Skill("new-feature")` → โหลด `skills/new-feature.md` (ชื่อไฟล์ตรงกัน)
- `Task(subagent_type="Developer")` → โหลด `agents/developer.md` (YAML name ตรงกัน)

**สิ่งที่ AI เลือกเอง (ไม่แน่นอน):**
- จะใช้ Skill ไหม? → อ่าน description เทียบกับ request
- จะ spawn Agent ไหม? → ประเมินความซับซ้อน + description ของ agent
- จะทำเองใน main session? → ถ้างานง่ายพอ

**Decision tree:**
1. ถ้าเริ่มด้วย `/` → **Hardcoded** โหลด command .md ทำใน main session
2. ถ้าไม่ → อ่าน skill descriptions → ถ้า match → เรียก Skill
3. ถ้า skill ไม่ match → ประเมินความซับซ้อน → ถ้าซับซ้อน spawn Agent
4. ถ้าไม่ซับซ้อน → ทำเองใน main session

## ข้อมูลที่น่าสนใจ

**ระดับความมั่นใจ:**
| วิธีสั่ง | ขั้นตอนถูก? | Agent ถูก? | ความมั่นใจ |
|---|---|---|---|
| `/new-feature widget` | 100% | ไม่ใช้ agent | สูง |
| "เพิ่ม feature widget" | ~90% | ไม่ใช้ agent | ปานกลาง |
| "ให้ Dev agent ทำ /new-feature" | ~95% | 100% | สูง |
| `/dev-new-feature widget` (command ที่ spawn agent) | 100% | 100% | สูงมาก |
| ปล่อยให้ AI เดา | ~70% | ~60% | ต่ำ |

**ปัญหาที่เกิดจาก "ไม่ผูก":**
1. Command ไม่รับประกันว่าจะใช้ Agent (จะไม่ได้ Jira/Outline workflow)
2. AI อาจเลือก Skill ผิด
3. Agent ทำตาม checklist ของตัวเอง ≠ Skill steps

**วิธีแก้:** เขียน Command ที่สั่ง spawn Agent ด้วย prompt ใน body = ผูก Command + Agent โดย convention

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ข้อมูลนี้อธิบายว่า `/new-feature` **ไม่ได้** ใช้ Developer agent โดยอัตโนมัติ — ตรงข้ามกับที่หลายคนอาจคิด (command ไม่ผูกกับ agent)

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/ai-dispatch-system|AI Dispatch System]]
- [[wiki/concepts/ai-agents-system|AI Agents System]]
- [[wiki/concepts/commands-and-skills|Commands and Skills]]
