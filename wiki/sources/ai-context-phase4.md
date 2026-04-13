---
title: "Learn Phase 4: Bindings — Agent, Skill, Command ผูกกันยังไง"
type: source
source_file: raw/notes/ai-context/LearnPhase4 Bindings Agent, Skill, Command ผูกกันยังไง.md
url: ""
published: 2026-04-13
tags: [claude-code, agents, skills, commands, bindings, architecture]
related:
  - wiki/concepts/ai-agents-system.md
  - wiki/concepts/commands-and-skills.md
  - wiki/concepts/ai-dispatch-system.md
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/ai-context/LearnPhase4 Bindings Agent, Skill, Command ผูกกันยังไง.md|Original file]]

## สรุป

Phase 4 อธิบายความสัมพันธ์ระหว่าง Agent, Skill, และ Command อย่างละเอียด — ทั้ง 3 ระบบเป็นอิสระต่อกัน ไม่ผูกกันโดยตรง แต่ทำงานร่วมกันผ่าน shared context (CLAUDE.md + rules/) โดยมี Claude เป็นตัวกลาง

## ประเด็นสำคัญ

**3 ระบบอิสระ:**
| คุณสมบัติ | Agent | Skill | Command |
|---|---|---|---|
| ที่อยู่ | `.claude/agents/` | `.claude/skills/` | `.claude/commands/` |
| เรียกโดย | Claude (Task tool) หรือ user ระบุ | Claude (auto-detect จาก description) | User พิมพ์ `/xxx` |
| มี YAML front matter | ใช่ (name, description, tools) | ใช่ (description เท่านั้น) | ไม่มี |
| จำกัด tools | ได้ (tools: [...]) | ไม่ได้ (ใช้ main session tools ทั้งหมด) | ไม่ได้ |
| ทำงานแบบ | subprocess (context แยก) | main session | main session |

**Skill ↔ Command: คู่แฝด 1:1** — เนื้อหาเหมือนกันทุกตัวอักษร ต่างกันแค่ trigger

**Context Inheritance:**
- Skill/Command → ทำงานใน main session → เห็น CLAUDE.md + rules/ + conversation history
- Agent → ทำงานใน subprocess → เห็น CLAUDE.md + rules/ + agent's own file แต่ **ไม่เห็น** conversation history

## ข้อมูลที่น่าสนใจ

**Tool restrictions by agent:**
| Agent | Read | Edit | Write | Bash | mcp__jira | mcp__outline |
|---|---|---|---|---|---|---|
| Developer | Y | Y | Y | Y | Y | Y |
| PO | Y | - | - | - | Y | Y |
| QA | Y | - | - | Y | Y | Y |
| Architect | Y | - | - | Y | Y | Y |

**Skill สามารถเรียก Agent ได้** (เพราะอยู่ใน main session มี Task tool)
แต่ **Agent ไม่สามารถเรียก Skill ได้** (ไม่มี Skill tool ใน agent)

**Hook ทำงานกับทุก context** — ไม่ว่าจะเรียกจาก agent, skill, หรือ command = global security layer

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่มี — เป็นการขยายความจาก Phase 1-3

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/ai-agents-system|AI Agents System]]
- [[wiki/concepts/commands-and-skills|Commands and Skills]]
- [[wiki/concepts/ai-dispatch-system|AI Dispatch System]]
