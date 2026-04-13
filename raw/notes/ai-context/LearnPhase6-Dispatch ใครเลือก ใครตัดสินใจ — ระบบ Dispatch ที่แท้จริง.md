# Learn Phase 6: ใครเลือก? ใครตัดสินใจ? — ระบบ Dispatch ที่แท้จริง test

> เอกสารนี้ตอบคำถามสำคัญ:
> "ถ้าสั่งทำงาน A แล้วจะมั่นใจได้ยังไงว่ามันจะใช้ Command, Agent, Skill ที่ถูกต้อง?"

---

## 1. คำตอบตรงๆ: ใช่ มันเลือกเอง — และนี่คือปัญหา

**ไม่มี mapping ใดๆ ระหว่าง Command ↔ Agent ↔ Skill**

```
❌ ไม่มีไฟล์ mapping.json ที่บอกว่า "/new-feature" → ใช้ Developer agent
❌ ไม่มี config ที่ผูก skill "add-usecase" → ต้องเรียก agent "developer"
❌ ไม่มี routing table, ไม่มี dispatch rule, ไม่มีอะไรเลย

✅ Claude ใช้ "ความเข้าใจภาษา" อ่าน description แล้วเลือกเอง
```

### สิ่งที่ hardcoded (แน่นอน 100%):

| สิ่งที่แน่นอน | กลไก |
|--------------|------|
| `/new-feature` → โหลด `commands/new-feature.md` | ชื่อไฟล์ตรงกับชื่อคำสั่ง |
| `Skill("new-feature")` → โหลด `skills/new-feature.md` | ชื่อไฟล์ตรงกับ skill name |
| `Task(subagent_type="Developer")` → โหลด `agents/developer.md` | name ใน YAML ตรงกับ subagent_type |
| `$ARGUMENTS` → ถูกแทนค่า | กลไก template ของ Claude Code |
| Agent tools list → จำกัดเครื่องมือ | YAML `tools:` บังคับ |

### สิ่งที่ AI เลือกเอง (ไม่แน่นอน):

| สิ่งที่ AI ตัดสินใจ | อาศัยอะไร |
|-------------------|----------|
| จะใช้ Skill ไหม? | อ่าน description ของ skill ทุกตัว เทียบกับคำขอ |
| จะ spawn Agent ไหม? | อ่าน description ของ agent เทียบกับความซับซ้อนของงาน |
| จะทำเองไหม? | ถ้างานง่ายพอ ไม่ต้องใช้ Skill หรือ Agent |

---

## 2. พิสูจน์: ดูจากกลไกจริง 3 เส้นทาง

### เส้นทาง A: ผู้ใช้พิมพ์ `/new-feature widget`

```
ผู้ใช้: /new-feature widget
         │
         ▼ (HARDCODED: ชื่อไฟล์ match)
commands/new-feature.md ถูกโหลด
$ARGUMENTS = "widget"
         │
         ▼ (INJECT เข้า main session เป็น prompt)
Claude ได้รับ prompt:
"Scaffold a complete new feature for the domain named: widget
 Follow Clean Architecture strictly..."
         │
         ▼ (AI ตัดสินใจ)
Claude ทำเองใน main session ← ไม่ได้เรียก Agent!
├── ใช้ Write tool สร้างไฟล์
├── ใช้ Edit tool แก้ไฟล์
├── ใช้ Bash tool รัน make unit-test
└── จบ

⚠️ ไม่มี Agent เข้ามาเกี่ยวข้อง!
⚠️ Command ทำงานใน main session โดยตรง!
```

### เส้นทาง B: ผู้ใช้พิมพ์ "เพิ่ม feature payment"

```
ผู้ใช้: "เพิ่ม feature payment"
         │
         ▼ (AI อ่าน skill descriptions ทั้งหมด)
Claude เห็น skills:
├── new-feature.md: "Scaffold a complete new feature..." ← ตรง!
├── add-usecase.md: "Add a new use case method..."
├── ...
         │
         ▼ (AI ตัดสินใจ: ใช้ skill new-feature)
เรียก Skill tool: skill="new-feature", args="payment"
         │
         ▼ (HARDCODED: ชื่อ match)
skills/new-feature.md ถูกโหลด
         │
         ▼ (INJECT เข้า main session)
Claude ทำเองใน main session ← ยังไม่มี Agent!
```

### เส้นทาง C: ผู้ใช้พิมพ์ "ให้ Developer agent สร้าง feature payment"

```
ผู้ใช้: "ให้ Developer agent สร้าง feature payment"
         │
         ▼ (AI ตัดสินใจ: ต้องใช้ Agent)
เรียก Task tool:
  subagent_type = "Developer"
  prompt = "สร้าง feature payment ตาม Clean Architecture..."
         │
         ▼ (HARDCODED: name match)
agents/developer.md ถูกโหลดเป็น system prompt
tools ถูกจำกัดตาม YAML: [Read, Edit, Write, Bash, ...]
         │
         ▼ (Agent ทำงานใน subprocess)
Developer agent ทำงานอิสระ
├── อ่าน rules/ (auto-loaded)
├── ทำตาม implementation checklist ใน agent file
└── return ผลลัพธ์
```

---

## 3. ปัญหาที่เกิดจากการ "ไม่ผูก"

### ปัญหา 1: Command ไม่รับประกันว่าจะใช้ Agent

```
/new-feature widget

คาดหวัง: Developer agent ทำงาน (มี Jira workflow, Outline docs)
จริง:    Claude ทำเองใน main session (ไม่มี Jira, ไม่มี Outline)

เหตุผล: Command ไม่ได้บอกให้ spawn Agent
```

### ปัญหา 2: AI อาจเลือก Skill ผิด

```
ผู้ใช้: "เพิ่ม error ใหม่สำหรับ payment not found"

AI อาจเลือก:
├── add-error skill    ← ถูก (เพิ่ม error sentinel + HTTP mapping)
├── new-feature skill  ← ผิด (scaffold ทั้ง domain)
└── ทำเอง             ← อาจไม่ครบ step
```

### ปัญหา 3: Agent ทำงานไม่ตรงกับ Skill

```
ถ้า Claude ตัดสินใจ spawn Developer agent
แต่ให้ prompt แค่ "สร้าง entity payment"

Developer agent จะทำตาม checklist ของตัวเอง (9 steps ครบ)
ไม่ใช่ตาม skill prompt (ที่อาจจะขอแค่ entity)

เหตุผล: Agent มี checklist ของตัวเอง ≠ Skill steps
```

---

## 4. วิธีแก้ — ทำให้มั่นใจว่า Dispatch ถูกต้อง

### วิธี 1: สั่งตรงๆ ด้วย `/command` (แน่นอนที่สุด)

```bash
# แน่นอน 100% ว่าจะทำตาม command steps
/new-feature payment
/add-usecase payment CreatePayment
/add-error ErrPaymentNotFound 404 payment_not_found
/unit-test
/coverage
/review-arch src/use_case/
```

**ข้อดี:** ขั้นตอนถูกกำหนดตายตัวใน .md file
**ข้อเสีย:** ไม่ได้ใช้ Agent (ไม่มี Jira workflow, ไม่มี Outline)

### วิธี 2: สั่งระบุ Agent ชัดเจน

```
"ให้ Developer agent ทำ /new-feature payment
 พร้อมอัพเดท Jira ticket LR-42 เป็น In Progress"
```

**ข้อดี:** ได้ Agent + Jira + Outline workflow
**ข้อเสีย:** ต้องพิมพ์เยอะ, ต้องรู้ว่า agent ไหนเหมาะ

### วิธี 3: เชื่อใจ AI แต่ตรวจสอบ

```
"เพิ่ม domain payment ทั้งหมด ตั้งแต่ entity จนถึง HTTP handler"

→ Claude จะเลือก skill/agent เอง
→ ดูว่าเลือกถูกไหม ถ้าไม่ถูกก็สั่งใหม่
```

### วิธี 4 (แนะนำ): เขียน Command ที่สั่ง Agent ให้ชัด

**นี่คือวิธี "ผูก" Command กับ Agent** — ไม่ใช่ผูกด้วย config แต่ผูกด้วย prompt:

สร้าง command ใหม่ `.claude/commands/dev-new-feature.md`:
```markdown
Scaffold a complete new feature using the Developer agent.

Domain: $ARGUMENTS

## Instructions

Use the Task tool to spawn a Developer agent with this prompt:

"Scaffold a complete new feature for the domain: $ARGUMENTS

Follow these steps in order:
1. Entity: src/entity/<domain>/ — struct + Validate() + test
2. Repository interface: src/use_case/repository/repository.go
3. Mock: src/use_case/repository/mock_<domain>_repository.go
4. GORM impl: src/repository/<domain>_repository/
5. UseCase: src/use_case/<domain>.go
6. Wire: src/use_case/use_case.go + cmd/generics_server/
7. HTTP handler: src/interface/fiber_server/route/typespec/
8. Domain errors: model/errors.go + helper/errors.go
9. Run make unit-test

After implementation:
- Update Jira ticket to In Progress
- Add implementation summary as Jira comment
- Update Outline documentation"

Use subagent_type: "Developer"
```

**ผลลัพธ์:** `/dev-new-feature payment` → **รับประกัน** ว่าจะ spawn Developer agent

---

## 5. ตารางสรุป: ระดับความมั่นใจ

| วิธีสั่ง | ขั้นตอนถูก? | Agent ถูก? | ความมั่นใจ |
|---------|-----------|-----------|-----------|
| `/new-feature widget` | 100% (hardcoded) | ไม่ใช้ agent | สูง (steps ตรง) |
| "เพิ่ม feature widget" | ~90% (AI เลือก skill) | ไม่ใช้ agent | ปานกลาง |
| "ให้ Dev agent ทำ /new-feature" | ~95% | 100% (สั่งตรง) | สูง |
| `/dev-new-feature widget` | 100% (hardcoded) | 100% (prompt บังคับ) | สูงมาก |
| ไม่สั่งอะไรเลย แค่บอก "ทำ payment" | ~70% (AI เดาเอง) | ~60% (อาจไม่ใช้) | ต่ำ |

---

## 6. Diagram: Decision Tree ของ Claude

```
ผู้ใช้ส่งข้อความ
│
├─ เริ่มด้วย / ?
│  ├── ใช่ → HARDCODED: โหลด commands/<name>.md
│  │         ├── แทนค่า $ARGUMENTS
│  │         └── Execute ใน main session (ไม่มี Agent)
│  │
│  └── ไม่ → AI Decision Layer
│           │
│           ├── 1) อ่าน skill descriptions ทั้ง 9 ตัว
│           │   ├── match? → เรียก Skill tool
│           │   └── ไม่ match → ข้อ 2
│           │
│           ├── 2) งานซับซ้อน? ต้องการ specific role?
│           │   ├── ใช่ → อ่าน agent descriptions ทั้ง 4 ตัว
│           │   │         ├── match Developer? → Task(subagent_type="Developer")
│           │   │         ├── match PO?        → Task(subagent_type="PO")
│           │   │         ├── match QA?        → Task(subagent_type="QA")
│           │   │         └── match Architect? → Task(subagent_type="Solution Architect")
│           │   └── ไม่ → ข้อ 3
│           │
│           └── 3) ทำเองใน main session
│                 └── ใช้ tools โดยตรง (Read, Edit, Bash, ...)
```

### ปัจจัยที่ AI ใช้ตัดสินใจ:

```
Skill matching อาศัย:
├── description field ใน YAML
│   "Scaffold a complete new feature..."  ← keyword matching
│   "Add a new use case method..."        ← keyword matching
└── ความตรงกับ user request

Agent matching อาศัย:
├── description field ใน YAML
│   "implementing features, writing Go code"  ← Developer
│   "managing Jira backlog, creating stories"  ← PO
│   "writing tests, checking coverage"         ← QA
│   "architectural decisions, design reviews"  ← Architect
├── ผู้ใช้ระบุชื่อ agent ตรงๆ หรือไม่
└── ความซับซ้อนของงาน (ง่าย → ทำเอง, ซับซ้อน → spawn agent)
```

---

## 7. ทำไมระบบถึงออกแบบมาแบบนี้?

### ข้อดีของ "ไม่ผูก":

| ข้อดี | อธิบาย |
|-------|--------|
| **ยืดหยุ่น** | เพิ่ม agent/skill ใหม่ได้ไม่ต้องแก้ mapping |
| **ไม่มี single point of failure** | ไม่มี routing config ที่พังแล้วทุกอย่างพัง |
| **Natural language interface** | สั่งภาษาธรรมชาติได้ ไม่ต้องจำ syntax |
| **Graceful degradation** | ถ้า AI เลือกผิด skill/agent ก็ยังทำงานได้ (แค่ไม่ optimal) |

### ข้อเสียของ "ไม่ผูก":

| ข้อเสีย | อธิบาย |
|---------|--------|
| **ไม่ deterministic** | สั่งเหมือนกัน 2 ครั้ง อาจได้ path ต่างกัน |
| **ต้องเขียน description ดี** | ถ้า description กำกวม AI เลือกผิด |
| **Command ไม่ได้ Agent workflow** | `/new-feature` ไม่ได้ Jira/Outline โดยอัตโนมัติ |
| **ต้องรู้ระบบ** | ผู้ใช้ต้องรู้ว่าเมื่อไหร่ควรสั่ง agent ตรง |

---

## 8. Best Practice: วิธีสั่งให้ได้ผลลัพธ์ที่ต้องการ

### ต้องการ "แค่โค้ด" (ไม่ต้อง Jira/Outline):
```
/new-feature payment         ← ใช้ Command ตรง
```

### ต้องการ "โค้ด + Jira + Outline" ครบ:
```
ให้ Developer agent scaffold feature payment ครบทุก layer
พร้อมอัพเดท Jira ticket LR-XX
```

### ต้องการ "Jira stories" อย่างเดียว:
```
ให้ PO agent สร้าง Epic [PAY] Payment สำหรับ LR project
พร้อม Tech Stories สำหรับ entity, repository, use case, HTTP handler
```

### ต้องการ "ตรวจ architecture":
```
/review-arch src/use_case/    ← Command: ตรวจแบบเร็ว
หรือ
ให้ Solution Architect agent review src/ ทั้งหมด
พร้อมสร้าง ADR ใน Outline       ← Agent: ตรวจแบบครบ + docs
```

### ต้องการ "เทสต์ + coverage":
```
/unit-test                    ← แค่รันเทสต์
/coverage                     ← แค่เช็ค coverage
หรือ
ให้ QA agent ตรวจ test coverage ของ src/use_case/
พร้อมเขียน test plan ใน Outline  ← Agent: ครบ workflow
```

---

## 9. สรุป

```
┌────────────────────────────────────────────────────────────────┐
│                                                                 │
│   Command, Skill, Agent ไม่ได้ "ผูกกัน" ด้วย config ใดๆ       │
│                                                                 │
│   แต่ทำงานร่วมกันผ่าน:                                         │
│   1. Shared context (CLAUDE.md + rules/)                       │
│   2. AI reasoning (อ่าน description แล้วเลือก)                 │
│   3. User instruction (สั่งตรงให้ใช้ agent/command)             │
│                                                                 │
│   วิธีมั่นใจที่สุด:                                              │
│   ├── สั่ง /command ตรง = ขั้นตอนแน่นอน, ไม่มี agent           │
│   ├── สั่งระบุ agent = ได้ agent ที่ต้องการ + workflow ครบ      │
│   └── เขียน command ที่สั่ง spawn agent = ได้ทั้งสองอย่าง       │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

| ถ้าต้องการ | ทำแบบนี้ |
|-----------|---------|
| ขั้นตอนแน่นอน 100% | พิมพ์ `/command` ตรงๆ |
| Agent ที่ถูกต้อง 100% | ระบุชื่อ agent ในคำสั่ง |
| ทั้งขั้นตอน + agent | เขียน command ที่ prompt ให้ spawn agent |
| ปล่อยให้ AI เลือก | ก็ได้ แต่ไม่ deterministic |
