---
title: "Learn Phase 5: Deep Dive — Settings Cascade, Hook Lifecycle, Source Code Map"
type: source
source_file: raw/notes/ai-context/LearnPhase5-Deep Dive — สิ่งที่ยังไม่ได้ครอบคลุม.md
url: ""
published: 2026-04-13
tags: [claude-code, settings, hooks, lifecycle, gotchas, clean-architecture]
related:
  - wiki/concepts/claude-code-ai-cli.md
  - wiki/concepts/ai-hooks-system.md
  - wiki/concepts/clean-architecture-go.md
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/ai-context/LearnPhase5-Deep Dive — สิ่งที่ยังไม่ได้ครอบคลุม.md|Original file]]

## สรุป

Phase 5 เจาะลึก Settings Cascade (3 ระดับ), Hook Lifecycle ครบทุก type, การ map AI config กับ source code จริง, Global Plugins (Context7, gopls-lsp), MCP Protocol deep dive, ข้อจำกัดและ gotchas, และวิธีขยายระบบ

## ประเด็นสำคัญ

**Settings Cascade (inner overrides outer):**
1. **Global** (`~/.claude/settings.json`) — ใช้ทุกโปรเจกต์: model=opus, 10+ hooks, plugins
2. **Project** (`.claude/settings.json`) — เฉพาะโปรเจกต์: env-protection hook (commit เข้า git)
3. **Local** (`.claude/settings.local.json`) — เฉพาะเครื่อง: permissions + MCP servers (ไม่ commit)

**Hooks merge = รวมกัน** (ไม่ทับกัน) — ทุก session ได้ hooks ทั้งหมดจากทุกระดับ

**Rules cascade:**
- Global (`~/.claude/rules/`): 8 ไฟล์ — agents, coding-style, git-workflow, hooks, patterns, performance, security, testing
- Project (`.claude/rules/`): 6 ไฟล์ — architecture, conventions, testing, codegen, project, outline-collections
- ทุก session ได้ **14 ไฟล์ rules** รวมกัน

**Hook Types ครบ:**
| Hook | ทำงานเมื่อ |
|---|---|
| SessionStart | เปิด session ใหม่ |
| PreToolUse | ก่อน tool ทำงาน |
| PostToolUse | หลัง tool ทำงาน |
| PreCompact | ก่อน context compact |
| Stop | Session จบ |
| SessionEnd | Session ปิดสมบูรณ์ |

## ข้อมูลที่น่าสนใจ

**Source Code → Config mapping ที่สำคัญ:**
- Entity struct จริง = ตรงกับ "Pure domain, ไม่พึ่ง framework ใดๆ" ใน config ทุกประการ
- UseCase fields เป็น interfaces ทั้งหมด → Dependency Inversion
- Error flow: entity error → use case throw → interface handler.ErrorHandler() → HTTP response พร้อม `issue_id` (trace ID จาก OTel!)

**Global Plugins:**
- **Context7** — ค้นหา library docs ล่าสุดสำหรับ AI
- **gopls-lsp** — Go Language Server → AI เข้าถึง diagnostics, type info

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

**Coverage threshold conflict:**
- Global rules: 80% minimum
- Project rules: 70% minimum
- Makefile: 70%
- **Project rules ชนะ** เพราะ CI ใช้ Makefile (70%)

## Gotchas สำคัญ

1. **Agent context isolation** — agent ไม่เห็น conversation history → ต้องใส่ context ครบใน Task prompt
2. **Skill ไม่มี tool restriction** — Skill/Command ใช้ tools ทั้งหมดของ main session
3. **Generated files ไม่มี hook บล็อก** — มีแค่ในเอกสาร ต้อง prompt ชัดเจน
4. **Global hooks JS-centric** — prettier/tsc hook ไม่ทำงานกับ .go files

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/claude-code-ai-cli|Claude Code AI CLI]]
- [[wiki/concepts/ai-hooks-system|AI Hooks System]]
- [[wiki/concepts/clean-architecture-go|Clean Architecture in Go]]
