# Phase 8: ระบบ AI สำหรับ Generate README.md แบบ Deterministic

> **เป้าหมาย:** สร้าง AI System ที่ gen README.md ได้ผลลัพธ์ใกล้เคียงกันทุกครั้ง ทุก project ที่ clone จาก boilerplate นี้

---

## 1. ปัญหาของ AI Gen README แบบปกติ

| ปัญหา                              | สาเหตุ                                                      |
| ---------------------------------- | ----------------------------------------------------------- |
| Gen แต่ละรอบได้เนื้อหาต่างกัน      | AI มี creativity/temperature ทำให้ output เปลี่ยนไปทุกครั้ง |
| README ไม่ consistent ข้ามโปรเจกต์ | ไม่มี template กลาง, แต่ละคน gen เอง                        |
| ข้อมูลผิดหรือไม่อัปเดต             | AI ไม่ได้อ่านโค้ดจริง เขียนจากจินตนาการ                     |
| ขาดหรือเกินจากที่ต้องการ           | ไม่มีข้อกำหนดว่าต้องมี section อะไรบ้าง                     |

## 2. หลักการออกแบบ: Deterministic README Generation

### แนวคิดหลัก

```
Determinism = Fixed Template + Extracted Data + Minimal AI Creativity
```

README ที่ deterministic ได้เพราะ:

1. **Fixed Structure** — section ทุกตัวถูกกำหนดไว้ตายตัว (ไม่ให้ AI คิดโครงสร้างเอง)
2. **Data Extraction** — ข้อมูลดึงจากโค้ดจริง (go.mod, src/, Makefile, docker-compose.yml)
3. **Template Filling** — AI มีหน้าที่เติมข้อมูลลง template ไม่ใช่สร้างเนื้อหาใหม่
4. **Constraint Prompting** — prompt สั่งอย่างเข้มงวด ห้ามเพิ่ม/ลด section

### Architecture

```
┌─────────────────────────────────────────────────────┐
│  Claude Code Skill/Command: /gen-readme             │
│                                                     │
│  1. Extract Data (deterministic)                    │
│     ├─ go.mod → module name                         │
│     ├─ src/entity/*/  → domain list                 │
│     ├─ src/repository/*/  → repository list         │
│     ├─ src/interface/*/  → adapter list             │
│     ├─ Makefile → make targets                      │
│     ├─ docker-compose.yml → infrastructure          │
│     ├─ cmd/*/  → entry points                       │
│     ├─ .claude/CLAUDE.md → slash commands           │
│     └─ deployment/*.yml → environments              │
│                                                     │
│  2. Fill Template (constrained)                     │
│     ├─ README.md (root)                             │
│     ├─ src/entity/README.md                         │
│     ├─ src/use_case/README.md                       │
│     ├─ src/repository/README.md                     │
│     ├─ src/interface/README.md                      │
│     └─ cmd/README.md                                │
│                                                     │
│  3. Write Files                                     │
└─────────────────────────────────────────────────────┘
```

---

## 3. README Template Specification

### 3.1 Root README.md — Required Sections (ห้ามเพิ่ม/ลด)

| # | Section | Data Source | หมายเหตุ |
|---|---------|-------------|---------|
| 1 | `# <service-name>` | go.mod module name (ตัวสุดท้าย) | ชื่อ service |
| 2 | `## Requirements` | go.mod Go version | คงที่: Go version |
| 3 | `## Private internal repository` | คงที่ (boilerplate text) | GOPROXY setup |
| 4 | `## Setup Instructions` | go.mod module path | clone + rename guide |
| 5 | `## AI Agent Setup` | .claude/, .opencode/ existence check | Claude Code + OpenCode setup |
| 6 | `## Dependency direction` | คงที่ + dependency_graph.svg existence | graph image |
| 7 | `## Project Structure` | `find src cmd -type d` | tree + description |
| 8 | `## Dependency Table` | คงที่ (Clean Architecture rules) | Entity/UseCase/Repo/Interface |
| 9 | `## Generate Interface` | คงที่ + TypeSpec/OpenAPI path check | TypeSpec + Fiber gen |
| 10 | `## gRPC server from Proto` | proto file existence check | ถ้ามี .proto |
| 11 | `## Start` | Makefile → `make run` | start command |
| 12 | `## Testing Guidelines` | scan src/**/*_test.go | test priority + examples |
| 13 | `## Unit Test / Coverage / Benchmark` | Makefile targets | test commands |
| 14 | `## Build Docker` | Dockerfile existence | docker commands |
| 15 | `## Enable CI/CD` | .gitlab-ci.yml existence | CI setup |

### 3.2 Layer README Template — Required Sections

ทุก layer README (`src/<layer>/README.md`) ต้องมี section เหล่านี้เท่านั้น:

```markdown
# <Layer Name>
(`<path pattern>`)

## Purpose
<1-2 paragraphs จาก template คงที่ของแต่ละ layer>

## Key Responsibilities
<bullet list ดึงจาก actual code: entities, interfaces, adapters>

## Design Philosophy
<1 paragraph คงที่ตาม Clean Architecture rules>

## Contribution Guidelines
<bullet list คงที่ตาม Clean Architecture rules>
```

---

## 4. Implementation: สร้าง Skill + Command

### 4.1 สร้างไฟล์ Command

**ไฟล์:** `.claude/commands/gen-readme.md`

```markdown
Generate deterministic README.md files for this project.

## Instructions

You MUST follow these rules EXACTLY:

### Step 1: Extract Project Data

Read the following files and extract data:

1. `go.mod` → extract: module path, Go version
2. `ls src/entity/` → list all domain directories
3. `ls src/repository/` → list all repository directories
4. `ls src/interface/` → list all adapter directories
5. `ls cmd/` → list all entry points
6. `Makefile` → extract all make targets with descriptions
7. `docker-compose.yml` → extract services (postgres, redis, kafka, etc.)
8. `ls deployment/` → list environment files
9. `ls src/**/*_test.go` → list test files per layer
10. Check existence of: `.claude/`, `.opencode/`, `.proto` files, `.gitlab-ci.yml`

### Step 2: Generate Root README.md

Use EXACTLY the following structure. Do NOT add or remove sections.
Replace `{{placeholders}}` with extracted data.

---

# {{service_name}}

## Requirements

- Golang {{go_version}}+

## Private internal repository

This project includes modules located in a private repository, accessible via VPN (production environment) at gitlab.sellsuki.com/sellsuki/sellsuki/backend/entity. To set access:

\```shell
# .zshrc or .bashrc
export GOPROXY=https://go.sellsuki.com
export GONOSUMDB=gitlab.sellsuki.com/*
\```

## Setup Instructions

1. Clone the repository:
    \```shell
    git clone git@gitlab.sellsuki.com:{{module_path}}.git
    cd {{service_name}}
    go mod download
    \```

2. Update the Package Name in `go.mod`:
    Replace `module {{current_module}}` with your project module path.
    Use IDE refactor to update all import paths.

## AI Agent Setup

> Only include this section if `.claude/` or `.opencode/` directories exist.

This repository is configured for {{ai_tools_list}}. After cloning, update context files.

### 1. Update project context
{{instructions per tool}}

### 2. Update Outline collections
{{instructions per tool}}

### 3. Configure MCP servers
{{instructions per tool}}

### 4. Available slash commands
{{table from .claude/CLAUDE.md or scan .claude/commands/}}

## Dependency direction

\```sh
make dependency-graph
\```

![Dependency Graph](./cmd/format_dependency_graph/dependency_graph.svg)

## Project Structure

\```
{{output of: find . -type d -not -path './.git/*' -not -path './vendor/*' formatted as tree}}
\```

{{description of each top-level directory — use FIXED text per layer}}

| Module | Can Depends On | Depended Upon |
|--------|---------------|---------------|
| Entity | - | UseCase, Repository, InterfaceAdapter |
| UseCase | Entity | InterfaceAdapter, Repository |
| Repository | Entity, UseCase | - |
| InterfaceAdapter | Entity, UseCase | Repository |

## Generate Interface

### Generate OpenAPI from TypeSpec

> Only include if `cmd/typespec_openapi_generator/` exists.

{{TypeSpec compile instructions — FIXED text}}

### Fiber server spec from OpenAPI

\```shell
make generate-interface-fiber
\```

## gRPC server from Proto

> Only include if `.proto` files exist.

{{gRPC generation instructions — FIXED text}}

## Start

\```shell
make run
\```

## Testing Guidelines

{{FIXED text about test characteristics}}

### Testing Priority

1. **Entity Tests**: {{list actual test files from src/entity/**/*_test.go}}
2. **Use Case Tests**: {{list actual test files from src/use_case/*_test.go}}
3. **Repository Tests**: {{list actual test files from src/repository/**/*_test.go}}
4. **Interface Adapter Tests**: {{list or "incoming..."}}

### Integration Tests

{{FIXED text about integration test approach}}

## Unit Test

\```shell
make unit-test
\```

## Run Coverage Test

\```shell
make coverage-test-html
\```

## Run Benchmark

\```shell
make benchmark-test
\```

## Build Docker

\```shell
docker build -t {{service_name}} .
docker run -p 8080:8080 --env-file ./.env --net host {{service_name}}
\```

## Enable CI/CD

> Only include if `.gitlab-ci.yml` exists.

{{FIXED CI instructions}}

---

### Step 3: Generate Layer READMEs

For each layer, use the FIXED template below. Only replace {{dynamic}} parts.

#### src/entity/README.md
- Purpose: FIXED text about entity layer (pure domain, no framework deps)
- Key Responsibilities: list actual domains from `ls src/entity/`
- Design Philosophy: FIXED text (simplicity, isolation, core business logic)
- Contribution Guidelines: FIXED text (business logic only, testing standards, isolation, data validation)

#### src/use_case/README.md
- Purpose: FIXED text about use case layer (business logic orchestrator)
- Key Responsibilities: extract actual use case method names from `src/use_case/*.go`
- Design Philosophy: FIXED text (oblivious to invocation source, clean separation)
- Common Use Cases: describe unexported helper methods if found

#### src/repository/README.md
- Purpose: FIXED text about repository layer (data access implementations)
- Package Conventions: FIXED text (_repository suffix convention)
- Directory Structure: list actual repos from `ls src/repository/`
- Contribution Guidelines: FIXED text (adherence to contracts, documentation, testing)

#### src/interface/README.md
- Purpose: FIXED text about interface adapter layer (translation only)
- Adapters: list actual adapters from `ls src/interface/`
- Rules: FIXED text (no business logic, basic validation only, dependency flow)

#### cmd/README.md
- Purpose: FIXED text about entry points
- Entry Points: list actual cmd directories from `ls cmd/`
- Docker: list Dockerfiles found in cmd/

### Step 4: Verify

After generating all files:
1. Confirm all sections match this spec exactly
2. Confirm no extra sections were added
3. Confirm all {{placeholders}} were replaced with real data
4. Confirm all conditional sections (gRPC, CI/CD) are included/excluded correctly

DO NOT add any sections, comments, or content not specified in this template.
```

### 4.2 สร้างไฟล์ Skill

**ไฟล์:** `.claude/skills/gen-readme.md`

```markdown
---
description: Generate deterministic README.md files for all project directories based on fixed templates and extracted code data
---

Generate deterministic README.md files for this project.

{{copy exact same content as the command file above}}
```

### 4.3 อัปเดต CLAUDE.md

เพิ่มใน table ของ Slash commands:

```markdown
| `/gen-readme` | Generate all README.md files deterministically |
```

---

## 5. ทำไมถึง Deterministic

### 5.1 Ratio ของ Fixed vs Dynamic content

| ส่วน | Fixed (คงที่) | Dynamic (ดึงจากโค้ด) | AI Creative |
|------|:---:|:---:|:---:|
| Section headings | 100% | 0% | 0% |
| Section order | 100% | 0% | 0% |
| Boilerplate text (setup, CI, testing philosophy) | 100% | 0% | 0% |
| Module name, Go version | 0% | 100% | 0% |
| Directory listings | 0% | 100% | 0% |
| Test file listings | 0% | 100% | 0% |
| Make targets | 0% | 100% | 0% |
| Infrastructure services | 0% | 100% | 0% |
| Conditional sections (gRPC, CI) | 0% | 100% | 0% |
| **รวม** | **~60%** | **~40%** | **0%** |

> **AI Creative = 0%** นี่คือกุญแจสำคัญ ไม่มีส่วนไหนที่ให้ AI "คิดเอง"

### 5.2 เปรียบเทียบ: ถ้า Gen 3 รอบ

```
Run 1: README.md (sha256: abc123...)
Run 2: README.md (sha256: abc123...)  ← identical
Run 3: README.md (sha256: abc123...)  ← identical
```

ถ้า code ไม่เปลี่ยน → README ไม่เปลี่ยน เพราะ:
- Template เดิม → โครงสร้างเดิม
- ข้อมูลจากโค้ดเดิม → เนื้อหาเดิม
- ไม่มี AI creativity → ไม่มีการสุ่ม

### 5.3 เมื่อไหร่ที่ README จะเปลี่ยน (ถูกต้อง)

| เมื่อ | README เปลี่ยนส่วน |
|-------|-------------------|
| เพิ่ม entity ใหม่ (`src/entity/payment/`) | entity list, test file list |
| เพิ่ม repository ใหม่ | repository list, directory structure |
| เปลัี่ยน Go version ใน go.mod | Requirements section |
| เพิ่ม make target ใหม่ | Make targets table |
| เพิ่ม .proto file | gRPC section appears |
| ลบ .gitlab-ci.yml | CI/CD section disappears |

---

## 6. Prompt Engineering Techniques ที่ใช้

### 6.1 Constraint Prompting

```
"Use EXACTLY the following structure. Do NOT add or remove sections."
```

บังคับให้ AI ไม่สร้างเนื้อหาใหม่ ทำตาม template เท่านั้น

### 6.2 Data Extraction Before Generation

```
"Read the following files and extract data FIRST, THEN fill template"
```

แยกขั้นตอน "หาข้อมูล" กับ "เขียน" ออกจากกัน ลด hallucination

### 6.3 FIXED Text Markers

```
"Purpose: FIXED text about entity layer"
```

บอก AI ว่าข้อความนี้ต้องเป็นเหมือนเดิมทุกครั้ง

### 6.4 Explicit Conditional Logic

```
"> Only include if `.proto` files exist."
```

ไม่ปล่อยให้ AI ตัดสินใจเอง — มี condition ชัดเจน

### 6.5 Verification Step

```
"Confirm all sections match this spec exactly"
```

ให้ AI ตรวจสอบตัวเองว่าทำตาม spec

---

## 7. วิธีใช้งาน

### 7.1 สร้างไฟล์ที่ต้องการ

```bash
# สร้าง command file
mkdir -p .claude/commands
# สร้าง skill file
mkdir -p .claude/skills
# สร้างตามข้อ 4.1 และ 4.2
```

### 7.2 รัน

```bash
# ใน Claude Code CLI
/gen-readme

# หรือพิมพ์
gen readme for this project
```

### 7.3 ตรวจสอบ

```bash
# ดู diff
git diff README.md
git diff src/*/README.md

# ถ้า OK
git add -A && git commit -m "docs: regenerate README files"
```

### 7.4 ใช้ข้ามโปรเจกต์

เมื่อ clone boilerplate ไปโปรเจกต์ใหม่:

```bash
# 1. Clone boilerplate
git clone ... my-new-service
cd my-new-service

# 2. Update go.mod, project context, etc.

# 3. Gen README — ได้ผลลัพธ์ตรงกับโครงสร้างจริงของ project ใหม่
/gen-readme
```

---

## 8. Advanced: เพิ่ม Hook สำหรับ Auto-Verify

สามารถเพิ่ม PostToolUse hook เพื่อ verify README หลัง gen:

**เพิ่มใน `.claude/settings.json`:**

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "if echo \"$TOOL_INPUT\" | grep -q 'README.md'; then echo 'README generated — verify sections match spec'; fi"
          }
        ]
      }
    ]
  }
}
```

---

## 9. Advanced: README Drift Detection (CI)

เพิ่มใน CI pipeline เพื่อตรวจว่า README ยังตรงกับโค้ดหรือไม่:

```yaml
# .gitlab-ci.yml
readme-check:
  stage: validate
  script:
    # Gen README ใน temp directory
    - claude --print "/gen-readme" > /tmp/generated-readme.md
    # Compare with current
    - diff README.md /tmp/generated-readme.md || echo "README is out of date"
  allow_failure: true
```

> หมายเหตุ: ต้องมี Claude Code CLI ใน CI runner หรือใช้ script แทน

---

## 10. Alternative: Pure Script Approach (ไม่ใช้ AI)

ถ้าต้องการ 100% deterministic โดยไม่พึ่ง AI เลย สามารถใช้ shell script:

```bash
#!/bin/bash
# scripts/gen-readme.sh

MODULE=$(head -1 go.mod | awk '{print $2}')
SERVICE_NAME=$(basename $MODULE)
GO_VERSION=$(grep '^go ' go.mod | awk '{print $2}')
ENTITIES=$(ls -d src/entity/*/ 2>/dev/null | xargs -I{} basename {})
REPOS=$(ls -d src/repository/*/ 2>/dev/null | xargs -I{} basename {})
INTERFACES=$(ls -d src/interface/*/ 2>/dev/null | xargs -I{} basename {})
CMDS=$(ls -d cmd/*/ 2>/dev/null | xargs -I{} basename {})

cat > README.md << HEREDOC
# ${SERVICE_NAME}

## Requirements

- Golang ${GO_VERSION}+

## Setup Instructions
...
HEREDOC
```

**เปรียบเทียบ:**

| | AI (Skill/Command) | Pure Script |
|--|:---:|:---:|
| Deterministic | ~99% (minor wording) | 100% |
| Flexibility | สูง (เข้าใจ context) | ต่ำ (hardcoded) |
| Maintenance | ต่ำ (AI adapt ได้) | สูง (ต้อง update script) |
| Cross-project | ใช้ได้เลย | ต้อง adjust path |
| Setup effort | copy 2 files | เขียน script ยาว |

**แนะนำ:** ใช้ AI approach (ข้อ 4) เป็นหลัก เพราะ ~99% deterministic แต่ flexible กว่ามาก

---

## 11. Checklist สำหรับ Implementation

- [ ] สร้าง `.claude/commands/gen-readme.md` ตาม Section 4.1
- [ ] สร้าง `.claude/skills/gen-readme.md` ตาม Section 4.2
- [ ] อัปเดต `.claude/CLAUDE.md` เพิ่ม `/gen-readme` ใน slash commands table
- [ ] ทดสอบ: รัน `/gen-readme` 3 รอบ เปรียบเทียบ output
- [ ] ทดสอบข้าม project: clone boilerplate ใหม่ รัน `/gen-readme`
- [ ] (Optional) เพิ่ม PostToolUse hook ตาม Section 8
- [ ] (Optional) เพิ่ม CI check ตาม Section 9
- [ ] Copy `.claude/commands/gen-readme.md` และ `.claude/skills/gen-readme.md` เข้า boilerplate

---

## 12. Summary

```
┌──────────────────────────────────────────────────┐
│  Deterministic README = Template + Data + Rules  │
│                                                  │
│  Template:  Fixed sections, fixed order          │
│  Data:      Extracted from actual source code    │
│  Rules:     "Do NOT add or remove sections"      │
│             "Use EXACTLY this structure"          │
│             "FIXED text" markers                 │
│                                                  │
│  Result:    ~99% identical across generations    │
│             Adapts to actual project structure    │
│             Works across all cloned projects     │
└──────────────────────────────────────────────────┘
```

**กุญแจสำคัญ:** ความ deterministic ไม่ได้มาจากการ "สั่ง AI ให้เขียนเหมือนเดิม" แต่มาจากการ **ลด AI creativity ให้เป็นศูนย์** และให้ AI ทำหน้าที่ **เติมข้อมูลลง template** เท่านั้น
