# Learn Phase 1: ภาพรวมโครงสร้าง .claude และ .opencode

> เอกสารนี้อธิบายทุกสิ่งในโฟลเดอร์ `.claude/` และ `.opencode/` แบบ Deep Dive
> สำหรับนักพัฒนาที่ต้องการเข้าใจระบบ AI-assisted development ทั้งหมดของ boilerplate นี้

---

## 1. ภาพรวม: ทั้งสองโฟลเดอร์คืออะไร?

### `.claude/` — คอนฟิกสำหรับ Claude Code (Anthropic CLI)
เป็นระบบตั้งค่าที่ทำให้ Claude Code (AI CLI tool ของ Anthropic) เข้าใจโปรเจกต์นี้โดยอัตโนมัติ ทุกครั้งที่เปิด session ใหม่ Claude จะอ่านไฟล์เหล่านี้เป็นบริบทเริ่มต้น ทำให้ AI "รู้" กฎสถาปัตยกรรม, รูปแบบโค้ด, วิธีสร้าง feature ใหม่ และเครื่องมือทั้งหมดที่ใช้ได้

### `.opencode/` — คอนฟิกสำหรับ OpenCode (AI coding CLI ทางเลือก)
เป็นเครื่องมือคล้ายกันแต่ใช้ OpenCode CLI แทน Claude Code มีเอกสารบริบทและ MCP server เช่นกัน

### ความสัมพันธ์
ทั้งสองโฟลเดอร์ทำหน้าที่เดียวกัน = **ให้ AI เข้าใจโปรเจกต์แบบอัตโนมัติ** แต่สำหรับเครื่องมือคนละตัว:
- `.claude/` → สำหรับ Claude Code CLI
- `.opencode/` → สำหรับ OpenCode CLI

---

## 2. โครงสร้างไฟล์ทั้งหมด

```
.claude/
├── CLAUDE.md                          ← ไฟล์หลัก: โหลดทุกครั้งเป็น system prompt
├── settings.json                      ← ตั้งค่า hooks (ระดับ project, commit เข้า git)
├── settings.local.json                ← ตั้งค่าส่วนตัว (permissions, MCP servers, ไม่ commit)
├── hooks/
│   └── env-protection.sh              ← Hook ป้องกันไม่ให้ AI อ่านไฟล์ .env
├── rules/
│   ├── architecture.md                ← กฎ Clean Architecture 4 layer
│   ├── conventions.md                 ← Naming, error handling, tracing, transaction
│   ├── testing.md                     ← กฎการเขียนเทสต์, coverage threshold
│   ├── codegen.md                     ← TypeSpec → OpenAPI → Go pipeline
│   ├── project.md                     ← ข้อมูล Jira project (key, ID)
│   └── outline-collections.md         ← Outline workspace collection IDs
├── agents/
│   ├── developer.md                   ← Agent: นักพัฒนา Go
│   ├── po.md                          ← Agent: Product Owner
│   ├── qa.md                          ← Agent: QA Engineer
│   └── solution-architect.md          ← Agent: Solution Architect
├── commands/
│   ├── new-feature.md                 ← /new-feature: สร้าง feature ใหม่ end-to-end
│   ├── add-usecase.md                 ← /add-usecase: เพิ่ม use case method
│   ├── add-http-route.md              ← /add-http-route: implement HTTP handler
│   ├── add-error.md                   ← /add-error: เพิ่ม domain error + HTTP mapping
│   ├── gen-http.md                    ← /gen-http: regenerate HTTP interface
│   ├── gen-grpc.md                    ← /gen-grpc: regenerate gRPC stubs
│   ├── unit-test.md                   ← /unit-test: รัน unit tests
│   ├── coverage.md                    ← /coverage: เช็ค coverage threshold
│   └── review-arch.md                 ← /review-arch: ตรวจ architecture violations
└── skills/
    ├── new-feature.md                 ← Skill version ของ /new-feature
    ├── add-usecase.md                 ← Skill version ของ /add-usecase
    ├── add-http-route.md              ← Skill version ของ /add-http-route
    ├── add-error.md                   ← Skill version ของ /add-error
    ├── gen-http.md                    ← Skill version ของ /gen-http
    ├── gen-grpc.md                    ← Skill version ของ /gen-grpc
    ├── unit-test.md                   ← Skill version ของ /unit-test
    ├── coverage.md                    ← Skill version ของ /coverage
    └── review-arch.md                 ← Skill version ของ /review-arch

.opencode/
├── AGENTS.md                          ← ไฟล์หลัก: เทียบเท่า CLAUDE.md สำหรับ OpenCode
├── opencode.jsonc                     ← ตั้งค่า MCP servers สำหรับ OpenCode
└── plugin/
    └── env-protection.ts              ← Plugin ป้องกัน .env เหมือน hook ใน .claude
```

---

## 3. CLAUDE.md — หัวใจของระบบ

### ทำงานอย่างไร?
ไฟล์ `CLAUDE.md` จะถูก **โหลดอัตโนมัติทุก session** เป็น system prompt ของ Claude Code ทำหน้าที่เป็น "สมองเริ่มต้น" ที่ AI จะรู้ทุกอย่างเกี่ยวกับโปรเจกต์โดยไม่ต้องถาม

### เนื้อหาสำคัญที่อยู่ใน CLAUDE.md:

| หัวข้อ | ความหมาย |
|--------|----------|
| **Architecture layers** | แผนผัง 4 layer (Entity → UseCase → Repository → Interface) พร้อมกฎว่า layer ไหน import ได้จากไหน |
| **Key source files** | ตารางไฟล์สำคัญ เช่น `use_case.go`, `repository.go`, `errors.go`, `route.go` |
| **Code generation pipeline** | อธิบาย TypeSpec → OpenAPI → Go Fiber interface (`make gen-all`) |
| **Make targets** | คำสั่งทั้งหมด: `make run`, `make unit-test`, `make check-coverage` ฯลฯ |
| **Slash commands** | ลิสต์คำสั่งลัด `/new-feature`, `/add-usecase`, `/add-http-route` ฯลฯ |
| **Rules references** | ชี้ไปยังไฟล์ใน `.claude/rules/` ที่จะถูกโหลดเพิ่มเติม |

### เมื่อ clone boilerplate ต้องทำอะไร?
ต้องเปลี่ยน placeholder ทั้งหมด:
- `<service-name>` → ชื่อ service จริง เช่น `order-service`
- `<module-path>` → Go module path จริง เช่น `gitlab.sellsuki.com/sellsuki/team/backend/order-service`

---

## 4. Rules — กฎสถาปัตยกรรมที่ AI ต้องทำตาม

ไฟล์ใน `.claude/rules/` จะถูก **auto-loaded** เข้า context ของ Claude ทุกครั้ง ทำให้ AI รู้กฎโดยไม่ต้องบอกซ้ำ

### 4.1 `architecture.md` — กฎ Clean Architecture

**แก่นสำคัญ: Unidirectional Dependencies (ทิศทางเดียว)**

```
Interface → UseCase → Entity
Repository ─────────→ Entity
```

**กฎ "ห้ามทำ" (Forbidden imports):**

| ถ้าอยู่ใน Layer นี้ | ห้าม import จาก |
|---------------------|----------------|
| Entity | ทุก layer อื่น (ห้ามเลย) |
| UseCase | `src/interface/`, `src/repository/` |
| Interface | `src/repository/` (ห้าม import concrete repository) |
| Repository | `src/interface/`, `src/use_case/` (ยกเว้น interfaces) |

**หน้าที่แต่ละ layer:**
- **Entity** (`src/entity/<domain>/`): Pure domain, ไม่พึ่ง framework ใดๆ เก็บ structs, `Validate()`, state machine, sentinel errors
- **UseCase** (`src/use_case/`): Business logic เท่านั้น เรียก repository ผ่าน interface ห้ามเรียก DB หรือ API โดยตรง
- **Repository** (`src/repository/<domain>_repository/`): Data access ทั้งหมด implement interface จาก `src/use_case/repository/`
- **Interface** (`src/interface/`): Translation layer แปลง request/response ห้ามมี business logic

### 4.2 `conventions.md` — กฎการเขียนโค้ด

**Naming:**
- Package: `snake_case` เช่น `use_case`, `order_repository`
- Constructor: `New()` หรือ `NewXxx()` เช่น `NewGormPostgres(db)`
- GORM model: `postgresGorm<Entity>Model` (private struct)
- HTTP DTO: `<Entity>Request`, `<Entity>Response`

**Error Handling แบ่ง 3 ระดับ:**
1. **Domain error** = ประกาศใน `src/entity/<domain>/` → `var ErrInvalid<Domain>Data = errors.New(...)`
2. **Business error** = ประกาศใน `src/use_case/model/errors.go` → `var Err<Condition> = errors.New(...)`
3. **HTTP mapping** = ลงทะเบียนใน `src/interface/fiber_server/helper/errors.go` → map error → HTTP status code

**การเชื่อมต่อ:** Error ถูกสร้างใน entity/use_case → throw ขึ้นมาถึง interface layer → `helper.ErrorHandler(c, err)` จะ loop ใน `errorList` map เทียบด้วย `errors.Is()` แล้วตอบ HTTP status ที่ถูกต้อง

**Tracing (บังคับทุก use case method):**
```go
ctx, sp := tracer.Start(ctx, "use_case.MethodName")
defer sp.End()
// sp.AddEvent("...") ที่แต่ละ checkpoint
// sp.RecordError(err) ก่อน return error ทุกครั้ง
```

**Transaction Safety:**
```go
id, tx, err := uc.repo.CreateWithTransaction(ctx, entity)
committed := false
defer func() { if !committed { tx.Rollback() } }()
// ... operations ...
tx.Commit()
committed = true
```

### 4.3 `testing.md` — กฎการเทสต์

- **Coverage threshold: 70%** (CI gate = `make check-coverage`)
- **บังคับ Table-Driven Tests** ทุก test function
- **Mock injection** สำหรับ use case tests ใช้ mock จาก `src/use_case/repository/mock_<domain>_repository.go`
- **ลำดับความสำคัญ:** Entity tests > UseCase tests > Repository tests > Interface tests
- **ทุก use case method ต้องเทสต์ 4 branch:** happy path, permission denied, validation error, repository error

### 4.4 `codegen.md` — Code Generation Pipeline

```
TypeSpec (.tsp) → OpenAPI (openapi.yaml) → Go Fiber interface (spec.gen.go)
```

- **TypeSpec sources** อยู่ที่ `cmd/typespec_openapi_generator/v1/`
  - `model/<domain>.model.tsp` = โมเดล
  - `route/<domain>.route.tsp` = เส้นทาง API
  - `main.tsp` = service definition + imports
- **Generated files ห้ามแก้ไข:** `openapi.yaml`, `spec.gen.go`, `*.pb.go`, `*_grpc.pb.go`
- หลัง `make gen-all` ต้องเทียบ `route.go` กับ `ServerInterface` ใน `spec.gen.go` แล้วเพิ่ม/ลบ method ตามที่เปลี่ยน

### 4.5 `project.md` — ข้อมูลโปรเจกต์

เก็บข้อมูล Jira project:
- Project Name: LINE-R
- Project Key: `LR`
- Project ID: `10192`
- Cloud ID: `50dc7e4a-0539-4dcf-9c33-5481a395a5ab`

### 4.6 `outline-collections.md` — Outline Workspace IDs

เก็บ Collection ID ของ Outline (internal wiki) 20 collections สำหรับใช้กับ MCP tool `mcp__outline`

---

## 5. Settings — การตั้งค่า Claude Code

### 5.1 `settings.json` (commit เข้า git, ใช้ร่วมกันทั้งทีม)

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/env-protection.sh"
          }
        ]
      }
    ]
  }
}
```

**ความหมาย:**
- เป็น **Hook** ที่ทำงาน **ก่อน** Claude จะใช้เครื่องมือ `Read` (อ่านไฟล์)
- ทุกครั้งที่ Claude พยายามอ่านไฟล์ จะรัน `env-protection.sh` ก่อน
- ถ้าไฟล์เป็น `.env` หรือ `.env.*` → **บล็อก** (exit code 2) พร้อมข้อความ "Blocked: AI is not allowed to read .env files"
- **เหตุผล:** ป้องกัน AI อ่าน secrets, API keys, database credentials

### 5.2 `settings.local.json` (ไม่ commit, เฉพาะเครื่อง)

```json
{
  "permissions": {
    "allow": [
      "Bash(git -C ... rev-parse --show-toplevel)",
      "Bash(go vet:*)",
      "Bash(npx --yes tsc:*)",
      "Bash(make:*)",
      "Bash(node --check:*)"
    ]
  },
  "enabledMcpjsonServers": ["outline", "jira", "gitlab"]
}
```

**ความหมาย:**
- **permissions.allow** = คำสั่ง Bash ที่อนุญาตให้ AI รันโดยไม่ต้องถามยืนยันทุกครั้ง
  - `go vet:*` = ตรวจโค้ด Go
  - `npx --yes tsc:*` = TypeScript compiler
  - `make:*` = ทุก make target
  - `node --check:*` = ตรวจ syntax Node.js
- **enabledMcpjsonServers** = MCP servers ที่เปิดใช้: Outline (wiki), Jira (project management), GitLab (source code)

---

## 6. Hooks — ระบบ Trigger อัตโนมัติ

### 6.1 `env-protection.sh`

```bash
#!/bin/bash
input=$(cat)
file_path=$(echo "$input" | python3 -c "import sys,json; print(json.load(sys.stdin).get('file_path',''))")
basename=$(basename "$file_path")

if [[ "$basename" == ".env" || "$basename" == .env.* ]]; then
    echo "Blocked: AI is not allowed to read .env files" >&2
    exit 2
fi
```

**วิธีทำงาน:**
1. Claude Code ส่ง JSON ของ tool input เข้า stdin (เช่น `{"file_path": "/path/.env"}`)
2. Script ดึง `file_path` ออกมาด้วย python3
3. เช็คว่า basename เป็น `.env` หรือ `.env.*` หรือไม่
4. ถ้าใช่ → exit 2 = **บล็อก tool call** ทำให้ Claude อ่านไฟล์ไม่ได้
5. ถ้าไม่ → ไม่ทำอะไร = อนุญาต

**Hook Types ที่มีในระบบ:**
- `PreToolUse` = ก่อนใช้เครื่องมือ (ใช้ตรงนี้)
- `PostToolUse` = หลังใช้เครื่องมือ
- `Stop` = ตอน session จบ

---

## สรุป Phase 1

| สิ่งที่เรียนรู้ | รายละเอียด |
|----------------|-----------|
| `.claude/` | ระบบ config ให้ AI (Claude Code) เข้าใจโปรเจกต์ |
| `.opencode/` | ระบบ config เดียวกันสำหรับ OpenCode CLI |
| `CLAUDE.md` | System prompt หลัก โหลดทุก session |
| `rules/` | กฎสถาปัตยกรรม, coding conventions, testing, codegen |
| `settings.json` | Hooks configuration (ระดับ project) |
| `settings.local.json` | Permissions + MCP servers (ระดับเครื่อง) |
| `hooks/` | Script ที่รันอัตโนมัติก่อน/หลังเครื่องมือ AI |
| Security | ป้องกัน AI อ่าน .env ด้วย hook |

> **ต่อ Phase 2:** Agents, Commands, Skills — ระบบ AI ทำงานแทนบทบาทต่างๆ
