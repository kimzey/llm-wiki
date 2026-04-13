---
title: "MCP — Model Context Protocol"
type: concept
tags: [mcp, external-services, jira, outline, gitlab, protocol, ai-tools]
sources:
  - ai-context-phase3.md
  - ai-context-phase5.md
related:
  - wiki/concepts/claude-code-ai-cli.md
  - wiki/concepts/ai-agents-system.md
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

MCP (Model Context Protocol) คือมาตรฐานที่ให้ AI tools เชื่อมต่อกับ external services ผ่าน protocol เดียวกัน เหมือนเป็น "USB port" สำหรับ AI — เสียบ server ไหนก็ใช้ได้ ใช้งานใน Claude Code ผ่าน tools ที่ชื่อขึ้นต้นด้วย `mcp__`

## อธิบาย

MCP server คือ service ที่ expose capabilities ให้ AI เรียกใช้ผ่าน protocol มาตรฐาน โดยมี 3 transport types:

| Type | วิธีสื่อสาร | ใช้กับ |
|---|---|---|
| **HTTP** | Request-Response | Stateless services (Outline) |
| **SSE** | Server-Sent Events (streaming) | Real-time services (Jira, GitLab) |
| **stdio** | Standard I/O (subprocess) | Local tools (Context7, gopls-lsp) |

**MCP Servers ในโปรเจกต์ Sellsuki Boilerplate:**

| Server | URL Pattern | Protocol | ใช้ทำอะไร |
|---|---|---|---|
| **Outline** | `mcp-outline.internal.../mcp` | HTTP | Internal wiki, documentation, ADRs |
| **Jira** | `mcp.atlassian.com/v1/sse` | SSE | Project management, tickets, sprints |
| **GitLab** | `mcp-gitlab.internal.../sse` | SSE | Source code, MR management, CI/CD |

**Tool Naming Convention:**
```
mcp__<server-name>__<tool-name>
ตัวอย่าง:
  mcp__jira__create_issue
  mcp__outline__search_documents
  mcp__gitlab__get_merge_request
  mcp__plugin_context7_context7__query-docs
  mcp__ide__getDiagnostics
```

## ประเด็นสำคัญ

- **เปิดใช้ใน Claude Code** — ต้องระบุใน `settings.local.json`:
  ```json
  "enabledMcpjsonServers": ["outline", "jira", "gitlab"]
  ```
- **Agent ใช้ MCP** — ต้องมี `mcp__xxx` ใน tools list ของ agent นั้น
- **Main session** — ใช้ MCP ได้ทุก server ที่เปิดใน settings.local.json
- **Agent tools granularity** — ระบุ `mcp__jira` = ใช้ได้ **ทุก tool** ใน Jira server (ไม่ใช่แค่ตัวเดียว)

**Plugins ทำงานผ่าน MCP เช่นกัน:**
- **Context7** — `mcp__plugin_context7_context7__*` ค้นหา library docs ล่าสุด
- **gopls-lsp** — `mcp__ide__getDiagnostics` Go Language Server

**Config ใน OpenCode** (`.opencode/opencode.jsonc`):
```jsonc
{
  "mcp": {
    "jira": { "type": "sse", "url": "..." },
    "outline": { "type": "http", "url": "..." }
  }
}
```

## ตัวอย่าง / กรณีศึกษา

**Developer Agent ย้าย Jira ticket:**
```
Developer agent เรียก mcp__jira__transition_issue
→ Jira ticket LR-42 เปลี่ยนสถานะเป็น "In Progress"
```

**PO Agent สร้าง Epic:**
```
PO agent เรียก mcp__jira__create_issue (type: Epic)
→ ได้ Epic ID กลับมา
→ เรียก mcp__jira__create_issue (type: Story) หลายครั้ง ผูกกับ Epic
```

**ถ้า MCP server ไม่ทำงาน:**
```
อาการ: Agent บอก "cannot use mcp__jira"
สาเหตุ:
  a) settings.local.json ไม่ได้เปิด server
  b) Agent ไม่มี mcp__jira ใน tools list
  c) MCP server URL ผิดหรือ down
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/ai-agents-system|AI Agents System]] — Agent ใช้ MCP ผ่าน tools list ใน YAML
- [[wiki/concepts/claude-code-ai-cli|Claude Code AI CLI]] — MCP เปิดใช้ผ่าน settings.local.json

## แหล่งที่มา

- [[wiki/sources/ai-context-phase3|Phase 3: .opencode, MCP, Security]]
- [[wiki/sources/ai-context-phase5|Phase 5: Deep Dive]]
