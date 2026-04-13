---
title: "Learn Phase 8: ระบบ AI สำหรับ Generate README.md แบบ Deterministic"
type: source
source_file: raw/notes/ai-context/LearnPhase8-README-Generator ระบบ AI สำหรับ Generate README.md แบบ Deterministic.md
url: ""
published: 2026-04-13
tags: [ai-generation, deterministic, readme, prompt-engineering, template]
related:
  - wiki/concepts/deterministic-ai-generation.md
  - wiki/concepts/claude-code-ai-cli.md
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/ai-context/LearnPhase8-README-Generator ระบบ AI สำหรับ Generate README.md แบบ Deterministic.md|Original file]]

## สรุป

Phase 8 อธิบายการออกแบบระบบ AI gen README.md ที่ได้ผลลัพธ์เหมือนกันทุกครั้ง (deterministic) ผ่านหลักการ Fixed Template + Data Extraction + Minimal AI Creativity พร้อม implementation เป็น Skill + Command

## ประเด็นสำคัญ

**หลักการ Deterministic AI Generation:**
```
Determinism = Fixed Template + Extracted Data + Minimal AI Creativity
```

1. **Fixed Structure** — กำหนด section ตายตัว ห้าม AI คิดโครงสร้างเอง
2. **Data Extraction** — ดึงข้อมูลจากโค้ดจริง (go.mod, src/, Makefile, docker-compose.yml)
3. **Template Filling** — AI มีหน้าที่แค่เติมข้อมูลลง template
4. **Constraint Prompting** — prompt สั่งเข้มงวด "Do NOT add or remove sections"

**Ratio Fixed vs Dynamic:**
| ส่วน | Fixed | Dynamic | AI Creative |
|---|---|---|---|
| Section headings | 100% | 0% | 0% |
| Boilerplate text | 100% | 0% | 0% |
| Module name, Go version | 0% | 100% | 0% |
| Directory listings | 0% | 100% | 0% |
| **รวม** | **~60%** | **~40%** | **0%** |

**AI Creative = 0%** คือกุญแจสำคัญ

## ข้อมูลที่น่าสนใจ

**Prompt Engineering Techniques ที่ใช้:**
1. **Constraint Prompting** — "Use EXACTLY the following structure. Do NOT add or remove sections."
2. **Data Extraction Before Generation** — แยก "หาข้อมูล" กับ "เขียน" ออกจากกัน → ลด hallucination
3. **FIXED Text Markers** — บอก AI ว่าข้อความไหนต้องคงที่ทุกครั้ง
4. **Explicit Conditional Logic** — "> Only include if `.proto` files exist." ไม่ปล่อยให้ AI ตัดสินใจเอง
5. **Verification Step** — ให้ AI ตรวจสอบตัวเองว่าทำตาม spec

**เมื่อใด README จะเปลี่ยน (ถูกต้อง):**
- เพิ่ม entity ใหม่ → entity list เปลี่ยน
- เพิ่ม .proto file → gRPC section ปรากฏ
- ลบ .gitlab-ci.yml → CI/CD section หายไป

**AI approach vs Pure Script:**
- AI (~99% deterministic): flexible, adapts to context, maintenance ต่ำ
- Pure script (100%): rigid, hardcoded, maintenance สูง
- **แนะนำ AI approach** เพราะ flexibility คุ้มค่ากว่า 1% ที่ต่างกัน

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/deterministic-ai-generation|Deterministic AI Generation]]
- [[wiki/concepts/claude-code-ai-cli|Claude Code AI CLI]]
- [[wiki/concepts/commands-and-skills|Commands and Skills]]
