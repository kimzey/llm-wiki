---
title: "Learn Phase 2: Agents, Commands, และ Skills — ระบบ AI Workflow อัตโนมัติ"
type: source
source_file: raw/notes/ai-context/LearnPhase2  Agents, Commands, และ Skills — ระบบ AI Workflow อัตโนมัติ.md
url: ""
published: 2026-04-13
tags: [claude-code, agents, commands, skills, jira, mcp]
related:
  - wiki/concepts/ai-agents-system.md
  - wiki/concepts/commands-and-skills.md
  - wiki/concepts/claude-code-ai-cli.md
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/ai-context/LearnPhase2  Agents, Commands, และ Skills — ระบบ AI Workflow อัตโนมัติ.md|Original file]]

## สรุป

Phase 2 อธิบายระบบ 3 อย่างหลักใน Claude Code: Agents (บทบาท AI), Commands (คำสั่งลัดที่ user เรียก), และ Skills (ทักษะที่ AI เลือกเองได้) พร้อม Jira/Outline workflow ของแต่ละ agent

## ประเด็นสำคัญ

**4 Project Agents:**
- **Developer** — นักพัฒนา Go ครบเครื่อง (Read, Edit, Write, Bash, mcp__jira, mcp__outline)
- **Product Owner** — จัดการ backlog, Jira tickets — ไม่มี Edit/Write/Bash (ห้ามแก้โค้ด)
- **QA Engineer** — เทสต์, coverage — ไม่มี Edit/Write (อ่านและรันเทสต์ได้ ไม่แก้ code)
- **Solution Architect** — review, ADR — เหมือน QA แต่มุ่ง architecture

**9 Commands (user-invoked):**
`/new-feature`, `/add-usecase`, `/add-http-route`, `/add-error`, `/gen-http`, `/gen-grpc`, `/unit-test`, `/coverage`, `/review-arch`

**Skills vs Commands:**
- เนื้อหา **เหมือนกัน** ทุกตัวอักษร (9 pairs)
- Skill มี YAML description → Claude **auto-detect** ได้
- Command ต้องผู้ใช้พิมพ์ `/xxx` เรียกตรงๆ

## ข้อมูลที่น่าสนใจ

**PO Agent issue hierarchy** — 3 ระดับ เคร่งครัด:
- Level 1: Epic
- Level 0: Story / Tech Story / Bug / Tech Enhancement
- Level -1: Dev Task / QA Task
- **ห้าม** ใช้ "Task" หรือ "Sub-task" เด็ดขาด

**Developer implementation checklist** (9 steps ตามลำดับ):
entity → repository interface → mock → GORM impl → use case → wire use_case.go → DI → HTTP handler → make unit-test

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/ai-agents-system|AI Agents System]]
- [[wiki/concepts/commands-and-skills|Commands and Skills]]
- [[wiki/concepts/mcp-model-context-protocol|MCP Model Context Protocol]]
