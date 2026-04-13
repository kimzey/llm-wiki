---
title: "Learn Phase 1: ภาพรวมโครงสร้าง .claude และ .opencode"
type: source
source_file: raw/notes/ai-context/Learn Phase 1 ภาพรวมโครงสร้าง .claude และ .opencode.md
url: ""
published: 2026-04-13
tags: [claude-code, ai-cli, settings, hooks, rules, architecture]
related:
  - wiki/concepts/claude-code-ai-cli.md
  - wiki/concepts/ai-hooks-system.md
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/ai-context/Learn Phase 1 ภาพรวมโครงสร้าง .claude และ .opencode.md|Original file]]

## สรุป

เอกสาร Phase 1 อธิบายโครงสร้างของโฟลเดอร์ `.claude/` และ `.opencode/` ที่ใช้ตั้งค่าให้ AI (Claude Code / OpenCode) เข้าใจโปรเจกต์โดยอัตโนมัติ ครอบคลุม CLAUDE.md (system prompt), rules/, settings.json, settings.local.json, และ hooks/

## ประเด็นสำคัญ

- **`.claude/`** คือ config สำหรับ Claude Code CLI — โหลดเข้า session ทุกครั้งเป็น system prompt
- **`.opencode/`** คือ equivalent สำหรับ OpenCode CLI — รวมทุกอย่างใน AGENTS.md ไฟล์เดียว
- **CLAUDE.md** คือหัวใจของระบบ — บอก AI architecture, key files, make targets, slash commands
- **rules/** ถูก auto-load ทุก session: architecture.md, conventions.md, testing.md, codegen.md, project.md, outline-collections.md
- **settings.json** ตั้งค่า hooks (ระดับ project, commit เข้า git ร่วมกันทีม)
- **settings.local.json** ตั้งค่า permissions (คำสั่ง bash ที่ auto-allow) + MCP servers (ไม่ commit)
- **hooks/** คือ script ที่รันอัตโนมัติก่อน/หลัง tool call — ใช้ป้องกัน AI อ่าน .env

## ข้อมูลที่น่าสนใจ

**Clean Architecture 4 Layer (Unidirectional Dependencies):**
```
Interface → UseCase → Entity
Repository ─────────→ Entity
```

**Hook protocol** — Claude Code ส่ง JSON tool input เข้า stdin → script ตัดสินใจ:
- exit 0 = อนุญาต
- exit 1 = warning ไม่บล็อก
- exit 2 = **บล็อก tool call**

**env-protection.sh** ทำงานก่อนทุก Read tool call → ถ้า filename เป็น `.env` หรือ `.env.*` → exit 2

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/claude-code-ai-cli|Claude Code AI CLI]]
- [[wiki/concepts/ai-hooks-system|AI Hooks System]]
- [[wiki/concepts/commands-and-skills|Commands and Skills]]
