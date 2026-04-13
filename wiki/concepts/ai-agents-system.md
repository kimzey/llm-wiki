---
title: "AI Agents System — บทบาทและการทำงานของ Agents"
type: concept
tags: [claude-code, agents, subprocess, roles, tools, context-isolation]
sources:
  - ai-context-phase2.md
  - ai-context-phase4.md
  - ai-context-phase5.md
  - ai-context-phase7.md
related:
  - wiki/concepts/claude-code-ai-cli.md
  - wiki/concepts/commands-and-skills.md
  - wiki/concepts/ai-dispatch-system.md
  - wiki/concepts/mcp-model-context-protocol.md
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

Agent คือ "บุคลิก" ที่กำหนดให้ AI ทำงานเฉพาะบทบาท แต่ละ agent ทำงานใน **subprocess แยก** มี tools จำกัด และ identity ชัดเจน ถูกเรียกผ่าน Task tool โดย Claude (main session) เมื่องานต้องการ role เฉพาะ

## อธิบาย

Agent file ประกอบด้วย 2 ส่วน:
1. **YAML Front Matter** — metadata: name, description, tools list, model (optional)
2. **Markdown Body** — system prompt ของ agent: identity, workflow, checklist

**Project Agents (4 ตัว)** — อยู่ใน `.claude/agents/`:

| Agent | บทบาท | Tools | ข้อจำกัดพิเศษ |
|---|---|---|---|
| **Developer** | Senior Go Developer | Read, Edit, Write, Bash, Glob, Grep, mcp__jira, mcp__outline | — |
| **PO** | Product Owner | Read, Glob, mcp__jira, mcp__outline | ไม่มี Edit/Write/Bash (ห้ามแก้โค้ด) |
| **QA** | QA Engineer | Read, Glob, Grep, Bash, mcp__jira, mcp__outline | ไม่มี Edit/Write |
| **Solution Architect** | ดูแล technical vision | Read, Glob, Grep, Bash, mcp__jira, mcp__outline | ไม่มี Edit/Write |

**Global Agents (10 ตัว)** — อยู่ใน `~/.claude/agents/` ใช้ได้ทุกโปรเจกต์ ไม่มี MCP tools:
planner, architect, code-reviewer, tdd-guide, security-reviewer, build-error-resolver, database-reviewer, e2e-runner, refactor-cleaner, doc-updater

## ประเด็นสำคัญ

**Context Isolation (ข้อจำกัดสำคัญ):**
- Agent ทำงานใน subprocess แยก
- ได้รับ: CLAUDE.md + rules/ + agent's own file + Task prompt
- **ไม่เห็น**: conversation history, previous tool results, ไฟล์ agent ตัวอื่น
- → ต้องใส่ context ครบใน Task prompt เสมอ

**Tools Binding:**
- tools list ใน YAML **บังคับ** — agent ใช้ tools อื่นนอก list ไม่ได้
- `mcp__jira` ใน tools list = สามารถใช้ **ทุก tool** ใน Jira MCP server
- Agent ที่มี `mcp__jira` แต่ server ไม่เปิดใน `settings.local.json` → ใช้ไม่ได้

**Agent ≠ Command/Skill:**
- Agent ไม่รู้จัก Skills หรือ Commands โดยตรง
- Agent มี checklist workflow ของตัวเองใน markdown body
- เมื่อ spawn agent → agent ทำตาม checklist ของตัวเอง ไม่ใช่ตาม skill steps

## ตัวอย่าง / กรณีศึกษา

**เรียก Developer agent:**
```
ให้ Developer agent scaffold domain payment ตั้งแต่ entity จนถึง HTTP handler
แล้วอัพเดท Jira ticket LR-15 เป็น In Progress
```
→ Claude เรียก Task tool: `subagent_type="Developer"` + prompt
→ Developer subprocess ได้รับ CLAUDE.md + rules/ + developer.md + prompt
→ ทำงาน 9 steps ตาม implementation checklist ใน developer.md

**PO agent สร้าง Jira tickets:**
```
ให้ PO agent สร้าง Epic [PAY] Payment Processing ใน Jira project LR
พร้อม Tech Stories 5 ข้อ
```
→ PO agent ไม่มี Edit/Write → ไม่สามารถแก้โค้ดได้
→ ใช้ mcp__jira สร้าง Epic + Tech Stories ตาม naming convention

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/ai-dispatch-system.md|AI Dispatch System]] — Claude ตัดสินใจว่าจะ spawn agent หรือทำเอง
- [[wiki/concepts/commands-and-skills.md|Commands and Skills]] — Command/Skill ทำงานใน main session; Agent ทำงานใน subprocess
- [[wiki/concepts/mcp-model-context-protocol.md|MCP]] — Agent ใช้ MCP tools ผ่าน tools list ใน YAML

## แหล่งที่มา

- [[wiki/sources/ai-context-phase2.md|Phase 2: Agents, Commands, Skills]]
- [[wiki/sources/ai-context-phase4.md|Phase 4: Bindings]]
- [[wiki/sources/ai-context-phase5.md|Phase 5: Deep Dive]]
- [[wiki/sources/ai-context-phase7.md|Phase 7: Global Agents + Practical Manual]]
