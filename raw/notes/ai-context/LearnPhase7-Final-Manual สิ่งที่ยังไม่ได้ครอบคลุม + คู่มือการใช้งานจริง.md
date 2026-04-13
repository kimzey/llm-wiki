# Learn Phase 7: สิ่งที่ยังไม่ได้ครอบคลุม + คู่มือการใช้งานจริง

> เอกสารสุดท้าย ครอบคลุม:
> - Global Agents 10 ตัว (ที่ Phase 1-6 ยังไม่ได้พูดถึง)
> - Auto Memory System
> - Plan Mode
> - Context Window Management
> - Model Selection
> - Troubleshooting Guide
> - **คู่มือการใช้งานจริง** พร้อมตัวอย่างประโยคสั่งงาน

---

# ส่วน A: สิ่งที่ยังไม่ได้ครอบคลุม

## 1. Global Agents — 10 Agents ที่ใช้ได้กับทุกโปรเจกต์

Phase 2 พูดถึง 4 Project Agents (Developer, PO, QA, Solution Architect)
แต่ยังมี **10 Global Agents** อยู่ที่ `~/.claude/agents/` ที่ใช้ได้ด้วย:

| Agent | หน้าที่ | เรียกใช้เมื่อ | Tools |
|-------|--------|-------------|-------|
| **planner** | วางแผน implementation | feature ซับซ้อน, refactoring | Read, Grep, Glob (อ่านอย่างเดียว) |
| **architect** | ออกแบบ system | architectural decisions | Read, Grep, Glob (อ่านอย่างเดียว) |
| **code-reviewer** | review code quality | หลังเขียนโค้ดเสร็จ | Read, Grep, Glob, Bash |
| **tdd-guide** | บังคับ TDD workflow | feature ใหม่, bug fix | Read, Write, Edit, Bash, Grep |
| **security-reviewer** | หาช่องโหว่ security | user input, auth, API | Read, Write, Edit, Bash, Grep, Glob |
| **build-error-resolver** | แก้ build errors | build fail, type errors | Read, Write, Edit, Bash, Grep, Glob |
| **database-reviewer** | PostgreSQL specialist | SQL, migration, schema | Read, Write, Edit, Bash, Grep, Glob |
| **e2e-runner** | E2E testing | critical user flows | Read, Write, Edit, Bash, Grep, Glob |
| **refactor-cleaner** | ลบ dead code | code maintenance | Read, Write, Edit, Bash, Grep, Glob |
| **doc-updater** | อัพเดท docs | documentation outdated | Read, Write, Edit, Bash, Grep, Glob |

### Global vs Project Agents

```
Global Agents (10 ตัว) — ~/.claude/agents/
├── ใช้ได้กับทุกโปรเจกต์
├── เน้น generic tasks (review, test, build fix)
├── ไม่มี MCP tools (ไม่มี Jira/Outline)
└── ไม่มี project-specific knowledge

Project Agents (4 ตัว) — .claude/agents/
├── ใช้เฉพาะโปรเจกต์นี้
├── เน้น project-specific roles
├── มี MCP tools (Jira, Outline)
└── รู้ Clean Architecture, conventions, Jira project key
```

### "PROACTIVELY" — Agent ที่ Claude ควรเรียกเอง

Global rules (`~/.claude/rules/agents.md`) กำหนดว่า:
```
ไม่ต้องรอให้ user สั่ง — Claude ควรเรียกเองเมื่อ:
1. Feature ซับซ้อน    → planner agent
2. เขียนโค้ดเสร็จ     → code-reviewer agent
3. Feature ใหม่/bug fix → tdd-guide agent
4. Architectural decision → architect agent
```

นี่คือ **กฎใน global rules ที่บอกให้ Claude เรียก agent โดยไม่ต้องถาม**

---

## 2. Auto Memory System

Claude Code มี persistent memory ที่ `~/.claude/projects/<project-hash>/memory/`

```
สำหรับโปรเจกต์นี้:
~/.claude/projects/-Users-kimzey-Desktop-patona-boilerplate-backend-go/memory/

ปัจจุบัน: ว่างเปล่า (ยังไม่มี memory)
```

### Memory ทำงานอย่างไร:
- `MEMORY.md` จะถูกโหลดเข้า context ทุก session (เหมือน CLAUDE.md)
- Claude สามารถ Write/Edit ไฟล์ memory ได้ระหว่าง session
- Memory คงอยู่ข้าม sessions → Claude "จำ" สิ่งที่เรียนรู้

### ใช้ทำอะไร:
- จำ patterns ที่ค้นพบ
- จำ architectural decisions
- จำ user preferences
- จำ solutions ของ problems ที่เคยแก้

### ตัวอย่าง:
```
ผู้ใช้: "จำไว้ว่า project นี้ใช้ gofumpt แทน gofmt"

Claude จะเขียนลงไฟล์:
~/.claude/projects/.../memory/MEMORY.md:
- ใช้ gofumpt แทน gofmt สำหรับ formatting
```

---

## 3. Plan Mode

Plan Mode คือโหมดที่ Claude **วางแผนก่อนทำ** ไม่เขียนโค้ดจนกว่า user จะ approve

### วิธีเข้า Plan Mode:
```
Claude จะเข้า Plan Mode เอง เมื่อ:
- งานซับซ้อน (หลายไฟล์, หลาย layer)
- มีหลายวิธีทำ (ต้องเลือก approach)
- User สั่ง "วางแผนก่อน" หรือ "plan first"
```

### Flow:
```
1. Claude เข้า Plan Mode (EnterPlanMode)
2. สำรวจ codebase (Glob, Grep, Read)
3. เขียนแผนลงไฟล์ .claude/plans/<name>.md
4. ออก Plan Mode (ExitPlanMode) → user เห็นแผน
5. User approve / ให้แก้ / reject
6. Claude เริ่ม implement ตามแผน
```

### เมื่อไหร่ใช้ Plan Mode:
- `/new-feature` สำหรับ domain ที่ซับซ้อน
- Refactoring ที่กระทบหลาย layers
- Architectural changes

---

## 4. Context Window Management

### ปัญหา:
Claude มี context window จำกัด (~200k tokens) ถ้างานยาว → context เต็ม → ลืม

### กลไกที่มี:

**1. Auto-compact** — ระบบย่อ context อัตโนมัติเมื่อใกล้เต็ม
```
PreCompact hook → pre-compact.js → save important context
Auto-compact ทำงาน → ย่อข้อความเก่า
```

**2. suggest-compact hook** — แนะนำให้ compact เมื่อเหมาะสม
```
ทุกครั้งที่ Edit/Write → suggest-compact.js ตรวจ
ถ้า context ใกล้เต็ม → แนะนำให้ /compact
```

**3. Strategic compact** — compact ตาม logical phases
```
ไม่ใช่ compact ตอน context เต็ม (สายเกินไป)
แต่ compact ตอนจบแต่ละ phase ของงาน
```

### Best practice:
- งานยาว: แบ่งเป็น phases, compact ระหว่าง phase
- ใช้ Agent สำหรับงานย่อย: agent มี context แยก → ไม่กินของ main
- จำสิ่งสำคัญ: ให้ Claude เขียน memory ก่อน compact

---

## 5. Model Selection

```json
// ~/.claude/settings.json
"model": "opus"  ← default model
```

### Models ที่ใช้ได้:

| Model | ความสามารถ | ค่าใช้จ่าย | ใช้เมื่อ |
|-------|-----------|----------|---------|
| **opus** | สูงสุด (reasoning ลึก) | สูงสุด | งานซับซ้อน, architectural decisions |
| **sonnet** | สูง (coding ดีสุด) | ปานกลาง | งาน coding ทั่วไป |
| **haiku** | ดี (เร็วสุด) | ต่ำสุด | งานเล็ก, agent ที่เรียกบ่อย |

### Agent-level model override:
```yaml
# agents สามารถกำหนด model เฉพาะได้
---
name: planner
model: opus      ← ใช้ opus เสมอ (ต้องการ deep reasoning)
---
```

### ใช้ /model command:
```
/model sonnet     ← เปลี่ยนเป็น sonnet
/model opus       ← เปลี่ยนกลับ opus
```

---

# ส่วน B: คู่มือการใช้งานจริง (Practical Manual)

---

## Recipe 1: สร้าง Domain ใหม่ตั้งแต่ศูนย์

### สถานการณ์: ต้องเพิ่ม domain "payment" ทั้งหมด

**วิธี A: เร็วที่สุด (Command ตรง)**
```
/new-feature payment
```
- Claude จะสร้าง 8+ ไฟล์ตาม Clean Architecture
- จบด้วย `make unit-test`
- ไม่มี Jira/Outline

**วิธี B: ครบ workflow (ระบุ Agent)**
```
ให้ Developer agent scaffold domain payment ตั้งแต่ entity จนถึง HTTP handler
แล้วอัพเดท Jira ticket LR-15 เป็น In Progress
```
- ได้โค้ดครบ + Jira updated + Outline docs

**วิธี C: มีแผนก่อน (Plan first)**
```
วางแผนก่อนว่า payment domain จะมี fields อะไรบ้าง
API endpoints อะไรบ้าง error cases อะไรบ้าง
พอ approve แล้วค่อย implement
```
- Claude จะเข้า Plan Mode → เขียนแผน → รอ approve → implement

---

## Recipe 2: เพิ่ม Use Case Method ใหม่

### สถานการณ์: เพิ่ม method CancelOrder ใน order domain

**วิธี A: Command**
```
/add-usecase order CancelOrder
```

**วิธี B: ภาษาธรรมดา**
```
เพิ่ม method CancelOrder ใน order use case
ต้องมี permission check, validate, update status เป็น cancelled
ถ้ามี reservation ต้อง cancel ด้วย
```

**วิธี C: พร้อม TDD**
```
ให้ tdd-guide agent เขียนเทสต์ก่อนสำหรับ CancelOrder
แล้วค่อย implement ให้ pass
```

---

## Recipe 3: แก้ TypeSpec แล้ว Regenerate

### สถานการณ์: เพิ่ม endpoint ใหม่ใน TypeSpec

**ขั้นตอน:**
```
1. แก้ไฟล์ .tsp:
   "แก้ cmd/typespec_openapi_generator/v1/route/payment.route.tsp
    เพิ่ม POST /payment endpoint"

2. Regenerate:
   /gen-http

3. Implement handler:
   /add-http-route PaymentCreatePayment
```

---

## Recipe 4: ตรวจสอบคุณภาพโค้ด

### สถานการณ์: ก่อน push ต้องตรวจทุกอย่าง

**วิธี A: ทีละ command**
```
/unit-test                          ← เทสต์ pass?
/coverage                           ← coverage >= 70%?
/review-arch src/use_case/payment.go ← architecture ถูกต้อง?
```

**วิธี B: ใช้ Agents แบบ parallel** (เร็วกว่า)
```
ตรวจโค้ด payment domain ทั้งหมดก่อน push:
1. code-reviewer ตรวจ code quality
2. security-reviewer ตรวจ vulnerabilities
3. QA agent ตรวจ test coverage
รัน parallel ได้เลย
```

---

## Recipe 5: Debug เมื่อ Build Fail

### สถานการณ์: `make unit-test` fail

**วิธี A: ให้แก้ตรง**
```
/unit-test
```
- Claude จะรัน tests, เห็น error, แก้ให้, รันใหม่

**วิธี B: ใช้ build-error-resolver agent**
```
build fail ช่วยแก้ให้หน่อย
```
- Claude จะ spawn build-error-resolver agent
- agent จะ focus เฉพาะแก้ build error (minimal diff, ไม่ refactor)

---

## Recipe 6: สร้าง Jira Tickets

### สถานการณ์: ต้องสร้าง Epic + Stories สำหรับ feature ใหม่

```
ให้ PO agent สร้าง Epic [PAY] Payment Processing ใน Jira project LR
พร้อม Tech Stories:
1. Entity + validation
2. Repository interface + GORM implementation
3. Use case methods (CreatePayment, GetPayment, ListPayments)
4. HTTP handlers
5. Unit tests
```

- PO agent จะสร้าง Epic + 5 Tech Stories ใน Jira
- ทุก story มี acceptance criteria + DoD
- ตาม naming convention: `[PAY] As a developer, I want to...`

---

## Recipe 7: เขียน Tests จาก Acceptance Criteria

### สถานการณ์: มี Jira ticket พร้อม acceptance criteria ต้องเขียน tests

```
ให้ QA agent อ่าน Jira ticket LR-25
แล้วเขียน unit tests ตาม acceptance criteria
ใน src/use_case/payment_test.go
```

- QA agent จะอ่าน Jira → ดึง acceptance criteria → เขียน tests
- ทุก criteria กลายเป็น test case

---

## Recipe 8: Architecture Review ก่อน Merge

### สถานการณ์: ต้อง review PR ก่อน merge

**วิธี A: Command (เร็ว)**
```
/review-arch src/
```

**วิธี B: Agent (ครบ + ADR)**
```
ให้ Solution Architect agent review src/ ทั้งหมด
ตรวจ layer violations, missing tracing, error handling
ถ้ามี issue ให้สร้าง ADR ใน Outline
```

---

## Recipe 9: จัดการ Context เมื่องานยาว

### สถานการณ์: implement feature ใหญ่ หลาย files

```
Phase 1: Entity + Tests
→ /new-feature payment
→ "จำ memory ว่า payment entity มี fields: Id, Amount, Status, OrderId"
→ /compact

Phase 2: Repository
→ "ต่อจาก memory, implement payment repository"
→ /compact

Phase 3: Use Case
→ "ต่อจาก memory, implement payment use cases"
→ /compact

Phase 4: HTTP Handler
→ "ต่อจาก memory, implement payment HTTP handlers"
→ /unit-test
→ /coverage
```

---

## Recipe 10: วันแรกของทีม — Onboarding

### สถานการณ์: สมาชิกใหม่เข้าทีม ต้องเข้าใจโปรเจกต์

```
1. อ่าน CLAUDE.md เข้าใจ overview
2. docker compose up -d (start infrastructure)
3. make run (start server)
4. /review-arch src/ (ดู architecture compliance)
5. /unit-test (ดูว่า tests pass)
6. /coverage (ดู coverage level)

ลองสร้าง feature:
7. /new-feature demo_widget (ลองสร้าง demo domain)
8. /unit-test (ต้อง pass)
9. git checkout . (ลบ demo ทิ้ง)
```

---

# ส่วน C: Troubleshooting Guide

## ปัญหาที่พบบ่อย

### 1. "AI แก้ generated file"
```
อาการ: Claude แก้ spec.gen.go หรือ openapi.yaml
สาเหตุ: ไม่มี hook บล็อก generated files
แก้ไข: เพิ่ม PreToolUse hook (ดู Phase 5 วิธีที่ 7.3)
ป้องกัน: ย้ำใน prompt "ห้ามแก้ generated files"
```

### 2. "AI อ่าน .env"
```
อาการ: Claude พยายามอ่าน .env แต่ถูกบล็อก
สาเหตุ: Hook ทำงานถูกต้อง!
แก้ไข: ไม่ต้องแก้ — นี่คือ security ที่ออกแบบไว้
ถ้าต้องการให้ AI รู้ config: บอก values ในข้อความ (ไม่ใช่ให้อ่านไฟล์)
```

### 3. "Command ไม่ทำตาม steps"
```
อาการ: /new-feature ข้าม step หรือทำไม่ครบ
สาเหตุ: Context window เต็ม หรือ AI ตัดสินใจเอง
แก้ไข:
  1. /compact แล้วสั่งใหม่
  2. สั่งทีละ step: "ทำ step 1-3 ก่อน" → "ต่อ step 4-6"
  3. ใช้ Agent: "ให้ Developer agent ทำ /new-feature"
```

### 4. "Agent ไม่เห็น context ก่อนหน้า"
```
อาการ: spawn Developer agent แล้วมันถามว่า "domain อะไร?"
สาเหตุ: Agent อยู่ใน subprocess แยก ไม่เห็น conversation history
แก้ไข: ใส่ context ครบใน Task prompt:
  "Scaffold payment domain ที่มี fields: Id, Amount, Status, OrderId, CreatedAt
   Repository ต้องมี CreatePayment, GetPaymentById, ListPayments
   Use case ต้อง check permission PatonaPaymentCreate"
```

### 5. "make unit-test fail หลัง gen-all"
```
อาการ: generate ใหม่แล้ว compile error
สาเหตุ: route.go ไม่ match กับ spec.gen.go ใหม่
แก้ไข: /gen-http (command จะ reconcile route.go ให้)
หรือ: "เทียบ route.go กับ ServerInterface ใน spec.gen.go แล้วเพิ่ม/ลบ stubs"
```

### 6. "MCP server ไม่ทำงาน"
```
อาการ: Agent บอก "cannot use mcp__jira"
สาเหตุ:
  a) settings.local.json ไม่ได้เปิด server
  b) Agent ไม่มี mcp__jira ใน tools list
  c) MCP server URL ผิดหรือ down
แก้ไข:
  a) เพิ่ม "jira" ใน enabledMcpjsonServers
  b) เพิ่ม "mcp__jira" ใน agent's tools list
  c) ตรวจ URL ใน .mcp.json หรือ opencode.jsonc
```

### 7. "Claude ไม่เรียก Agent ที่ควรเรียก"
```
อาการ: สั่ง "review architecture" แต่ Claude ทำเองไม่ใช้ Architect agent
สาเหตุ: AI ตัดสินใจว่าไม่จำเป็น (ดู Phase 6)
แก้ไข: ระบุ agent ตรงๆ:
  "ให้ Solution Architect agent review architecture ของ src/"
```

---

# ส่วน D: Checklist ความเข้าใจทั้งหมด

## ทุกสิ่งที่ต้องรู้ — 7 Phases

| Phase | เนื้อหา | ถ้าเข้าใจแล้วจะทำได้ |
|-------|--------|-------------------|
| **1** | โครงสร้าง, CLAUDE.md, Rules, Settings, Hooks | ตั้งค่าโปรเจกต์ใหม่ |
| **2** | Agents 4 ตัว, Commands 9, Skills 9 | ใช้คำสั่งทำงานได้ |
| **3** | .opencode, MCP, Security, Clone | ตั้งค่า tools ภายนอก |
| **4** | Bindings — ใครผูกกับใคร | เข้าใจ architecture ทั้งระบบ |
| **5** | Cascade, Hooks, Source code map, Gotchas | แก้ปัญหาได้ |
| **6** | Dispatch — AI เลือกอย่างไร | สั่งงานได้ถูกต้อง |
| **7** | Global Agents, Memory, Plan Mode, Manual | ใช้งานได้อย่างเชี่ยวชาญ |

## "ผมรู้ครบหรือยัง?"

```
✅ ถ้าตอบได้ทุกข้อ = เข้าใจครบ

โครงสร้าง:
□ CLAUDE.md ทำอะไร? → system prompt โหลดทุก session
□ rules/ ทำอะไร? → กฎที่ auto-load เข้า context
□ settings.json vs settings.local.json? → project vs local, hooks vs permissions

Agents:
□ Project agents มีกี่ตัว? → 4 (Developer, PO, QA, Architect)
□ Global agents มีกี่ตัว? → 10 (planner, code-reviewer, tdd-guide, ...)
□ Agent tools ถูกจำกัดอย่างไร? → YAML tools list ใน front matter
□ Agent เห็น conversation history ไหม? → ไม่ (subprocess แยก)

Commands/Skills:
□ ต่างกันอย่างไร? → เนื้อหาเหมือน, Skill มี description ให้ AI detect
□ Command ผูกกับ Agent ไหม? → ไม่ ทำงานใน main session
□ จะให้ใช้ Agent ต้องทำอย่างไร? → ระบุ agent ชื่อตรงๆ ใน prompt

Dispatch:
□ AI เลือก skill อย่างไร? → อ่าน description เทียบกับ request
□ AI เลือก agent อย่างไร? → อ่าน description + ความซับซ้อน
□ จะมั่นใจได้อย่างไร? → สั่ง /command ตรง หรือ ระบุ agent

Security:
□ .env ถูกป้องกันอย่างไร? → PreToolUse hook → exit 2
□ Generated files ถูกป้องกันอย่างไร? → แค่ในเอกสาร (ไม่มี hook)
□ Permissions ตั้งที่ไหน? → settings.local.json

MCP:
□ MCP servers มีกี่ตัว? → 3 (Jira, Outline, GitLab)
□ Agent ใช้ MCP ได้เมื่อไหร่? → ต้องมี mcp__xxx ใน tools list

Workflow:
□ สร้าง feature ใหม่? → /new-feature <domain>
□ เพิ่ม use case? → /add-usecase <domain> <Method>
□ ตรวจคุณภาพ? → /unit-test + /coverage + /review-arch
□ Regenerate? → /gen-http หรือ /gen-grpc
```

---

## Quick Reference Card (ติดข้างจอ)

```
╔══════════════════════════════════════════════════╗
║           CLAUDE CODE QUICK REFERENCE            ║
╠══════════════════════════════════════════════════╣
║                                                  ║
║  COMMANDS (ขั้นตอนแน่นอน, ไม่มี Agent)          ║
║  /new-feature <domain>    สร้าง feature ครบ      ║
║  /add-usecase <d> <M>     เพิ่ม use case method  ║
║  /add-http-route <M>      implement HTTP handler ║
║  /add-error <E> <s> <k>   เพิ่ม domain error     ║
║  /gen-http                 regenerate HTTP        ║
║  /gen-grpc                 regenerate gRPC        ║
║  /unit-test                รัน tests              ║
║  /coverage                 เช็ค 70% threshold     ║
║  /review-arch <path>       ตรวจ architecture      ║
║                                                  ║
║  AGENTS (ระบุตรง = ได้ agent ที่ต้องการ)         ║
║  "ให้ Developer agent ..."    → เขียนโค้ด+Jira   ║
║  "ให้ PO agent ..."           → Jira tickets     ║
║  "ให้ QA agent ..."           → tests+coverage   ║
║  "ให้ Sol Architect agent ..." → review+ADR      ║
║  "ให้ code-reviewer ..."      → code quality     ║
║  "ให้ security-reviewer ..."  → vulnerabilities  ║
║  "ให้ tdd-guide ..."          → TDD workflow     ║
║  "ให้ planner ..."            → implementation   ║
║                                                  ║
║  UTILITIES                                       ║
║  /compact          ย่อ context                    ║
║  /model <name>     เปลี่ยน model                  ║
║  "จำไว้ว่า..."     บันทึก memory                   ║
║                                                  ║
║  RULES OF THUMB                                  ║
║  • Command = แน่นอน แต่ไม่มี Agent               ║
║  • ระบุ Agent = ได้ Agent แต่ต้องให้ context ครบ  ║
║  • ปล่อย AI = สะดวก แต่ไม่ deterministic         ║
║  • ห้ามแก้ spec.gen.go, openapi.yaml, *.pb.go    ║
║  • Coverage >= 70% เสมอ                          ║
║                                                  ║
╚══════════════════════════════════════════════════╝
```
