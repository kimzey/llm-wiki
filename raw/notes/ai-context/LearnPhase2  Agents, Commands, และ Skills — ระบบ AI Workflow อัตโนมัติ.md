# Learn Phase 2: Agents, Commands, และ Skills — ระบบ AI Workflow อัตโนมัติ

> เอกสารนี้อธิบายระบบ Agent (บทบาท AI), Commands (คำสั่งลัด), และ Skills (ทักษะ) อย่างละเอียด
> เป็นส่วนที่ทำให้ AI สามารถทำงานแทนทีมพัฒนาได้จริง

---

## 1. Agents — AI รับบทบาทต่างๆ ในทีม

### 1.1 Agent คืออะไร?

Agent คือ "บุคลิก" ที่กำหนดให้ AI ทำงานเฉพาะบทบาท แต่ละ agent มี:
- **Identity** = บทบาทชัดเจน (Developer, PO, QA, Solution Architect)
- **Tools** = เครื่องมือที่ agent นั้นใช้ได้ (Bash, Read, Edit, Jira, Outline ฯลฯ)
- **Project context** = ความรู้เฉพาะเกี่ยวกับโปรเจกต์
- **Workflow** = ขั้นตอนการทำงานที่กำหนดไว้

### 1.2 ไฟล์ Agent อยู่ที่ไหน?

```
.claude/agents/
├── developer.md         ← นักพัฒนา Go
├── po.md                ← Product Owner
├── qa.md                ← QA Engineer
└── solution-architect.md ← Solution Architect
```

### 1.3 โครงสร้างไฟล์ Agent

ทุก agent file มี 2 ส่วน:

**ส่วนที่ 1: YAML Front Matter** — metadata
```yaml
---
name: Developer
description: Use this agent for implementing features...
tools:
  - Read
  - Edit
  - Write
  - Bash
  - mcp__jira
  - mcp__outline
---
```

**ส่วนที่ 2: Markdown Body** — system prompt ของ agent
- คำอธิบาย identity
- กฎและ workflow ที่ต้องทำตาม
- checklist สำหรับงาน

---

## 2. Agent แต่ละตัวละเอียด

### 2.1 Developer Agent (`developer.md`)

**บทบาท:** Senior Go Developer — implement features end-to-end

**เครื่องมือที่ใช้ได้:**
| Tool | ใช้ทำอะไร |
|------|----------|
| Read | อ่านไฟล์ |
| Edit | แก้ไขไฟล์ |
| Write | สร้างไฟล์ใหม่ |
| Glob | ค้นหาไฟล์ |
| Grep | ค้นหาเนื้อหา |
| Bash | รันคำสั่ง terminal |
| mcp__jira | จัดการ Jira tickets |
| mcp__outline | จัดการเอกสาร Outline |

**Jira Workflow ของ Developer:**
1. ก่อนเริ่มงาน → อ่าน ticket ใน Jira, อ่าน acceptance criteria
2. เริ่มงาน → ย้ายสถานะเป็น "In Progress"
3. เสร็จงาน → เพิ่ม comment สรุปสิ่งที่ทำ
4. พร้อมรีวิว → ย้ายสถานะเป็น "Done/Review"

**Implementation Checklist (ลำดับที่ต้องทำ):**
```
1. src/entity/<domain>/           ← struct + Validate() + test
2. src/use_case/repository/       ← repository interface
3. src/use_case/repository/mock_  ← mock struct
4. src/repository/<domain>_repo/  ← GORM impl + model + options + dummy
5. src/use_case/<domain>.go       ← use case methods + tracing
6. src/use_case/use_case.go       ← wire repository field
7. cmd/generics_server/           ← DI wiring
8. src/interface/.../route.go     ← HTTP handler (ถ้าจำเป็น)
9. make unit-test                 ← ต้อง pass ก่อนจบ
```

**Coding Conventions ที่ Developer ต้องทำตาม:**
- ทุก use case method ต้องเริ่มด้วย `tracer.Start(ctx, ...)`
- Wrap errors: `fmt.Errorf("operation: %w", err)`
- Record errors ใน span: `sp.RecordError(err)`
- HTTP identity: `helper.GetIdentityFromHeader(c)`
- Transaction: ใช้ defer rollback pattern เสมอ

---

### 2.2 Product Owner Agent (`po.md`)

**บทบาท:** PO ที่จัดการ backlog, เขียน user stories, วางแผน sprint

**เครื่องมือที่ใช้ได้:**
| Tool | ใช้ทำอะไร |
|------|----------|
| Read | อ่านไฟล์โปรเจกต์ |
| Glob | ค้นหาไฟล์ |
| ToolSearch | ค้นหาเครื่องมือ MCP |
| mcp__jira | จัดการ Jira backlog |
| mcp__outline | เขียน product documentation |

**สังเกต:** PO agent **ไม่มี** Edit, Write, Bash → ไม่สามารถแก้โค้ดได้ ทำได้แค่อ่านและจัดการ Jira/Outline

**Issue Type Hierarchy (3 ระดับ, เคร่งครัดมาก):**

```
Level 1 (Epic)
└── Level 0 (Story / Tech Story / Bug / Tech Enhancement / UI Enhancement)
    └── Level -1 (Dev Task / QA Task)
```

**ห้ามใช้ "Task" หรือ "Sub-task" เด็ดขาด** — ต้องใช้ type ที่กำหนดเท่านั้น

**Summary Naming Convention:**
```
[DOMAIN] As a <role>, I want to <goal>      ← Story
[DOMAIN] As a developer, I want to <goal>   ← Tech Story
[DOMAIN] <factual description>               ← Bug
[DOMAIN] <what is being improved>            ← Enhancement
```

Domain codes: `[AUTH]`, `[OMS]`, `[PAY]`, `[NOTIF]`, `[ADMIN]`, `[CATALOG]` ฯลฯ

**Story vs Tech Story:**
- **Story** = end-user เห็นผลลัพธ์ → "As a user, I want to create an order"
- **Tech Story** = developer เห็นผลลัพธ์ → "As a developer, I want JWT middleware"

**Definition of Done ทุก Story:**
- [ ] Unit tests pass (`make unit-test`)
- [ ] Coverage >= 70% (`make check-coverage`)
- [ ] API contract updated ใน OpenAPI spec
- [ ] Architecture reviewed (ไม่มี layer violations)
- [ ] Deployed to staging
- [ ] Outline documentation updated

---

### 2.3 QA Agent (`qa.md`)

**บทบาท:** Senior QA Engineer — เขียนเทสต์, ตรวจ coverage, validate acceptance criteria

**เครื่องมือที่ใช้ได้:**
| Tool | ใช้ทำอะไร |
|------|----------|
| Read | อ่านซอร์สโค้ดและเทสต์ |
| Glob | ค้นหาไฟล์เทสต์ |
| Grep | ค้นหา pattern ในโค้ด |
| Bash | รัน `make unit-test`, `make check-coverage` |
| mcp__jira | จัดการ QA tickets |
| mcp__outline | เขียน test plans |

**สังเกต:** QA agent **ไม่มี** Edit, Write → ไม่สามารถแก้ซอร์สโค้ดได้ (แต่รัน tests ได้)

**Test Layers ตามลำดับความสำคัญ:**
1. **Entity tests** → validation paths ทุกเส้นทาง (เป้า 90%+)
2. **Use case tests** → business logic กับ mock repos (เป้า 80%+)
3. **Repository tests** → model conversion, SQL queries
4. **Interface tests** → request/response transformation

**QA Review Checklist:**
- [ ] Acceptance criteria ครบทุกข้อ
- [ ] Entity `Validate()` ครอบคลุมทุก constraint
- [ ] Use case tests ครบ: happy path, error branches, permission denied
- [ ] Repository tests: successful query, not-found, DB error
- [ ] Error types ใช้ `errors.Is` (ไม่ใช่ string comparison)
- [ ] Tracing: `tracer.Start` ในทุก use case method
- [ ] ไม่มีการแก้ generated files
- [ ] `make check-coverage` pass (>= 70%)

**Architecture Violations ที่ QA ต้องจับ:**
- Interface layer import repository packages (ผิดกฎ)
- Use case import infrastructure packages
- Missing `ctx` propagation
- Missing `sp.RecordError(err)` ก่อน return error
- Transaction ไม่มี defer rollback

---

### 2.4 Solution Architect Agent (`solution-architect.md`)

**บทบาท:** ดูแล technical vision, ตรวจ architecture, เขียน ADR

**เครื่องมือที่ใช้ได้:** เหมือน QA (Read, Glob, Grep, Bash, mcp__jira, mcp__outline)

**Infrastructure Topology ที่ Architect รู้:**
- HTTP: Fiber v2 (fasthttp) port 8080
- gRPC: port 50051
- Kafka: consumer workers with retry
- PostgreSQL: GORM ORM (1 DB per domain)
- Redis: distributed locks (bsm/redislock)
- Identity: Ory Kratos (HTTP admin API)
- Permissions: Ory Keto (gRPC, RBAC)
- Tracing: OpenTelemetry → OTLP → Jaeger
- Metrics: Prometheus at `/system/metrics`
- CI/CD: GitLab CI → Docker → Helm → Kubernetes

**Architecture Review Checklist:**
1. **Dependency violations** — ตรวจทิศทาง import
2. **Tracing completeness** — ทุก use case ต้องมี tracer
3. **Error handling** — sentinel errors, wrapping, HTTP mapping
4. **Repository pattern** — interfaces, mocks, options pattern
5. **Context propagation** — `ctx` เป็น parameter แรก, `c.UserContext()`
6. **Testing** — 70% coverage

**ADR Format (Architecture Decision Records):**
```
# ADR-NNN: <title>
## Status: Proposed | Accepted | Deprecated | Superseded
## Context
## Decision
## Consequences
## Alternatives considered
```

**Design Principles:**
1. Single responsibility — แต่ละ layer มีหน้าที่เดียว
2. Dependency inversion — use cases พึ่ง interfaces ไม่ใช่ implementations
3. Explicit over implicit — pass dependencies ผ่าน constructors
4. Fail fast — validate ที่ boundary (entity Validate())
5. Observability first — tracing, metrics, structured logs ขาดไม่ได้
6. One DB per domain — ไม่มี cross-domain DB joins

---

## 3. Commands — คำสั่งลัดสำหรับงานซ้ำๆ

### 3.1 Commands คืออะไร?

Commands คือ **ขั้นตอนที่กำหนดไว้ล่วงหน้า** ที่ AI จะทำตามทีละ step เมื่อผู้ใช้พิมพ์ `/command-name` เป็นเหมือน "recipe" สำเร็จรูป

**ตำแหน่งไฟล์:** `.claude/commands/<command-name>.md`

**วิธีใช้งาน:** พิมพ์ใน Claude Code session:
```
/new-feature widget
/add-usecase order CancelOrder
/add-error ErrStoreNotFound 404 store_not_found
```

`$ARGUMENTS` ใน command file จะถูกแทนที่ด้วยสิ่งที่พิมพ์หลังชื่อคำสั่ง

### 3.2 Command แต่ละตัว

#### `/new-feature <domain>` — สร้าง Feature ใหม่ End-to-End

**เป็นคำสั่งสำคัญที่สุด** scaffold ทุกอย่างตั้งแต่ entity ถึง HTTP handler

**9 ขั้นตอน:**

| Step | สิ่งที่สร้าง | ไฟล์ที่ได้ |
|------|------------|-----------|
| 1 | Entity struct + Validate() + test | `src/entity/<domain>/<domain>.go` + `_test.go` |
| 2 | Repository interface | เพิ่มใน `src/use_case/repository/repository.go` |
| 3 | Mock repository | `src/use_case/repository/mock_<domain>_repository.go` |
| 4 | GORM implementation | `src/repository/<domain>_repository/` (5 files) |
| 5 | Use case methods | `src/use_case/<domain>.go` |
| 6 | Wire to UseCase struct | แก้ `src/use_case/use_case.go` |
| 7 | Wire to main | แก้ `cmd/generics_server/` |
| 8 | HTTP handler (optional) | `src/interface/fiber_server/route/typespec/` |
| 9 | Domain errors | `src/use_case/model/errors.go` + `helper/errors.go` |

**Repository Implementation ประกอบด้วย 5 ไฟล์:**
1. `<domain>_repository.go` — tracer variable
2. `postgres_gorm_model.go` — GORM model + conversion functions
3. `postgres_gorm.go` — GORM implementation ทั้งหมด
4. `postgres_gorm_options.go` — filter/list/sort option handlers
5. `dummy.go` — no-op stub สำหรับ development ไม่ต้องมี DB

#### `/add-usecase <domain> <MethodName>` — เพิ่ม Use Case Method

**ใช้เมื่อ:** มี domain อยู่แล้ว ต้องการเพิ่ม business logic method ใหม่

**6 ขั้นตอน:**
1. อ่าน entity, repository interface, existing use case methods
2. สร้าง method ใหม่พร้อม tracing + permission check
3. เพิ่ม repository methods ถ้าจำเป็น (interface + mock + impl + dummy)
4. เพิ่ม domain errors ถ้าจำเป็น
5. เขียน tests (happy path, permission denied, validation error, repo error)
6. รัน `make unit-test` ต้อง pass

#### `/add-http-route <MethodName>` — Implement HTTP Handler

**ใช้เมื่อ:** TypeSpec generate route มาแล้ว แต่ handler ยังเป็น `panic("implement me")`

**Pattern ของ handler:**
```go
func (r routeDemoV1) HandlerName(c *fiber.Ctx, ...) error {
    id, err := helper.GetIdentityFromHeader(c)          // 1. Extract identity
    var req RequestType; c.BodyParser(&req)              // 2. Parse body
    result, err := r.useCase.Method(c.UserContext(), id) // 3. Call use case
    return c.JSON(ResponseType{...})                     // 4. Return response
}
```

**กฎเหล็ก:**
- ห้ามมี business logic ใน handler
- ใช้ `c.UserContext()` เสมอ ไม่ใช้ `context.Background()`
- ใช้ `helper.ErrorHandler(c, err)` สำหรับทุก error

#### `/add-error <ErrName> <status> <key>` — เพิ่ม Domain Error

**ใช้เมื่อ:** ต้องการ error ใหม่ที่ map กับ HTTP status code

**ตัวอย่าง:** `/add-error ErrStoreNotFound 404 store_not_found`

**สร้าง 2 ที่:**
1. `src/use_case/model/errors.go` → `var ErrStoreNotFound = errors.New("store not found")`
2. `src/interface/fiber_server/helper/errors.go` → `model.ErrStoreNotFound: {404, "store_not_found"}`

#### `/gen-http` — Regenerate HTTP Interface

**Pipeline:** TypeSpec → OpenAPI → Go Fiber interface

```bash
make gen-all
```

หลังรัน: เทียบ `route.go` กับ `spec.gen.go` → เพิ่ม/ลบ stubs

#### `/gen-grpc` — Regenerate gRPC Stubs

```bash
make generate-grpc
```

หลังรัน: เช็ค `*_grpc_server.go` ว่ายัง satisfy interface → เพิ่ม stub ถ้ามี RPC method ใหม่

#### `/unit-test` — รัน Unit Tests

```bash
make unit-test  # = go test -v ./src/...
```

รายงาน passed/failed per package → ถ้า fail ต้องหาสาเหตุและแก้

#### `/coverage` — เช็ค Coverage Threshold

```bash
make check-coverage
```

ถ้า < 70% → หา package ที่ coverage ต่ำสุด → เขียน tests เพิ่ม → รันใหม่จน pass

#### `/review-arch <path>` — ตรวจ Architecture Compliance

ตรวจ 5 หมวด:
1. **Dependency violations** — import ผิดทิศทาง
2. **Business logic in wrong layer** — logic อยู่ผิดที่
3. **Missing observability** — ไม่มี tracing
4. **Error handling** — error ไม่ wrap, ไม่ define sentinel
5. **Transaction safety** — ไม่มี defer rollback

**Output:** ไฟล์:บรรทัด + severity (CRITICAL/WARNING/SUGGESTION) + คำอธิบาย + วิธีแก้

---

## 4. Skills — ทักษะที่เรียกใช้ได้จากทุกบริบท

### 4.1 Skills คืออะไร? ต่างจาก Commands อย่างไร?

| | Commands | Skills |
|---|---------|--------|
| ตำแหน่ง | `.claude/commands/` | `.claude/skills/` |
| เรียกใช้ | พิมพ์ `/command-name` | Claude เรียกผ่าน Skill tool |
| มี description | ไม่มี (เป็นแค่ prompt) | มี YAML front matter พร้อม description |
| auto-detection | ไม่มี (ต้องเรียกเอง) | Claude สามารถจับได้จาก description |

### 4.2 โครงสร้าง Skill File

```yaml
---
description: Scaffold a complete new feature for the domain named $ARGUMENTS.
             Use when the user asks to add a new domain, entity, or feature.
---

[prompt เหมือน command file ทุกประการ]
```

**ความหมาย:** Skills มี `description` ช่วยให้ Claude **จับได้อัตโนมัติ** ว่าควรใช้ skill ไหน เช่น ถ้าผู้ใช้พิมพ์ "เพิ่ม domain widget ให้หน่อย" Claude จะรู้ว่าควรเรียก `new-feature` skill

### 4.3 เนื้อหาของ Skills

ทุก skill มีเนื้อหาเหมือน command คู่กันทุกประการ:

| Skill | Command คู่ | Description |
|-------|------------|-------------|
| `skills/new-feature.md` | `commands/new-feature.md` | Scaffold new feature end-to-end |
| `skills/add-usecase.md` | `commands/add-usecase.md` | Add new use case method |
| `skills/add-http-route.md` | `commands/add-http-route.md` | Implement HTTP handler |
| `skills/add-error.md` | `commands/add-error.md` | Add domain error + HTTP mapping |
| `skills/gen-http.md` | `commands/gen-http.md` | Regenerate HTTP interface |
| `skills/gen-grpc.md` | `commands/gen-grpc.md` | Regenerate gRPC stubs |
| `skills/unit-test.md` | `commands/unit-test.md` | Run unit tests |
| `skills/coverage.md` | `commands/coverage.md` | Check coverage threshold |
| `skills/review-arch.md` | `commands/review-arch.md` | Architecture compliance review |

**ทำไมต้องมีทั้ง Commands และ Skills?**
- **Commands** = user-initiated, พิมพ์ `/xxx` เรียกตรงๆ
- **Skills** = AI-initiated, Claude สามารถตัดสินใจเรียกเองได้จาก context

---

## 5. ความเชื่อมโยงทั้งระบบ

```
ผู้ใช้พิมพ์คำสั่ง
        │
        ▼
┌──────────────────┐
│  CLAUDE.md       │ ← โหลดทุก session เป็น context
│  (system prompt) │
└────────┬─────────┘
         │ อ้างอิง
         ▼
┌──────────────────┐     ┌──────────────────┐
│  rules/          │     │  settings.json   │
│  (auto-loaded)   │     │  (hooks config)  │
│  - architecture  │     └────────┬─────────┘
│  - conventions   │              │ trigger
│  - testing       │              ▼
│  - codegen       │     ┌──────────────────┐
│  - project       │     │  hooks/          │
│  - outline-colls │     │  env-protection  │
└──────────────────┘     └──────────────────┘

ผู้ใช้พิมพ์ /command หรือ AI เลือก skill
        │
        ▼
┌──────────────────┐     ┌──────────────────┐
│  commands/       │     │  skills/         │
│  (user-invoked)  │     │  (AI-invoked)    │
│  /new-feature    │     │  new-feature     │
│  /add-usecase    │     │  add-usecase     │
│  /add-http-route │     │  add-http-route  │
│  ...             │     │  ...             │
└──────────────────┘     └──────────────────┘

AI ถูกเรียกเป็น agent เฉพาะบทบาท
        │
        ▼
┌──────────────────┐
│  agents/         │
│  - developer     │ ← code, test, Jira, Outline
│  - po            │ ← backlog, stories, Outline
│  - qa            │ ← test, coverage, Jira
│  - sol-architect │ ← review, ADR, design
└────────┬─────────┘
         │ ใช้
         ▼
┌──────────────────────────────────┐
│  MCP Servers (External Tools)    │
│  - mcp__jira    (Jira Cloud)     │
│  - mcp__outline (Internal Wiki)  │
│  - mcp__gitlab  (Source Code)    │
└──────────────────────────────────┘
```

---

## สรุป Phase 2

| สิ่งที่เรียนรู้ | รายละเอียด |
|----------------|-----------|
| **Agents** | 4 บทบาท: Developer, PO, QA, Solution Architect |
| **Agent tools** | แต่ละ agent มี tools ต่างกันตามหน้าที่ |
| **Commands** | 9 คำสั่งลัด เรียกด้วย `/command-name` |
| **Skills** | 9 ทักษะ เนื้อหาเหมือน commands แต่ AI เรียกเองได้ |
| **Issue hierarchy** | 3 ระดับ: Epic → Story/Tech Story/Bug → Dev Task/QA Task |
| **Workflow** | ทุกงานเชื่อมกับ Jira (task tracking) + Outline (documentation) |

> **ต่อ Phase 3:** .opencode, MCP Servers, การ Clone Boilerplate, และ Best Practices
