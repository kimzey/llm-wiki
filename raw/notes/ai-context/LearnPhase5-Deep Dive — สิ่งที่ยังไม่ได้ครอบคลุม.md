# Learn Phase 5: Deep Dive — สิ่งที่ยังไม่ได้ครอบคลุม

> เอกสารนี้เติมเต็มทุกมิติที่ Phase 1-4 ยังไม่ได้ลง ได้แก่:
> 1. Settings Cascade (Global vs Project vs Local)
> 2. Hook Lifecycle แบบสมบูรณ์
> 3. Source Code จริง — ดูว่า AI config map กับโค้ดอย่างไร
> 4. Global Plugins (Context7, gopls-lsp)
> 5. MCP Protocol Deep Dive
> 6. ข้อจำกัด, gotchas, และสิ่งที่มักพลาด
> 7. วิธีขยายระบบ (เพิ่ม agent, skill, hook, MCP)

---

## 1. Settings Cascade — 3 ระดับที่ซ้อนกัน

Claude Code มีการตั้งค่า 3 ระดับ ซ้อนกันแบบ **inner overrides outer**:

```
┌─────────────────────────────────────────────────┐
│  Level 1: Global (~/.claude/settings.json)      │
│  ├── ใช้กับทุกโปรเจกต์ทุก session               │
│  ├── hooks, plugins, model, language             │
│  └── commit เข้า dotfiles ส่วนตัว                │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │  Level 2: Project (.claude/settings.json)  │  │
│  │  ├── ใช้กับโปรเจกต์นี้เท่านั้น              │  │
│  │  ├── hooks เฉพาะโปรเจกต์                    │  │
│  │  └── commit เข้า git (ทีมใช้ร่วมกัน)        │  │
│  │                                             │  │
│  │  ┌───────────────────────────────────────┐  │  │
│  │  │  Level 3: Local (settings.local.json) │  │  │
│  │  │  ├── ใช้เฉพาะเครื่องนี้               │  │  │
│  │  │  ├── permissions, MCP servers          │  │  │
│  │  │  └── ไม่ commit (.gitignore)           │  │  │
│  │  └───────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

### ของจริงในโปรเจกต์นี้:

| ระดับ | ไฟล์ | เนื้อหา | Commit? |
|-------|------|---------|---------|
| **Global** | `~/.claude/settings.json` | model=opus, hooks 10+ ตัว, plugins, language=Thai | ไม่ (dotfiles ส่วนตัว) |
| **Project** | `.claude/settings.json` | hooks 1 ตัว (env-protection) | ใช่ (ทีมใช้ร่วม) |
| **Local** | `.claude/settings.local.json` | permissions (make, go vet, tsc), MCP servers | ไม่ (.gitignore) |

### การ Merge:

**Hooks merge = รวมกัน** ไม่ใช่ทับกัน:
```
Session จริงจะมี hooks ทั้งหมด:
├── จาก Global: tmux reminder, git push review, doc blocker, prettier,
│                tsc check, console.log warning, console.log audit,
│                suggest-compact, session-start, session-end, evaluate-session
└── จาก Project: env-protection (Read matcher)
```

**Permissions = Local เท่านั้น:**
- Global ไม่มี permissions
- Project ไม่มี permissions
- Local กำหนด allowed commands

### Rules Cascade:

```
~/.claude/rules/ (Global Rules — 8 ไฟล์)
├── agents.md          ← agent orchestration guide
├── coding-style.md    ← immutability, file organization
├── git-workflow.md    ← commit format, PR workflow
├── hooks.md           ← hooks best practices
├── patterns.md        ← API response, repository pattern
├── performance.md     ← model selection, context window
├── security.md        ← mandatory security checks
└── testing.md         ← 80% coverage, TDD

.claude/rules/ (Project Rules — 6 ไฟล์)
├── architecture.md    ← Clean Architecture 4 layers
├── conventions.md     ← Go naming, error, tracing
├── testing.md         ← 70% coverage, table-driven
├── codegen.md         ← TypeSpec pipeline
├── project.md         ← Jira project info
└── outline-collections.md ← Outline IDs
```

**ทุก session ได้ rules ทั้ง 14 ไฟล์** (8 global + 6 project)

**ข้อสังเกต:** Global rules บอก coverage 80%, Project rules บอก 70% — **Project rules ชนะ** เพราะเฉพาะเจาะจงกว่า

---

## 2. Hook Lifecycle — เจาะลึกทุก Hook Type

### 2.1 Hook Types ทั้งหมด

| Hook Type | ทำงานเมื่อ | ใช้ทำอะไร |
|-----------|-----------|----------|
| **SessionStart** | เปิด session ใหม่ | Initialize state, load context |
| **PreToolUse** | ก่อน tool ทำงาน | Validate, block, modify |
| **PostToolUse** | หลัง tool ทำงาน | Format, check, log |
| **PreCompact** | ก่อน context compact | Save important context |
| **Stop** | Session จบ (agent หยุด) | Final checks, audit |
| **SessionEnd** | Session ปิดสมบูรณ์ | Cleanup, evaluate |

### 2.2 Hook ทั้งหมดที่ทำงานจริงในโปรเจกต์นี้

```
SessionStart
└── session-start.js          ← initialize session state

PreToolUse (ก่อนทุก tool call)
├── env-protection.sh         ← [Project] บล็อก Read .env files
├── tmux-blocker              ← [Global] บล็อก dev server ที่ไม่อยู่ใน tmux
├── tmux-reminder             ← [Global] แนะนำ tmux สำหรับ long-running commands
├── git-push-review           ← [Global] เตือนก่อน git push
├── doc-blocker               ← [Global] บล็อกสร้าง .md/.txt ที่ไม่จำเป็น
└── suggest-compact.js        ← [Global] แนะนำ compact เมื่อ context ใกล้เต็ม

PostToolUse (หลังทุก tool call)
├── pr-creation-logger        ← [Global] log PR URL หลัง gh pr create
├── build-analysis            ← [Global] analysis หลัง npm build
├── prettier                  ← [Global] auto-format .js/.ts/.tsx/.jsx หลัง Edit
├── tsc-check                 ← [Global] TypeScript check หลัง Edit .ts/.tsx
└── console-log-warning       ← [Global] เตือนถ้ามี console.log หลัง Edit

PreCompact
└── pre-compact.js            ← save important context ก่อน compact

Stop (agent หยุดทำงาน)
└── console-log-audit         ← [Global] ตรวจ console.log ในไฟล์ที่แก้

SessionEnd
├── session-end.js            ← cleanup
└── evaluate-session.js       ← evaluate session quality
```

### 2.3 Hook Protocol — วิธีสื่อสาร

```
Claude Code                         Hook Script
    │                                    │
    ├── เตรียม tool call               │
    │                                    │
    ├── stdin: JSON ──────────────────► │ รับ tool input
    │   {                                │
    │     "tool": "Read",                │
    │     "tool_input": {                │
    │       "file_path": "/path/.env"    │
    │     }                              │
    │   }                                │
    │                                    │
    │                              ◄──── │ ตัดสินใจ
    │                                    │
    │  Exit Code:                        │
    │  0 = อนุญาต (stdout = modified JSON)
    │  1 = error แต่ไม่บล็อก (stderr = warning)
    │  2 = บล็อก! (stderr = reason)       │
    │                                    │
    ├── ถ้า exit 0: ทำ tool call         │
    ├── ถ้า exit 2: ยกเลิก tool call     │
    └── แสดง stderr ให้ user เห็น        │
```

### 2.4 Matcher Syntax

```javascript
// Simple: match tool name
"matcher": "Read"

// Expression: match tool + input
"matcher": "tool == \"Bash\" && tool_input.command matches \"git push\""

// Wildcard: match everything
"matcher": "*"

// Complex: match tool + file extension
"matcher": "tool == \"Edit\" && tool_input.file_path matches \"\\\\.(ts|tsx)$\""
```

---

## 3. Source Code Map — AI Config ↔ Source Code จริง

### 3.1 Entity Layer (ตัวอย่าง: store)

**AI config พูดถึง:**
> Entity: Pure domain structs, Validate(), sentinel errors. Zero framework deps.

**Source code จริง:** `src/entity/store/store.go`
```go
type Store struct {
    Id           string
    ProviderCode string `validate:"required,min=2,max=64,slug"`
    CompanyId    string `validate:"required,min=2,max=64,slug"`
    Name         string `validate:"required,min=3,max=128"`
    Code         string `validate:"required,min=2,max=32,slug"`
    Address      Address        `validate:"required"`
    TaxInformation TaxInformation `validate:"required"`
}

func (s Store) Validate() error {
    v := validator.New()
    v.RegisterValidation("slug", ValidateSlug)
    return v.Struct(s)
}
```

**สังเกต:** ตรงกับ config ทุกประการ — struct + Validate() + custom validator + ไม่ import project package ใดๆ

### 3.2 Repository Interface Layer

**AI config พูดถึง:**
> Repository interfaces defined in `src/use_case/repository/repository.go`

**Source code จริง:** 6 interfaces
```
PermissionRepository     ← Ory Keto (RBAC)
IdentityRepository       ← Ory Kratos (auth)
DistributedLockRepository ← Redis (locks)
StoreRepository          ← PostgreSQL/GORM
OrderRepository          ← PostgreSQL/GORM
ProductRepository        ← External service
```

แต่ละตัวมี `HealthChecker` embedded = ทุก repository ต้อง implement `HealthCheck()` + `Name()`

**Shared interfaces:**
```go
type Transaction interface {
    Commit() error
    Rollback() error
}

type Lock interface {
    Release(ctx context.Context) error
    Extend(ctx context.Context, ttl time.Duration) error
}
```

### 3.3 Repository Implementation (ตัวอย่าง: store_repository)

**AI config พูดถึง:**
> แต่ละ domain มี 4-5 ไฟล์

**Source code จริง:**
```
src/repository/store_repository/
├── store_repository.go          ← tracer var
├── postgres_gorm.go             ← GORM implementation
├── postgres_gorm_model.go       ← GORM model + conversion
└── postgres_gorm_options.go     ← filter/list/sort handlers
```

**สังเกต:** store_repository ไม่มี `dummy.go`! (ใช้ GORM จริงเสมอ)
แต่ permission_repository และ identity_repository มี dummy → ดูจาก DI wiring:

```go
// cmd/generics_server/helper.go
permissionRepository = permission_repository.NewDummy()  // ← ใช้ dummy
identityRepository = identity_repository.NewDummy()      // ← ใช้ dummy
storeRepository = store_repository.NewGormPostgres(db)   // ← ใช้ GORM จริง
orderRepository = order_repository.NewGormPostgres(db)   // ← ใช้ GORM จริง
productRepository = product_repository.NewDummy()        // ← ใช้ dummy
```

### 3.4 UseCase Layer

**AI config พูดถึง:**
> UseCase struct + New(...) constructor, tracing mandatory

**Source code จริง:** `src/use_case/use_case.go`
```go
type UseCase struct {
    permissionRepository      repository.PermissionRepository
    identityRepository        repository.IdentityRepository
    distributedLockRepository repository.DistributedLockRepository
    storeRepository           repository.StoreRepository
    orderRepository           repository.OrderRepository
    productRepository         repository.ProductRepository
}

func New(...) *UseCase { return &UseCase{...} }
```

**สังเกต:** ทุก field เป็น **interface** ไม่ใช่ concrete type = Dependency Inversion ตรงตาม config

### 3.5 Use Case Method จริง (StoreCreateOrder)

**AI config บอก pattern:**
```
1. tracer.Start() + defer sp.End()
2. checkPermission
3. Validate()
4. Repository call
5. sp.AddEvent() ทุก checkpoint
6. sp.RecordError() ก่อน return error
```

**Source code จริง:** `src/use_case/store.go:16-99` ทำตามทุกข้อ:
```
✅ tracer.Start(ctx, "use_case.StoreCreateOrder")
✅ defer sp.End()
✅ checkPermission → sp.AddEvent("check permission, allowed")
✅ distributed lock → sp.AddEvent("lock acquired")
✅ o.Flow.Init(o) (state machine)
✅ o.Validate() → sp.AddEvent("validate data")
✅ CreateOrderWithTransaction → deferred rollback with committed flag
✅ ReserveProduct
✅ Commit → otxCommitted = true
✅ sp.RecordError(err) ก่อนทุก return error
```

### 3.6 Error Flow — จาก Entity ถึง HTTP Response

```
src/use_case/model/errors.go                    ← กำหนด sentinel
├── ErrPermissionDenied = errors.New("permission denied")
├── ErrOrderNotFound = errors.New("order not found")
└── ErrOrderNoAlreadyExist = errors.New("order no already exist")
        │
        │  use case throw error
        ▼
src/interface/fiber_server/route/typespec/route.go  ← handler จับ error
├── err := r.useCase.StoreCreateOrder(...)
├── if err != nil { return helper.ErrorHandler(c, err) }
        │
        ▼
src/interface/fiber_server/helper/handler.go     ← ErrorHandler loop
├── for iErr, code := range errorList {
│       if errors.Is(err, iErr) {
│           return SendError(c, code.StatusCode, err, code.ErrorCode)
│       }
│   }
├── fallback: 500 "unexpected_error"
        │
        ▼
src/interface/fiber_server/helper/errors.go      ← error → HTTP mapping
├── ErrOrderNoAlreadyExist: {500, "order_no_already_exist"}
└── ErrOrderNotFound:       {404, "order_not_found"}
        │
        ▼
HTTP Response:
{
  "error": "order not found",
  "error_code": "order_not_found",
  "issue_id": "<trace_id>"     ← จาก OpenTelemetry span!
}
```

**สังเกต:** `issue_id` ดึงจาก `span.SpanContext().TraceID()` → ผู้ใช้ส่ง issue_id กลับมา → หา trace ใน Jaeger ได้เลย

### 3.7 DI Wiring — จุดเชื่อมทั้งหมด

```
cmd/generics_server/helper.go
├── initEnvironment()      ← load .env → parse config struct
├── initLogger()           ← sellsuki-go-logger
├── initTracer()           ← OpenTelemetry → OTLP gRPC → Jaeger
├── initRepositories()     ← สร้าง repository implementations
│   ├── permission_repository.NewDummy()
│   ├── identity_repository.NewDummy()
│   ├── distributed_lock_repository.NewRedis(redis)
│   ├── store_repository.NewGormPostgres(db)
│   ├── order_repository.NewGormPostgres(db)
│   └── product_repository.NewDummy()
│
└── use_case.New(
        permissionRepo,
        identityRepo,
        distributedLockRepo,
        storeRepo,
        orderRepo,
        productRepo,
    )
    │
    └── initInterfaces(cfg, uc)
        ├── fiber_server.New(uc, ...)    ← HTTP :8080
        ├── grpc_server.New(uc)          ← gRPC :50051
        └── kafka_consumer_worker.New(uc) ← Kafka consumer
```

---

## 4. Global Plugins

### 4.1 Plugins ที่เปิดใช้

```json
// ~/.claude/settings.json
"enabledPlugins": {
    "context7@claude-plugins-official": true,
    "gopls-lsp@claude-plugins-official": true
}
```

**Context7:**
- ให้ AI ค้นหา documentation ล่าสุดของ library ได้
- เรียกผ่าน `mcp__plugin_context7_context7__resolve-library-id` + `query-docs`
- เช่น ถ้า AI ต้องเขียน GORM code → ค้นหา GORM docs ล่าสุดได้

**gopls-lsp:**
- เชื่อม Go Language Server (gopls) เข้ากับ Claude Code
- ให้ AI เข้าถึง Go diagnostics, type info, completions
- เรียกผ่าน `mcp__ide__getDiagnostics`

### 4.2 ความสำคัญต่อโปรเจกต์

เมื่อ Developer agent เขียน Go code:
1. **Context7** → ค้นหา GORM/Fiber/validator docs ล่าสุด
2. **gopls-lsp** → ตรวจ type errors, missing imports ทันที
3. **PostToolUse hooks** → ไม่มี Go-specific hook (มีแค่ TS/JS)

---

## 5. MCP Protocol — เจาะลึก

### 5.1 Protocol Types

| Type | วิธีสื่อสาร | ใช้กับ |
|------|------------|--------|
| **HTTP** | Request-Response | Outline (stateless) |
| **SSE** | Server-Sent Events (streaming) | Jira, GitLab (real-time) |
| **stdio** | Standard I/O (subprocess) | Local tools (Context7, gopls) |

### 5.2 MCP Tool Naming Convention

```
mcp__<server-name>__<tool-name>

เช่น:
mcp__jira__create_issue
mcp__outline__search_documents
mcp__plugin_context7_context7__query-docs
mcp__ide__getDiagnostics
```

### 5.3 Agent Tools ↔ MCP Mapping

เมื่อ agent ระบุ `mcp__jira` ใน tools list:
```yaml
tools:
  - mcp__jira
```

หมายความว่า agent สามารถเรียก **ทุก tool** ภายใต้ jira server:
- `mcp__jira__create_issue`
- `mcp__jira__get_issue`
- `mcp__jira__transition_issue`
- `mcp__jira__add_comment`
- ฯลฯ

ไม่ใช่แค่ 1 tool แต่เป็น **ทุก tool ใน server นั้น**

---

## 6. ข้อจำกัดและ Gotchas

### 6.1 Agent Context Isolation

```
⚠️ Agent ทำงานใน subprocess แยก context

Main session:                    Agent subprocess:
├── conversation history ✅      ├── conversation history ❌ (ไม่เห็น)
├── previous tool results ✅     ├── previous tool results ❌
├── CLAUDE.md ✅                 ├── CLAUDE.md ✅
├── rules/ ✅                    ├── rules/ ✅
└── agent files ❌               └── agent's own file ✅ (เฉพาะตัว)
```

**ปัญหา:** ถ้า main session คุยเรื่อง order มา 20 ข้อความ แล้ว spawn Developer agent → agent ไม่รู้เลยว่าคุยอะไรมา ต้องใส่ context ทั้งหมดใน Task prompt

### 6.2 Skill/Command ไม่มี Tool Restriction

```
⚠️ Skill/Command ใช้ tools ทั้งหมดของ main session

สิ่งที่เกิดได้:
- /unit-test (ซึ่งเป็น read-only task) สามารถเรียก Edit tool แก้โค้ดได้
- /review-arch (ซึ่งควรแค่ตรวจ) สามารถเรียก Write tool สร้างไฟล์ได้

AI ต้อง "เลือกที่จะไม่ทำ" ตาม prompt instructions ไม่ใช่ถูกบังคับ
```

### 6.3 Generated Files Trap

```
⚠️ ห้ามแก้ แต่ไม่มีกลไกบังคับทางเทคนิค

ไฟล์เหล่านี้ระบุว่า "ห้ามแก้" ใน docs:
- openapi.yaml
- spec.gen.go
- *.pb.go

แต่ไม่มี hook บล็อกการ Edit ไฟล์เหล่านี้!
AI อาจแก้ได้ถ้า prompt ไม่ชัดเจนพอ

วิธีแก้: เพิ่ม PreToolUse hook ที่บล็อก Edit/Write ไฟล์ generated
```

### 6.4 Coverage Threshold Conflict

```
⚠️ Global rules: 80% vs Project rules: 70%

~/.claude/rules/testing.md     → "Minimum Test Coverage: 80%"
.claude/rules/testing.md       → "CI requires >= 70% coverage"
Makefile check-coverage        → threshold = 70

ในทางปฏิบัติ: 70% ชนะ เพราะ CI ใช้ Makefile
```

### 6.5 Dummy Repository ≠ Complete

```
⚠️ ไม่ใช่ทุก domain จะมี dummy.go

มี dummy:     permission, identity, product, distributed_lock
ไม่มี dummy:  store, order (ใช้ GORM เสมอ)

ถ้าเพิ่ม domain ใหม่ด้วย /new-feature จะสร้าง dummy
แต่ domain เดิม (store, order) ไม่มี → ต้องมี DB เสมอ
```

### 6.6 Hook ที่ไม่เกี่ยวกับ Go

```
⚠️ Global hooks หลายตัวเป็น JS/TS-centric

- Prettier hook      → auto-format .js/.ts/.tsx/.jsx (ไม่ทำกับ .go)
- TSC check hook     → TypeScript check (ไม่ทำกับ .go)
- console.log hook   → เตือน console.log (Go ใช้ log.Println)

สำหรับ Go project → hooks เหล่านี้ไม่ทำงาน (matcher ไม่ match .go files)
ยกเว้นตอนแก้ TypeSpec (.ts) ใน cmd/typespec_openapi_generator/
```

---

## 7. วิธีขยายระบบ

### 7.1 เพิ่ม Agent ใหม่

**สร้างไฟล์:** `.claude/agents/devops.md`
```yaml
---
name: DevOps
description: Use this agent for CI/CD, Docker, Kubernetes, infrastructure.
tools:
  - Read
  - Bash
  - Glob
  - Grep
---

You are a DevOps engineer for <service-name>.
...
```

**ไม่ต้องแก้ไฟล์อื่น** — Claude จะรู้จัก agent ใหม่จาก description อัตโนมัติ

### 7.2 เพิ่ม Skill + Command ใหม่

**1. สร้าง Skill:** `.claude/skills/deploy.md`
```yaml
---
description: Deploy to staging/production environment. Use when user wants to deploy.
---

Deploy the service.
Target: $ARGUMENTS
...
```

**2. สร้าง Command:** `.claude/commands/deploy.md`
```
Deploy the service.
Target: $ARGUMENTS
...
```

**3. อัพเดท CLAUDE.md** (optional แต่แนะนำ):
เพิ่มใน Slash commands table

### 7.3 เพิ่ม Hook ใหม่

**ตัวอย่าง: บล็อกแก้ generated files**

**1. สร้าง script:** `.claude/hooks/protect-generated.sh`
```bash
#!/bin/bash
input=$(cat)
file_path=$(echo "$input" | python3 -c \
  "import sys,json; print(json.load(sys.stdin).get('file_path',''))" 2>/dev/null)

if [[ "$file_path" == *"spec.gen.go"* ]] || \
   [[ "$file_path" == *"openapi.yaml"* ]] || \
   [[ "$file_path" == *".pb.go"* ]]; then
    echo "Blocked: Cannot edit generated file" >&2
    exit 2
fi
```

**2. เพิ่มใน settings.json:**
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit",
        "hooks": [{ "type": "command", "command": "bash .claude/hooks/protect-generated.sh" }]
      },
      {
        "matcher": "Write",
        "hooks": [{ "type": "command", "command": "bash .claude/hooks/protect-generated.sh" }]
      }
    ]
  }
}
```

### 7.4 เพิ่ม MCP Server ใหม่

**สำหรับ Claude Code:**

เพิ่มใน project-level `.mcp.json` หรือ global MCP config:
```json
{
  "mcpServers": {
    "slack": {
      "type": "sse",
      "url": "https://mcp-slack.internal.company.com/sse"
    }
  }
}
```

แล้วเปิดใน `settings.local.json`:
```json
{
  "enabledMcpjsonServers": ["outline", "jira", "gitlab", "slack"]
}
```

แล้วเพิ่มใน agent tools list:
```yaml
tools:
  - mcp__slack
```

**สำหรับ OpenCode:**

เพิ่มใน `opencode.jsonc`:
```jsonc
{
  "mcp": {
    "slack": {
      "type": "sse",
      "url": "https://mcp-slack.internal.company.com/sse"
    }
  }
}
```

---

## 8. Lifecycle ทั้งหมด — จากเปิด Session ถึงปิด

```
1. เปิด Claude Code CLI
   │
   ▼
2. SessionStart hook ทำงาน
   ├── session-start.js (initialize state)
   │
   ▼
3. Load Context
   ├── CLAUDE.md (project)
   ├── rules/* (global 8 + project 6 = 14 ไฟล์)
   ├── settings.json (global + project merge)
   ├── settings.local.json (permissions + MCP)
   └── plugins (context7 + gopls-lsp)
   │
   ▼
4. Session Loop
   │
   ├── User พิมพ์ข้อความ / คำสั่ง
   │   │
   │   ├── ถ้าเป็น /command → load command .md
   │   ├── ถ้าเป็นข้อความ → Claude ตัดสินใจ
   │   │   ├── ใช้ Skill? → load skill .md
   │   │   ├── ใช้ Agent? → spawn subprocess (Task tool)
   │   │   └── ทำเอง? → ใช้ tools โดยตรง
   │   │
   │   ▼
   │   Tool Call
   │   ├── PreToolUse hooks ทำงาน
   │   │   ├── env-protection (ถ้า Read)
   │   │   ├── tmux reminder (ถ้า Bash)
   │   │   ├── doc blocker (ถ้า Write .md)
   │   │   └── suggest-compact (ถ้า Edit/Write)
   │   │
   │   ├── Exit 2? → บล็อก! → บอก user
   │   ├── Exit 0? → ทำ tool call
   │   │
   │   ├── PostToolUse hooks ทำงาน
   │   │   ├── prettier (ถ้า Edit .js/.ts)
   │   │   ├── tsc check (ถ้า Edit .ts)
   │   │   ├── console.log warning (ถ้า Edit .js/.ts)
   │   │   └── PR logger (ถ้า Bash gh pr create)
   │   │
   │   └── กลับไป User พิมพ์ข้อความ
   │
   ▼
5. Context ใกล้เต็ม?
   ├── PreCompact hook → pre-compact.js (save context)
   ├── Auto-compact ย่อ context
   └── กลับไป Session Loop
   │
   ▼
6. User จบ session / Agent หยุด
   ├── Stop hook → console.log audit
   │
   ▼
7. SessionEnd hooks
   ├── session-end.js (cleanup)
   └── evaluate-session.js (evaluate quality)
   │
   ▼
8. Session ปิด
```

---

## 9. Checklist สุดท้าย — "ผมเข้าใจครบหรือยัง?"

### ระดับ Beginner (ใช้งานได้)
- [ ] รู้ว่า `.claude/` คืออะไร ทำไมถึงมี
- [ ] ใช้ slash commands ได้ (`/new-feature`, `/unit-test`)
- [ ] รู้ว่าห้ามแก้ generated files
- [ ] รู้ว่า AI อ่าน .env ไม่ได้ (และทำไม)

### ระดับ Intermediate (ปรับแต่งได้)
- [ ] เข้าใจ 4 agents + ความต่างของ tools
- [ ] เข้าใจ Skill vs Command (คู่แฝด, trigger ต่างกัน)
- [ ] รู้ Settings cascade (global > project > local)
- [ ] รู้ Error flow ตั้งแต่ entity → HTTP response
- [ ] เข้าใจ DI wiring ใน `cmd/generics_server/helper.go`

### ระดับ Advanced (สร้างเองได้)
- [ ] สร้าง agent ใหม่ พร้อมกำหนด tools
- [ ] สร้าง skill + command คู่ใหม่
- [ ] สร้าง hook ใหม่ (PreToolUse/PostToolUse)
- [ ] เพิ่ม MCP server ใหม่ + ผูกกับ agent
- [ ] เข้าใจ Hook protocol (stdin JSON, exit codes)
- [ ] เข้าใจ context isolation ระหว่าง agent กับ main session
- [ ] รู้ gotchas ทั้งหมด (coverage conflict, missing dummy, etc.)

### ระดับ Expert (ออกแบบระบบเองได้)
- [ ] ออกแบบ agent team ใหม่สำหรับโปรเจกต์อื่น
- [ ] เขียน Hook lifecycle ที่ซับซ้อน (PreCompact save/restore)
- [ ] ปรับ TypeSpec → OpenAPI → Go pipeline ให้รองรับ domain ใหม่
- [ ] สร้าง .opencode/ equivalent สำหรับ AI tool อื่น
- [ ] ออกแบบ MCP server architecture สำหรับ internal tools

---

## สรุปรวม 5 Phases

| Phase | เนื้อหาหลัก |
|-------|------------|
| **1** | โครงสร้าง `.claude/` + `.opencode/`, CLAUDE.md, Rules, Settings, Hooks |
| **2** | Agents (4 บทบาท), Commands (9 คำสั่ง), Skills (9 ทักษะ) |
| **3** | .opencode detail, MCP Servers, Security, Clone process |
| **4** | Binding relationships — ใครผูกกับใคร อย่างไร |
| **5** | Settings Cascade, Hook Lifecycle, Source Code mapping, Plugins, ข้อจำกัด, วิธีขยาย |

**ทั้ง 5 Phases รวมกันคือความเข้าใจที่สมบูรณ์ในการ:**
1. ใช้งานระบบ AI-Assisted Development
2. ปรับแต่งให้เหมาะกับโปรเจกต์
3. สร้างระบบแบบนี้สำหรับโปรเจกต์ใหม่ตั้งแต่ศูนย์
