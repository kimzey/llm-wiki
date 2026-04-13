# Learn Phase 3: .opencode, MCP Servers, Security, การ Clone, และ Best Practices

> เอกสารสุดท้าย ครอบคลุม .opencode/, MCP server integrations, ระบบ security,
> ขั้นตอนการ clone boilerplate, และแนวปฏิบัติที่ดีที่สุด

---

## 1. `.opencode/` — คอนฟิกสำหรับ OpenCode CLI

### 1.1 OpenCode คืออะไร?

OpenCode เป็น AI coding CLI ทางเลือก (คล้าย Claude Code) ที่ใช้ LLM ช่วยเขียนโค้ด โฟลเดอร์ `.opencode/` ทำหน้าที่เดียวกับ `.claude/` แต่สำหรับ OpenCode

### 1.2 โครงสร้างไฟล์

```
.opencode/
├── AGENTS.md              ← system prompt หลัก (เทียบเท่า CLAUDE.md)
├── opencode.jsonc          ← MCP server configuration
└── plugin/
    └── env-protection.ts   ← Plugin ป้องกัน .env
```

### 1.3 `AGENTS.md` — System Prompt ของ OpenCode

**เนื้อหาเหมือน CLAUDE.md แต่รวมทุกอย่างในไฟล์เดียว:**
- Architecture layers + forbidden imports
- Key source files
- Naming conventions
- Error handling patterns
- Tracing template
- Transaction safety pattern
- HTTP handler pattern
- Testing conventions (table-driven, mocks, 70% coverage)
- Code generation pipeline
- Make targets
- Project context (Jira)
- Outline collections

**ข้อแตกต่างจาก `.claude/`:**
| Claude Code | OpenCode |
|-------------|----------|
| แยกเป็นหลายไฟล์ (CLAUDE.md + rules/ + agents/ + commands/) | รวมทุกอย่างใน AGENTS.md ไฟล์เดียว |
| มี agents, commands, skills แยก | ไม่มีระบบ agents/commands |
| มี hooks system | มี plugin system แทน |
| settings.json + settings.local.json | opencode.jsonc ไฟล์เดียว |

### 1.4 `opencode.jsonc` — MCP Server Configuration

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "outline": {
      "type": "http",
      "url": "https://mcp-outline.internal.production.sellsuki.com/mcp"
    },
    "jira": {
      "type": "sse",
      "url": "https://mcp.atlassian.com/v1/sse"
    },
    "gitlab": {
      "type": "sse",
      "url": "https://mcp-gitlab.internal.production.sellsuki.com/sse"
    }
  }
}
```

**ความหมาย:**
- กำหนด MCP servers 3 ตัวเหมือนกับ Claude Code
- `type: "http"` = HTTP-based MCP server (Outline)
- `type: "sse"` = Server-Sent Events (Jira, GitLab) — ใช้ real-time streaming

### 1.5 `env-protection.ts` — OpenCode Plugin

```typescript
export const EnvProtection = async () => {
  return {
    "tool.execute.before": async (input: any, output: any) => {
      if (input.tool === "read" && output.args.filePath.includes(".env")) {
        throw new Error("Do not read .env files")
      }
    }
  }
}
```

**ทำงานเหมือน `env-protection.sh` ของ Claude Code:**
- Hook เข้าก่อนทุก tool execution
- ถ้า tool เป็น "read" และ filePath มี ".env" → throw error → บล็อก
- ต่างกันที่ภาษา: Claude ใช้ Bash script, OpenCode ใช้ TypeScript

---

## 2. MCP Servers — เครื่องมือภายนอกที่ AI ใช้ได้

### 2.1 MCP คืออะไร?

**MCP (Model Context Protocol)** เป็นมาตรฐานที่ให้ AI tools เชื่อมต่อกับ external services ผ่าน protocol เดียวกัน เหมือนเป็น "USB port" สำหรับ AI — เสียบ server ไหนก็ใช้ได้

### 2.2 MCP Servers ที่ใช้ในโปรเจกต์นี้

| Server | URL | Protocol | ใช้ทำอะไร |
|--------|-----|----------|----------|
| **Outline** | `https://mcp-outline.internal.production.sellsuki.com/mcp` | HTTP | Internal wiki/documentation |
| **Jira** | `https://mcp.atlassian.com/v1/sse` | SSE | Project management, tickets |
| **GitLab** | `https://mcp-gitlab.internal.production.sellsuki.com/sse` | SSE | Source code management |

### 2.3 MCP Server ใช้อย่างไร?

**ใน Claude Code:**
- เปิดใช้ใน `settings.local.json` → `"enabledMcpjsonServers": ["outline", "jira", "gitlab"]`
- AI เรียกผ่าน tools: `mcp__jira`, `mcp__outline`, `mcp__gitlab`

**ใน OpenCode:**
- กำหนดใน `opencode.jsonc` → `"mcp": { ... }`
- AI เรียกผ่าน tool name ที่ตรงกับ key

### 2.4 การใช้งานจริง

**Jira MCP (`mcp__jira`):**
- สร้าง/อ่าน/อัพเดท issues
- ย้ายสถานะ (In Progress → Done)
- เพิ่ม comments
- สร้าง sub-tasks ภายใต้ epic
- ใช้ Project Key `LR` (จาก `project.md`)

**Outline MCP (`mcp__outline`):**
- ค้นหา/สร้าง/อัพเดท pages
- จัดการ collections (ใช้ ID จาก `outline-collections.md`)
- เขียน product docs, technical docs, ADRs, test plans

**GitLab MCP (`mcp__gitlab`):**
- จัดการ MR (Merge Requests)
- ดู pipeline status
- จัดการ issues

---

## 3. ระบบ Security

### 3.1 การป้องกัน .env Files

**ปัญหา:** AI อาจอ่าน `.env` ที่มี secrets (API keys, DB passwords, tokens)

**วิธีแก้ (Defense in Depth):**

| Layer | Tool | ไฟล์ | วิธีการ |
|-------|------|------|--------|
| 1 | Claude Code | `.claude/hooks/env-protection.sh` | PreToolUse hook → exit 2 |
| 2 | OpenCode | `.opencode/plugin/env-protection.ts` | tool.execute.before → throw Error |

ทั้งสองทำงานเหมือนกัน: **ดักจับก่อน AI อ่านไฟล์ → ถ้าเป็น .env → บล็อก**

### 3.2 Permission Control (Claude Code)

**`settings.local.json`** กำหนดว่า AI รันคำสั่งอะไรได้โดยไม่ต้องถามยืนยัน:

```json
"permissions": {
  "allow": [
    "Bash(go vet:*)",
    "Bash(npx --yes tsc:*)",
    "Bash(make:*)",
    "Bash(node --check:*)"
  ]
}
```

**เฉพาะคำสั่งที่ปลอดภัยเท่านั้น:**
- `go vet` = static analysis (อ่านอย่างเดียว)
- `tsc` = TypeScript compiler (อ่านอย่างเดียว)
- `make` = build system (ควบคุมได้ผ่าน Makefile)
- `node --check` = syntax check (อ่านอย่างเดียว)

คำสั่งอื่นๆ เช่น `rm`, `git push`, `curl` ต้องถามยืนยันก่อน

### 3.3 Generated Files Protection

กฎ "ห้ามแก้ไข" สำหรับ generated files:
- `openapi.yaml` — generated by TypeSpec compile
- `spec.gen.go` — generated by oapi-codegen
- `*.pb.go` — generated by protoc
- `*_grpc.pb.go` — generated by protoc-gen-go-grpc

ถูกระบุทั้งใน `codegen.md`, `CLAUDE.md`, `AGENTS.md` เพื่อป้องกัน AI แก้ไขโดยไม่ตั้งใจ

---

## 4. ขั้นตอนการ Clone Boilerplate

เมื่อ clone boilerplate นี้ไปใช้กับโปรเจกต์ใหม่ ต้องทำ:

### 4.1 แก้ไข Placeholders

| Placeholder | แก้เป็น | ตัวอย่าง |
|-------------|---------|----------|
| `<service-name>` | ชื่อ service จริง | `order-service` |
| `<module-path>` | Go module path | `gitlab.sellsuki.com/sellsuki/team/backend/order-service` |

**ไฟล์ที่ต้องแก้:**
- `.claude/CLAUDE.md`
- `.claude/agents/developer.md`
- `.claude/agents/po.md`
- `.claude/agents/qa.md`
- `.claude/agents/solution-architect.md`
- `.opencode/AGENTS.md`

### 4.2 อัพเดทข้อมูลโปรเจกต์

**`.claude/rules/project.md`** — แก้ Jira project details:
```
| Project Name | <your-project-name> |
| Project Key | <YOUR_KEY> |
| Project ID | <your-id> |
```

**`.claude/rules/outline-collections.md`** — แก้ collection IDs ถ้าใช้ Outline workspace อื่น

**`.opencode/AGENTS.md`** — แก้ส่วน "Project Context" และ "Outline Collections"

### 4.3 ตั้งค่า MCP Servers

**สำหรับ Claude Code:**
สร้าง `.claude/settings.local.json`:
```json
{
  "permissions": {
    "allow": ["Bash(make:*)"]
  },
  "enabledMcpjsonServers": ["outline", "jira", "gitlab"]
}
```

**สำหรับ OpenCode:**
แก้ `.opencode/opencode.jsonc` ถ้า MCP URLs ต่างกัน

### 4.4 อัพเดท PO Agent Domain List

ใน `.claude/agents/po.md`:
```
Domains: `<list your project domains here>`
```
เปลี่ยนเป็น domain codes จริง เช่น `AUTH, OMS, PAY, NOTIF, CATALOG`

---

## 5. Best Practices — แนวปฏิบัติที่ดีที่สุด

### 5.1 Workflow แนะนำสำหรับ Feature ใหม่

```
1. PO Agent → สร้าง Epic + Stories ใน Jira
2. Solution Architect → review design, สร้าง ADR ใน Outline
3. Developer Agent → /new-feature <domain>
4. Developer Agent → /add-usecase + /add-http-route (ต่อเนื่อง)
5. Developer Agent → /unit-test (ต้อง pass)
6. Developer Agent → /coverage (ต้อง >= 70%)
7. QA Agent → review tests, validate acceptance criteria
8. Solution Architect → /review-arch src/... (ตรวจ violations)
9. Developer Agent → commit + push
```

### 5.2 เมื่อไหร่ใช้ Command ไหน

| สถานการณ์ | Command |
|-----------|---------|
| เพิ่ม domain ใหม่ทั้งหมด | `/new-feature widget` |
| เพิ่ม method ใน domain เดิม | `/add-usecase order CancelOrder` |
| TypeSpec มี route ใหม่ต้อง implement | `/add-http-route CompanyStoreCreateStore` |
| ต้องการ error ใหม่ | `/add-error ErrNotFound 404 not_found` |
| แก้ TypeSpec แล้วต้อง generate ใหม่ | `/gen-http` |
| แก้ .proto แล้วต้อง generate ใหม่ | `/gen-grpc` |
| ตรวจว่าโค้ด compile และ test pass | `/unit-test` |
| ตรวจ coverage ก่อน commit | `/coverage` |
| ตรวจ architecture compliance | `/review-arch src/use_case/` |

### 5.3 Agent Selection Guide

| ต้องการทำอะไร | Agent |
|---------------|-------|
| เขียนโค้ด, implement feature | Developer |
| สร้าง Jira tickets, เขียน stories | PO |
| เขียนเทสต์, ตรวจ coverage | QA |
| ตรวจ architecture, เขียน ADR | Solution Architect |

### 5.4 ข้อควรระวัง

1. **ห้ามแก้ generated files** — `spec.gen.go`, `openapi.yaml`, `*.pb.go` จะถูก overwrite
2. **ห้ามให้ AI อ่าน .env** — hooks จะบล็อก แต่อย่าลบ hook ออก
3. **ต้อง pass `make unit-test` ก่อนจบทุก command** — เป็นขั้นตอนสุดท้ายของทุก command
4. **Coverage >= 70%** — CI จะ fail ถ้าต่ำกว่า
5. **อัพเดท placeholders เมื่อ clone** — ไม่งั้น AI จะสับสนกับ `<service-name>`
6. **`settings.local.json` ไม่ควร commit** — มี permission ส่วนตัว

---

## 6. Diagram: ภาพรวมทั้งระบบ

```
┌─────────────────────────────────────────────────────────────────┐
│                    AI-Assisted Development System                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────┐        │
│  │ Claude Code  │    │  OpenCode    │    │  Human Dev   │        │
│  │  (CLI)       │    │  (CLI)       │    │  (IDE)       │        │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘        │
│         │                   │                    │                │
│         ▼                   ▼                    │                │
│  ┌─────────────┐    ┌──────────────┐             │                │
│  │ .claude/    │    │ .opencode/   │             │                │
│  │ CLAUDE.md   │    │ AGENTS.md    │             │                │
│  │ rules/      │    │ opencode.jsonc│            │                │
│  │ agents/     │    │ plugin/      │             │                │
│  │ commands/   │    └──────────────┘             │                │
│  │ skills/     │                                 │                │
│  │ hooks/      │                                 │                │
│  │ settings.*  │                                 │                │
│  └──────┬───────┘                                │                │
│         │                                        │                │
│         ▼                                        ▼                │
│  ┌────────────────────────────────────────────────┐              │
│  │              Go Microservice Codebase           │              │
│  │                                                 │              │
│  │  src/entity/         ← Pure domain              │              │
│  │  src/use_case/       ← Business logic           │              │
│  │  src/repository/     ← Data access              │              │
│  │  src/interface/      ← HTTP/gRPC/Kafka          │              │
│  │  cmd/                ← Entry points             │              │
│  └────────────────────────────────────────────────┘              │
│         │                                                        │
│         ▼                                                        │
│  ┌────────────────────────────────────────────────┐              │
│  │              External Services (via MCP)        │              │
│  │                                                 │              │
│  │  Jira Cloud      → Project management           │              │
│  │  Outline          → Internal documentation      │              │
│  │  GitLab           → Source code + CI/CD          │              │
│  │  PostgreSQL       → Database                    │              │
│  │  Redis            → Cache + Locks               │              │
│  │  Kafka            → Event streaming             │              │
│  │  Ory Kratos/Keto  → Identity + Permissions      │              │
│  │  Jaeger           → Distributed tracing         │              │
│  └────────────────────────────────────────────────┘              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. Quick Reference Card

### ไฟล์สำคัญ — เมื่อต้องการแก้อะไร ไปที่ไหน

| ต้องการ | ไฟล์ |
|---------|------|
| แก้กฎ architecture | `.claude/rules/architecture.md` |
| แก้ naming conventions | `.claude/rules/conventions.md` |
| แก้ test requirements | `.claude/rules/testing.md` |
| แก้ codegen pipeline | `.claude/rules/codegen.md` |
| แก้ Jira project info | `.claude/rules/project.md` |
| แก้ Outline collection IDs | `.claude/rules/outline-collections.md` |
| แก้ Developer agent behavior | `.claude/agents/developer.md` |
| แก้ PO agent behavior | `.claude/agents/po.md` |
| เพิ่ม command ใหม่ | สร้างไฟล์ใน `.claude/commands/` |
| เพิ่ม skill ใหม่ | สร้างไฟล์ใน `.claude/skills/` |
| เพิ่ม hook ใหม่ | แก้ `.claude/settings.json` + สร้าง script |
| แก้ auto-allow permissions | `.claude/settings.local.json` |
| แก้ MCP servers (OpenCode) | `.opencode/opencode.jsonc` |

### คำสั่งที่ใช้บ่อย

```bash
# Development
make run                 # Start server
make gen-all             # TypeSpec → OpenAPI → Go
make generate-grpc       # Regenerate gRPC stubs
docker compose up -d     # Start infrastructure

# Testing
make unit-test           # Run all unit tests
make check-coverage      # Verify >= 70% coverage
make integration-test    # Run integration tests
make coverage-test-html  # Visual coverage report

# Claude Code slash commands
/new-feature <domain>              # Scaffold full feature
/add-usecase <domain> <Method>     # Add use case method
/add-http-route <MethodName>       # Implement HTTP handler
/add-error <Err> <status> <key>    # Add domain error
/gen-http                          # Regenerate HTTP interface
/gen-grpc                          # Regenerate gRPC
/unit-test                         # Run tests with reporting
/coverage                          # Check coverage threshold
/review-arch <path>                # Architecture audit
```

---

## สรุป Phase 3

| สิ่งที่เรียนรู้ | รายละเอียด |
|----------------|-----------|
| `.opencode/` | เทียบเท่า `.claude/` สำหรับ OpenCode CLI รวมทุกอย่างใน AGENTS.md |
| MCP Servers | Jira + Outline + GitLab เชื่อมผ่าน MCP protocol |
| Security | .env protection (hook + plugin), permission control, generated file protection |
| Clone process | แก้ placeholders, อัพเดท project/Outline info, ตั้งค่า MCP |
| Best practices | ใช้ Agent ที่ถูกบทบาท, ทำตาม command ตามลำดับ, ตรวจ coverage ก่อน commit |

---

## สรุปรวม 3 Phases

| Phase | เนื้อหา |
|-------|---------|
| **Phase 1** | ภาพรวมโครงสร้าง, CLAUDE.md, Rules, Settings, Hooks |
| **Phase 2** | Agents (4 บทบาท), Commands (9 คำสั่ง), Skills (9 ทักษะ) |
| **Phase 3** | .opencode, MCP Servers, Security, Clone Process, Best Practices |

ทั้ง 3 Phase รวมกันคือ **ระบบ AI-Assisted Development ครบวงจร** ที่ทำให้ AI สามารถ:
1. เข้าใจสถาปัตยกรรมโปรเจกต์โดยอัตโนมัติ
2. ทำงานตามบทบาทที่กำหนด (Dev, PO, QA, Architect)
3. สร้างโค้ดตาม Clean Architecture patterns
4. จัดการงานใน Jira และเอกสารใน Outline
5. ป้องกัน security issues (secrets, generated files)
6. ตรวจสอบคุณภาพโค้ด (tests, coverage, architecture compliance)
