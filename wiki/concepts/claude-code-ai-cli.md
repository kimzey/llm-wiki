---
title: "Claude Code / AI Coding CLI"
type: concept
tags: [claude-code, ai-cli, system-prompt, settings, rules, opencode]
sources:
  - ai-context-phase1.md
  - ai-context-phase3.md
  - ai-context-phase5.md
  - ai-context-phase7.md
related:
  - wiki/concepts/ai-agents-system.md
  - wiki/concepts/ai-hooks-system.md
  - wiki/concepts/commands-and-skills.md
  - wiki/concepts/mcp-model-context-protocol.md
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

Claude Code คือ AI coding CLI tool ของ Anthropic ที่โหลด context โปรเจกต์โดยอัตโนมัติทุก session ผ่านไฟล์ `CLAUDE.md` และ `rules/` ทำให้ AI "รู้" สถาปัตยกรรม, conventions, และคำสั่งทั้งหมดของโปรเจกต์โดยไม่ต้องอธิบายซ้ำ

## อธิบาย

Claude Code ทำงานในรูปแบบ CLI ที่อ่านไฟล์ config ของโปรเจกต์เพื่อสร้าง "สมองเริ่มต้น" ของ AI สิ่งสำคัญคือการแยกความรู้ออกเป็นสองส่วน:

1. **Project-level knowledge** — เก็บใน `.claude/` ของโปรเจกต์ (commit เข้า git ร่วมกันทีม)
2. **Machine-level config** — เก็บใน `~/.claude/` (ส่วนตัว, ไม่ commit)

**ไฟล์และโฟลเดอร์หลัก:**

| ไฟล์/โฟลเดอร์ | หน้าที่ | ระดับ |
|---|---|---|
| `CLAUDE.md` | System prompt หลัก — โหลดทุก session | Project |
| `rules/` | กฎสถาปัตยกรรม, conventions, testing (auto-load) | Project |
| `agents/` | Agent definitions (บทบาท + tools list) | Project |
| `commands/` | User-invoked procedures (`/xxx`) | Project |
| `skills/` | AI-detected procedures | Project |
| `hooks/` | Automation scripts (PreToolUse, PostToolUse ฯลฯ) | Project |
| `settings.json` | Hook configuration (commit ร่วมทีม) | Project |
| `settings.local.json` | Permissions + MCP servers (ไม่ commit) | Machine |
| `~/.claude/settings.json` | Global hooks, model, plugins | Global |
| `~/.claude/rules/` | Global rules (ใช้ทุกโปรเจกต์) | Global |
| `~/.claude/agents/` | Global agents (10 ตัว) | Global |

## ประเด็นสำคัญ

- **CLAUDE.md ทำงานอย่างไร** — ถูกโหลดทุก session เป็น system prompt บอก AI เรื่อง architecture, key files, make targets, slash commands ที่ใช้ได้
- **Settings Cascade** — Global → Project → Local (inner overrides outer); Hooks **merge รวมกัน** ไม่ใช่ทับกัน
- **Rules auto-load** — ทุกไฟล์ใน `rules/` ถูกโหลดเข้า context อัตโนมัติ ไม่ต้องบอก AI ซ้ำ
- **CLAUDE.md ต้องอัพเดท placeholders** เมื่อ clone boilerplate: `<service-name>`, `<module-path>`
- **OpenCode** คือ alternative CLI ที่ใช้ `.opencode/` แทน `.claude/` — รวมทุกอย่างใน AGENTS.md ไฟล์เดียว

## ตัวอย่าง / กรณีศึกษา

**Session Lifecycle:**
1. เปิด Claude Code → SessionStart hook → โหลด CLAUDE.md + rules/ + settings
2. User พิมพ์ข้อความ → Claude ตัดสินใจ: Skill / Agent / ทำเอง
3. Tool call → PreToolUse hooks → ทำ tool → PostToolUse hooks
4. Context ใกล้เต็ม → PreCompact hook → auto-compact
5. ปิด session → Stop hook → SessionEnd hooks

**ทีม copy boilerplate ไปโปรเจกต์ใหม่:**
1. clone repo
2. แก้ `<service-name>` ทุกไฟล์ใน `.claude/`
3. อัพเดท Jira project info ใน `rules/project.md`
4. สร้าง `settings.local.json` พร้อม permissions + MCP servers
5. `make run` → AI รู้จักโปรเจกต์ใหม่ทันที

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/ai-agents-system|AI Agents System]] — Agent definitions อยู่ใน `.claude/agents/`
- [[wiki/concepts/ai-hooks-system|AI Hooks System]] — Hook configurations ใน settings.json
- [[wiki/concepts/commands-and-skills|Commands and Skills]] — Slash commands และ AI-detected skills
- [[wiki/concepts/mcp-model-context-protocol|MCP Model Context Protocol]] — External service integration ผ่าน settings.local.json

## แหล่งที่มา

- [[wiki/sources/ai-context-phase1|Phase 1: โครงสร้าง .claude]]
- [[wiki/sources/ai-context-phase3|Phase 3: .opencode, MCP, Security]]
- [[wiki/sources/ai-context-phase5|Phase 5: Deep Dive]]
- [[wiki/sources/ai-context-phase7|Phase 7: Global Agents + Practical Manual]]
