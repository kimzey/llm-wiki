---
title: "Deterministic AI Generation — การสร้าง Output ที่สม่ำเสมอจาก AI"
type: concept
tags: [ai-generation, deterministic, prompt-engineering, template, hallucination-reduction]
sources:
  - ai-context-phase8.md
related:
  - wiki/concepts/claude-code-ai-cli.md
  - wiki/concepts/commands-and-skills.md
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

Deterministic AI Generation คือแนวทางลด creativity ของ AI ให้เป็นศูนย์ โดยให้ AI ทำหน้าที่แค่ **เติมข้อมูลลง template** แทนที่จะ "คิดเอง" ผลคือ output เหมือนกัน ~99% ทุกครั้งที่รัน (และ 100% ถ้า input ไม่เปลี่ยน)

## อธิบาย

**สูตรหลัก:**
```
Determinism = Fixed Template + Extracted Data + Minimal AI Creativity
```

**3 เสาหลัก:**

1. **Fixed Structure** — กำหนด section, ลำดับ, และ boilerplate text ตายตัวใน prompt ห้าม AI คิดโครงสร้างเอง

2. **Data Extraction** — ดึงข้อมูลจาก source of truth (code, config files) แทนที่จะให้ AI "เดา" หรือ "จำ"

3. **Constraint Prompting** — ใช้ภาษาคำสั่งชัดเจน:
   - "Use EXACTLY the following structure"
   - "Do NOT add or remove sections"
   - "FIXED text" markers
   - "> Only include if X exists" (explicit conditionals)

**Ratio ที่ดี:**
| ส่วน | Fixed | Dynamic (จากโค้ด) | AI Creative |
|---|---|---|---|
| โครงสร้าง/ลำดับ | 100% | 0% | 0% |
| Boilerplate text | 100% | 0% | 0% |
| ข้อมูลโปรเจกต์ | 0% | 100% | 0% |
| **รวม** | **~60%** | **~40%** | **0%** |

## ประเด็นสำคัญ

**Prompt Engineering Techniques:**

1. **Constraint Prompting** — บังคับ AI ไม่สร้างเนื้อหานอก template
2. **Extract-then-Generate** — แยกขั้นตอน "หาข้อมูล" กับ "เขียน" ออกจากกัน → ลด hallucination
3. **FIXED Text Markers** — บอก AI ว่าข้อความนี้ต้องคงที่ทุกครั้ง
4. **Explicit Conditional Logic** — ไม่ปล่อยให้ AI ตัดสินใจเอง → มี condition ชัดเจน
5. **Self-Verification Step** — ให้ AI ตรวจสอบตัวเองว่าทำตาม spec ก่อนจบ

**เมื่อใด output จะเปลี่ยน (อย่างถูกต้อง):**
- เมื่อ source data เปลี่ยน (เพิ่ม entity ใหม่, เปลี่ยน Go version ฯลฯ)
- ไม่ใช่เพราะ AI "คิดใหม่" ทุกครั้ง

**เปรียบเทียบกับ Pure Script:**
| | AI Approach | Pure Script |
|---|---|---|
| Deterministic | ~99% | 100% |
| Flexibility | สูง (เข้าใจ context) | ต่ำ |
| Maintenance | ต่ำ | สูง |
| Setup effort | ต่ำ (copy 2 files) | สูง (เขียน script) |

**แนะนำ AI approach** เพราะ 1% ที่ต่างกันจาก pure script คุ้มค่าด้วย flexibility ที่ได้มา

## ตัวอย่าง / กรณีศึกษา

**README Generator (`/gen-readme`):**
```markdown
## Instructions

### Step 1: Extract Project Data
Read: go.mod → module name, Go version
Read: ls src/entity/ → domain list
Read: ls src/repository/ → repository list
Read: Makefile → make targets

### Step 2: Generate README.md
Use EXACTLY the following structure. Do NOT add or remove sections.
Replace {{placeholders}} with extracted data.

# {{service_name}}
## Requirements
- Golang {{go_version}}+
...
```

**ผลลัพธ์:**
- รัน 3 ครั้งกับ code เดิม → ได้ README เหมือนกัน (sha256 identical)
- รันหลัง add entity ใหม่ → entity list เปลี่ยน, ส่วนอื่นเหมือนเดิม

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/commands-and-skills|Commands and Skills]] — implement เป็น Skill + Command คู่

## แหล่งที่มา

- [[wiki/sources/ai-context-phase8|Phase 8: README Generator]]
