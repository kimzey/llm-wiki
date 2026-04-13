---
title: "Learn Phase 3: .opencode, MCP Servers, Security, การ Clone, และ Best Practices"
type: source
source_file: raw/notes/ai-context/LearnPhase3 opencode, MCP Servers, Security, การ Clone, และ Best Practices.md
url: ""
published: 2026-04-13
tags: [opencode, mcp, security, hooks, best-practices]
related:
  - wiki/concepts/mcp-model-context-protocol.md
  - wiki/concepts/ai-hooks-system.md
  - wiki/concepts/claude-code-ai-cli.md
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/ai-context/LearnPhase3 opencode, MCP Servers, Security, การ Clone, และ Best Practices.md|Original file]]

## สรุป

Phase 3 อธิบาย `.opencode/` (Claude Code alternative), MCP servers 3 ตัวที่ใช้งาน, ระบบ security แบบ defense-in-depth, ขั้นตอน clone boilerplate, และ best practices สำหรับการใช้งานจริง

## ประเด็นสำคัญ

**OpenCode vs Claude Code:**
| Claude Code | OpenCode |
|---|---|
| แยกเป็นหลายไฟล์ (CLAUDE.md + rules/ + agents/ + commands/) | รวมทุกอย่างใน AGENTS.md ไฟล์เดียว |
| มี agents, commands, skills แยก | ไม่มีระบบ agents/commands |
| hooks system (bash scripts) | plugin system (TypeScript) |

**MCP Servers 3 ตัว:**
- **Outline** (HTTP) — internal wiki/documentation
- **Jira** (SSE) — project management, tickets
- **GitLab** (SSE) — source code management

**Security (Defense in Depth):**
- Layer 1: `.claude/hooks/env-protection.sh` — PreToolUse hook → exit 2
- Layer 2: `.opencode/plugin/env-protection.ts` — tool.execute.before → throw Error
- ทั้งสองบล็อก AI ไม่ให้อ่าน .env files

## ข้อมูลที่น่าสนใจ

**MCP Tool Naming:**
```
mcp__<server-name>__<tool-name>
เช่น: mcp__jira__create_issue, mcp__outline__search_documents
```

เมื่อ agent ระบุ `mcp__jira` ใน tools list = สามารถใช้ **ทุก tool** ในนั้น ไม่ใช่แค่ตัวเดียว

**Clone boilerplate checklist:**
1. แก้ `<service-name>` และ `<module-path>` ใน CLAUDE.md + agents/*.md + AGENTS.md
2. อัพเดท project.md (Jira info) + outline-collections.md
3. สร้าง settings.local.json + เปิด MCP servers
4. อัพเดท domain list ใน po.md

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/mcp-model-context-protocol.md|MCP Model Context Protocol]]
- [[wiki/concepts/ai-hooks-system.md|AI Hooks System]]
- [[wiki/concepts/claude-code-ai-cli.md|Claude Code AI CLI]]
