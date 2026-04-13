
test
> เอกสารนี้อธิบายความสัมพันธ์ระหว่าง Agent, Skill, Command, Rules, CLAUDE.md, Hooks
> และ MCP Servers อย่างละเอียด ว่าอะไรผูกกับอะไร ใครเรียกใคร ทำงานร่วมกันอย่างไร

---

## 1. คำตอบสั้น: Agent ไม่ผูกกับ Skill โดยตรง

**Agent, Skill, และ Command เป็น 3 ระบบอิสระต่อกัน** ไม่ได้ผูกกันแบบ 1:1

แต่ทั้ง 3 ระบบ **ใช้ context ร่วมกัน** ผ่าน Rules และ CLAUDE.md

```
┌──────────┐   ┌──────────┐   ┌──────────┐
│  Agent   │   │  Skill   │   │ Command  │
│ (บทบาท)  │   │ (ทักษะ)   │   │ (คำสั่ง)  │
└────┬─────┘   └────┬─────┘   └────┬─────┘
     │              │              │
     │   ไม่ผูกกัน    │   1:1 คู่แฝด   │
     │   โดยตรง      │              │
     └──────────┬───┴──────────────┘
                │
                ▼  ทุกตัวอ่าน context เดียวกัน
     ┌──────────────────────┐
     │  CLAUDE.md + rules/  │
     │  (shared context)    │
     └──────────────────────┘
```

---

## 2. ระบบ 3 อย่างคืออะไร — เปรียบเทียบจุดต่อจุด

| คุณสมบัติ | Agent | Skill | Command |
|-----------|-------|-------|---------|
| **ไฟล์อยู่ที่** | `.claude/agents/` | `.claude/skills/` | `.claude/commands/` |
| **เรียกใช้โดย** | `Task` tool (subagent_type) | `Skill` tool | ผู้ใช้พิมพ์ `/xxx` |
| **ใครเป็นคนเรียก** | Claude เรียกเอง หรือ ผู้ใช้สั่ง | Claude เรียกเอง (auto-detect) | ผู้ใช้เท่านั้น (manual) |
| **มี YAML front matter** | มี (name, description, tools) | มี (description เท่านั้น) | ไม่มี |
| **กำหนด tools ได้** | ได้ (tools: [...]) | ไม่ได้ | ไม่ได้ |
| **มี identity/persona** | มี (Developer, PO, QA, Architect) | ไม่มี | ไม่มี |
| **รับ arguments** | ผ่าน prompt ของ Task tool | `$ARGUMENTS` | `$ARGUMENTS` |
| **ทำงานแบบ** | subprocess (context แยก) | ใน main session | ใน main session |

---

## 3. การผูกกันแบบละเอียด — ใครรู้จักใคร?

### 3.1 Agent ไม่รู้จัก Skill/Command โดยตรง

ดูใน `developer.md`:
```yaml
---
name: Developer
tools:
  - Read
  - Edit
  - Write
  - Bash
  - mcp__jira
  - mcp__outline
---
```

**สังเกต:** ไม่มีบรรทัดไหนอ้างอิง skill หรือ command เลย

Agent รู้จักแค่:
- **tools** ที่กำหนดให้ (Read, Edit, Bash, mcp__jira ฯลฯ)
- **rules** ที่ auto-loaded (architecture.md, conventions.md ฯลฯ)
- **CLAUDE.md** ที่เป็น project context

### 3.2 Skill ไม่รู้จัก Agent โดยตรง

ดูใน `skills/new-feature.md`:
```yaml
---
description: Scaffold a complete new feature for the domain named $ARGUMENTS.
---
```

**สังเกต:** ไม่มี field ไหนชี้ไป agent

Skill รู้จักแค่:
- **description** ของตัวเอง (ใช้ให้ Claude auto-detect)
- **prompt** ข้างใน (ขั้นตอนที่ต้องทำ)

### 3.3 Command ไม่รู้จัก Agent หรือ Skill

ดูใน `commands/new-feature.md`:
```
Scaffold a complete new feature for the domain named: $ARGUMENTS
```

**สังเกต:** ไม่มี metadata, ไม่มี YAML, ไม่มีการอ้างอิง agent หรือ skill

### 3.4 แต่! Skill กับ Command เป็น "คู่แฝด"

ทุก Skill มี Command คู่กัน เนื้อหาเหมือนกันทุกตัวอักษร:

```
skills/new-feature.md     ←→  commands/new-feature.md      (เนื้อหาเหมือนกัน)
skills/add-usecase.md     ←→  commands/add-usecase.md      (เนื้อหาเหมือนกัน)
skills/add-http-route.md  ←→  commands/add-http-route.md   (เนื้อหาเหมือนกัน)
skills/add-error.md       ←→  commands/add-error.md        (เนื้อหาเหมือนกัน)
skills/gen-http.md        ←→  commands/gen-http.md         (เนื้อหาเหมือนกัน)
skills/gen-grpc.md        ←→  commands/gen-grpc.md         (เนื้อหาเหมือนกัน)
skills/unit-test.md       ←→  commands/unit-test.md        (เนื้อหาเหมือนกัน)
skills/coverage.md        ←→  commands/coverage.md         (เนื้อหาเหมือนกัน)
skills/review-arch.md     ←→  commands/review-arch.md      (เนื้อหาเหมือนกัน)
```

**ต่างกันแค่:**
- Skill มี `--- description: ... ---` (YAML front matter) → Claude auto-detect ได้
- Command ไม่มี → ต้องพิมพ์ `/xxx` เอง

**ทำไมต้องมีทั้งสอง?**
- **Command** = entry point สำหรับคน → พิมพ์ `/new-feature widget`
- **Skill** = entry point สำหรับ AI → Claude ตัดสินใจเรียกเองจาก description

---

## 4. สิ่งที่ "ผูกกันจริงๆ" — Shared Context

ระบบทั้งหมดผูกกันผ่าน **shared context layer** ไม่ใช่ผูกกันโดยตรง:

```
┌─────────────────────────────────────────────────────────┐
│                SHARED CONTEXT LAYER                      │
│                                                          │
│  ┌──────────────┐  ┌──────────────────────────────┐     │
│  │  CLAUDE.md   │  │  rules/ (auto-loaded)        │     │
│  │  (loaded     │  │  - architecture.md            │     │
│  │   every      │  │  - conventions.md             │     │
│  │   session)   │  │  - testing.md                 │     │
│  └──────┬───────┘  │  - codegen.md                 │     │
│         │          │  - project.md                  │     │
│         │          │  - outline-collections.md      │     │
│         │          └──────────────────────────────┘     │
│         │                                                │
│  ┌──────┴────────────────────────────────────────┐      │
│  │  settings.json → hooks/ (env-protection.sh)   │      │
│  │  settings.local.json → permissions + MCP       │      │
│  └───────────────────────────────────────────────┘      │
└──────────────────────┬──────────────────────────────────┘
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │  Agent   │ │  Skill   │ │ Command  │
   │          │ │          │ │          │
   │ ทำงานใน  │ │ ทำงานใน  │ │ ทำงานใน  │
   │ subprocess│ │ main     │ │ main     │
   │ (แยก     │ │ session  │ │ session  │
   │  context) │ │          │ │          │
   └──────────┘ └──────────┘ └──────────┘
```

### 4.1 อะไรที่ Agent ได้รับ (เมื่อถูกเรียกผ่าน Task tool)

```
Agent ได้รับ:
├── CLAUDE.md content (project context)
├── rules/* ทุกไฟล์ (auto-loaded)
├── Agent's own .md file content (system prompt ของตัวเอง)
│   ├── name, description
│   ├── tools list → กำหนดว่าใช้ tools อะไรได้
│   └── role identity + workflow instructions
├── settings.json → hooks ทำงาน
└── Task prompt → คำสั่งจาก caller
```

**Agent ไม่ได้รับ:**
- Skills files
- Commands files
- ไฟล์ agent ตัวอื่น

### 4.2 อะไรที่ Skill ได้รับ (เมื่อถูกเรียกผ่าน Skill tool)

```
Skill ได้รับ:
├── Current conversation context (main session)
├── CLAUDE.md content (already loaded)
├── rules/* (already loaded)
├── Skill's own .md file content (prompt)
│   └── $ARGUMENTS ถูกแทนค่าแล้ว
└── settings.json → hooks ทำงาน
```

**Skill ทำงานใน main session** = ได้ tools ทั้งหมดที่ main session มี

### 4.3 อะไรที่ Command ได้รับ (เมื่อผู้ใช้พิมพ์ `/xxx`)

```
Command ได้รับ:
├── Current conversation context (main session)
├── CLAUDE.md content (already loaded)
├── rules/* (already loaded)
├── Command's .md file content (prompt)
│   └── $ARGUMENTS ถูกแทนค่าแล้ว
└── settings.json → hooks ทำงาน
```

**Command = Skill ในแง่ execution** → ต่างกันแค่ใครเป็นคนเรียก

---

## 5. Flow การทำงานจริง — ตัวอย่างสถานการณ์

### สถานการณ์ 1: ผู้ใช้พิมพ์ `/new-feature widget`

```
ผู้ใช้: /new-feature widget
         │
         ▼
Claude Code ค้นหาใน .claude/commands/
         │
         ▼
พบ commands/new-feature.md
         │
         ▼
แทนค่า $ARGUMENTS = "widget"
         │
         ▼
Inject prompt เข้า main session:
"Scaffold a complete new feature for the domain named: widget
 Follow Clean Architecture strictly..."
         │
         ▼
Claude (main session) ทำตาม prompt ทีละ step
├── Step 1: สร้าง entity (ใช้ Write tool)
├── Step 2: เพิ่ม repository interface (ใช้ Edit tool)
├── Step 3: สร้าง mock (ใช้ Write tool)
├── ...
└── Step 9: make unit-test (ใช้ Bash tool)
```

**ใช้ Agent ไหม?** ไม่ ทำใน main session

---

### สถานการณ์ 2: ผู้ใช้พิมพ์ "เพิ่ม domain payment ให้หน่อย"

```
ผู้ใช้: "เพิ่ม domain payment ให้หน่อย"
         │
         ▼
Claude อ่าน skill descriptions ทั้งหมด:
├── new-feature.md: "Scaffold a complete new feature..."  ← ตรง!
├── add-usecase.md: "Add a new use case method..."
├── add-http-route.md: "Implement a new HTTP route..."
├── ...
         │
         ▼
Claude เรียก Skill tool:
  skill: "new-feature", args: "payment"
         │
         ▼
skills/new-feature.md ถูกโหลด
$ARGUMENTS = "payment"
         │
         ▼
Claude ทำตาม prompt (เหมือน command ทุกประการ)
```

**ใช้ Agent ไหม?** ไม่ ทำใน main session

---

### สถานการณ์ 3: ผู้ใช้สั่ง "ให้ Developer agent implement order feature"

```
ผู้ใช้: "ให้ Developer agent implement order feature"
         │
         ▼
Claude เรียก Task tool:
  subagent_type: "Developer"
  prompt: "Implement order feature end-to-end"
         │
         ▼
Claude Code สร้าง subprocess ใหม่
         │
         ▼
Subprocess ได้รับ:
├── agents/developer.md content (system prompt)
│   ├── identity: Senior Go Developer
│   ├── tools: [Read, Edit, Write, Bash, mcp__jira, mcp__outline]
│   └── implementation checklist
├── CLAUDE.md content
├── rules/* (auto-loaded)
└── Task prompt: "Implement order feature end-to-end"
         │
         ▼
Developer agent ทำงานอิสระ:
├── อ่าน existing code
├── สร้าง entity, repository, use case
├── รัน make unit-test
├── อัพเดท Jira ticket (via mcp__jira)
└── return ผลลัพธ์กลับ main session
```

**ใช้ Skill/Command ไหม?** ไม่โดยตรง
Agent รู้ขั้นตอนจาก checklist ใน agent file เอง ไม่ได้เรียก skill

---

### สถานการณ์ 4: Main session เลือกใช้ Agent เพื่อทำ Skill

```
ผู้ใช้: "review architecture ของ src/use_case/"
         │
         ▼
Claude ตัดสินใจ: งานนี้ใช้ Solution Architect agent เหมาะที่สุด
         │
         ▼
Claude เรียก Task tool:
  subagent_type: "Solution Architect"
  prompt: "Review the code at src/use_case/ for Clean Architecture compliance.
           Check dependency violations, missing tracing, error handling,
           transaction safety..."
         │
         ▼
Architect agent ทำงาน + return ผลลัพธ์
```

**สังเกต:** Claude ไม่ได้เรียก `review-arch` skill
แต่ **คัดลอกเนื้อหาจาก skill/command ใส่ใน Task prompt เอง**

นี่คือวิธีที่ Agent + Skill/Command **เชื่อมกันโดยอ้อม** ผ่าน Claude ที่เป็นตัวกลาง

---

## 6. Binding Map — ใครผูกกับใคร

### 6.1 Direct Bindings (ผูกตรง)

```
settings.json ──────────► hooks/env-protection.sh
   │                         │
   └─ matcher: "Read"        └─ ทำงานก่อนทุก Read tool call
                                (ทุก Agent, Skill, Command ที่ใช้ Read)

settings.local.json ────► MCP Servers (jira, outline, gitlab)
   │                         │
   └─ enabledMcpjsonServers  └─ Agent ที่มี mcp__xxx ใน tools list
                                จะใช้ได้ก็ต่อเมื่อเปิดที่นี่

CLAUDE.md ──────────────► rules/* (referenced, auto-loaded)

Skill file ─────────────► Command file (1:1 content duplicate)
```

### 6.2 Indirect Bindings (ผูกอ้อมผ่าน Claude)

```
Agent ←─── Claude (ตัวกลาง) ───► Skill
  │              │                  │
  │   Claude อ่าน Skill description │
  │   แล้วตัดสินใจว่า               │
  │   - ทำเองใน main session (Skill)│
  │   - หรือส่งให้ Agent ทำ (Task)  │
  │                                 │
  └─── ไม่รู้จักกันโดยตรง ──────────┘
```

### 6.3 Context Inheritance (สืบทอด context)

```
                    CLAUDE.md + rules/*
                         │
            ┌────────────┼────────────┐
            │            │            │
            ▼            ▼            ▼
         Agent        Skill        Command
            │            │            │
            │     ┌──────┴──────┐     │
            │     │ main session│     │
            │     │  context    │     │
            │     └─────────────┘     │
            │                         │
     ┌──────┴──────┐                  │
     │ subprocess  │    ไม่แชร์ context กัน
     │  context    │  (agent ไม่เห็น conversation history)
     └─────────────┘
```

---

## 7. Tools Binding — เครื่องมือผูกกับ Agent อย่างไร

### 7.1 Agent มี tools จำกัด

แต่ละ agent กำหนด tools ไว้ชัดเจนใน YAML front matter:

| Agent | Read | Edit | Write | Bash | Glob | Grep | mcp__jira | mcp__outline |
|-------|------|------|-------|------|------|------|-----------|-------------|
| **Developer** | Y | Y | Y | Y | Y | Y | Y | Y |
| **PO** | Y | - | - | - | Y | - | Y | Y |
| **QA** | Y | - | - | Y | Y | Y | Y | Y |
| **Sol. Architect** | Y | - | - | Y | Y | Y | Y | Y |

**หมายเหตุ:**
- **PO ไม่มี Edit, Write, Bash** → ไม่สามารถแก้โค้ดหรือรันคำสั่งได้ = ป้องกันไม่ให้ PO เปลี่ยนโค้ด
- **QA ไม่มี Edit, Write** → อ่านโค้ดและรัน tests ได้ แต่ไม่แก้ production code
- **Developer มีทุกอย่าง** → ทำได้หมด

### 7.2 Skill/Command ไม่จำกัด tools

Skill และ Command ทำงานใน **main session** → ใช้ tools ทุกตัวที่ main session มี:
- Read, Edit, Write, Bash, Glob, Grep
- MCP tools (jira, outline, gitlab) ถ้าเปิดใน settings.local.json
- Task tool (เรียก Agent ได้!)

**ดังนั้น Skill/Command สามารถเรียก Agent ได้** (ถ้า Claude ตัดสินใจ):

```
ผู้ใช้: /new-feature widget
         │
         ▼
Command prompt loaded into main session
         │
         ▼
Claude ตัดสินใจ: "งานนี้ซับซ้อน ให้ Developer agent ทำ"
         │
         ▼
Claude เรียก Task tool:
  subagent_type: "Developer"
  prompt: "Scaffold widget feature following these steps: ..."
```

**แต่ Agent ไม่สามารถเรียก Skill/Command ได้** เพราะ Agent ไม่มี Skill tool

---

## 8. MCP Server Binding — ใครใช้ MCP ได้

```
settings.local.json
└── enabledMcpjsonServers: ["outline", "jira", "gitlab"]
         │
         ├──► Main session: ใช้ได้ทุก MCP server
         │
         └──► Agent subprocess:
              ├── Developer: mcp__jira + mcp__outline (ระบุใน tools)
              ├── PO:        mcp__jira + mcp__outline (ระบุใน tools)
              ├── QA:        mcp__jira + mcp__outline (ระบุใน tools)
              └── Architect:  mcp__jira + mcp__outline (ระบุใน tools)
```

**หมายเหตุ:** ถึง `mcp__gitlab` จะเปิดใน settings.local.json แต่ไม่มี agent ไหนระบุ `mcp__gitlab` ใน tools list → **Agent ใช้ GitLab MCP ไม่ได้** ต้องทำผ่าน main session เท่านั้น

---

## 9. Hook Binding — Hook ทำงานกับทุกอย่าง

```
settings.json
└── hooks.PreToolUse
    └── matcher: "Read" → env-protection.sh
         │
         ├── Main session ใช้ Read → hook ทำงาน
         ├── Developer agent ใช้ Read → hook ทำงาน
         ├── QA agent ใช้ Read → hook ทำงาน
         ├── Architect agent ใช้ Read → hook ทำงาน
         └── Skill/Command ใช้ Read → hook ทำงาน
```

**Hook ทำงานกับทุก context** ไม่ว่าจะเรียกจาก agent, skill, หรือ command = **global security layer**

---

## 10. สรุปรวม: Binding Diagram

```
                    ┌─────────────────────┐
                    │    ผู้ใช้ (Human)     │
                    └──────┬──────────────┘
                           │
              ┌────────────┼────────────────┐
              │            │                │
         พิมพ์ /xxx    ถาม/สั่งธรรมดา    สั่งใช้ agent
              │            │                │
              ▼            ▼                ▼
        ┌──────────┐ ┌──────────┐  ┌──────────────┐
        │ Command  │ │  Claude  │  │  Task tool   │
        │ (.md)    │ │ (brain)  │  │ (subprocess) │
        └────┬─────┘ └────┬─────┘  └──────┬───────┘
             │            │               │
             │     detect skill?          │
             │      ┌─────┴────┐          │
             │      ▼          │          ▼
             │ ┌──────────┐    │   ┌──────────────┐
             │ │  Skill   │    │   │  Agent (.md) │
             │ │ (.md)    │    │   │ - identity   │
             │ └────┬─────┘    │   │ - tools list │
             │      │          │   └──────┬───────┘
             └──────┼──────────┘          │
                    │                     │
          ┌─────────┴─────────────┐       │
          │    Main Session       │       │
          │    (full tools)       │       │
          └─────────┬─────────────┘       │
                    │                     │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Shared Context     │
                    │  CLAUDE.md          │
                    │  rules/* (6 files)  │
                    │  hooks (security)   │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  External Services  │
                    │  MCP: Jira          │
                    │  MCP: Outline       │
                    │  MCP: GitLab        │
                    └─────────────────────┘
```

---

## 11. FAQ

### Q: Agent เรียก Skill ได้ไหม?
**A: ไม่ได้** Agent ไม่มี Skill tool ใน tools list

### Q: Skill เรียก Agent ได้ไหม?
**A: ได้** Skill ทำงานใน main session ซึ่งมี Task tool → สามารถ spawn agent ได้

### Q: Command กับ Skill ต่างกันอย่างไร?
**A: เนื้อหาเหมือนกัน** ต่างกันที่ Skill มี description → Claude auto-detect ได้ Command ต้องผู้ใช้พิมพ์ `/xxx`

### Q: ทำไมต้องมีทั้ง Skill และ Command?
**A:** เพื่อรองรับ 2 entry points:
- คนเรียกตรงๆ → Command
- AI ตัดสินใจเอง → Skill

### Q: Hook ทำงานกับ Agent ที่อยู่คนละ subprocess ด้วยไหม?
**A: ใช่** Hook เป็น global ทำงานทุก tool call ไม่ว่า context ไหน

### Q: ถ้าเพิ่ม command ใหม่ ต้องเพิ่ม skill ด้วยไหม?
**A: แนะนำให้เพิ่มทั้งคู่** ถ้ามีแค่ command → ผู้ใช้ต้องจำชื่อพิมพ์เอง ถ้ามี skill ด้วย → Claude จะ auto-suggest/invoke เมื่อเหมาะสม

### Q: ถ้าเพิ่ม agent ใหม่ ต้องทำอะไรบ้าง?
**A:** สร้างไฟล์ใน `.claude/agents/` โดยมี:
1. YAML front matter: name, description, tools list
2. Markdown body: identity, context, workflow, checklist

ไม่ต้องแก้ไฟล์อื่น Claude จะรู้จัก agent ใหม่อัตโนมัติจาก description

### Q: PO Agent สร้าง Jira ticket แล้ว Developer Agent เห็นไหม?
**A: เห็น** เพราะทั้งสองเรียก `mcp__jira` ซึ่งเชื่อมกับ Jira Cloud ตัวเดียวกัน ข้อมูลอยู่บน Jira ไม่ใช่ใน session

---

## 12. สรุป

| ความสัมพันธ์ | ลักษณะ |
|-------------|--------|
| Agent ↔ Skill | **ไม่ผูกตรง** แต่ Claude เป็นตัวกลางเชื่อม |
| Skill ↔ Command | **ผูกแบบ 1:1 คู่แฝด** เนื้อหาเหมือนกัน ต่างที่ trigger |
| Agent ↔ Rules | **ผูกผ่าน auto-load** ทุก agent ได้ rules เป็น context |
| Skill/Command ↔ Rules | **ผูกผ่าน main session** ที่มี rules loaded อยู่แล้ว |
| Hook ↔ ทุกอย่าง | **ผูก global** ทำงานกับทุก tool call ทุก context |
| MCP ↔ Agent | **ผูกผ่าน tools list** agent ต้องระบุ mcp__xxx จึงจะใช้ได้ |
| MCP ↔ Skill/Command | **ใช้ได้ทุกตัว** เพราะอยู่ใน main session ที่เปิด MCP ไว้แล้ว |
