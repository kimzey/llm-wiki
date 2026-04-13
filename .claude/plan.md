# Plan: Integrate New Skills into Wiki System
**Date:** 2026-04-14  
**Status:** pending

Skills ที่ติดตั้งใหม่: `defuddle`, `json-canvas`, `obsidian-bases`, `obsidian-cli`, `obsidian-markdown`

---

## หลักการ

| New Skill | บทบาทใน workflow |
|-----------|-----------------|
| `defuddle` | อ่าน URL → Markdown สะอาด (แทน WebFetch) |
| `obsidian-markdown` | ทุก `.md` ที่สร้างใหม่ใช้ Obsidian syntax ถูกต้อง |
| `obsidian-cli` | search vault, verify ไฟล์, backlinks — ต้องเปิด Obsidian (fallback ถ้าไม่เปิด) |
| `json-canvas` | สร้าง visual knowledge map (`.canvas`) |
| `obsidian-bases` | สร้าง database view (`.base`) |

---

## ส่วนที่ 1 — Skills เก่าที่ต้องแก้ไข

### 1.1 `wiki-ingest/SKILL.md`
- [ ] เพิ่ม step: ใช้ **obsidian-markdown** skill เมื่อสร้าง/แก้ wiki pages (wikilinks, callouts, frontmatter types)
- [ ] เพิ่ม step: หลัง Write ไฟล์แล้ว ใช้ `obsidian search` ตรวจ concept ซ้ำ (ถ้า Obsidian เปิดอยู่)
- [ ] เพิ่ม step: `obsidian read file="slug"` verify ว่า Obsidian เห็นไฟล์ถูกต้อง (optional)
- [ ] เพิ่ม fallback note: ถ้า Obsidian ไม่เปิด → ใช้ Glob/Grep เดิม
- [ ] **Write tool ยังคงใช้สร้างไฟล์หลัก** (ไม่ใช้ obsidian create สำหรับ content ยาว)

### 1.2 `wiki-research/SKILL.md`
- [ ] Step 2 (Search): เปลี่ยนจาก WebFetch → `defuddle parse <url> --md` สำหรับอ่านหน้าเว็บ
- [ ] เพิ่มกฎ: URL ลงท้าย `.md` → ใช้ WebFetch โดยตรง (defuddle ไม่เหมาะ)

### 1.3 `wiki-query/SKILL.md`
- [ ] Step 1 (Read index): เพิ่ม option ใช้ `obsidian search query="..."` เสริม index.md (ถ้า Obsidian เปิดอยู่)
- [ ] Step 5 (Save synthesis): ระบุให้ใช้ **obsidian-markdown** skill เมื่อสร้าง synthesis page

### 1.4 `wiki-lint/SKILL.md`
- [ ] Step A (Orphan): เพิ่ม option ใช้ `obsidian backlinks file="..."` แทน grep (ถ้า Obsidian เปิดอยู่)
- [ ] เพิ่ม fallback: ถ้าไม่เปิด → grep `\[\[filename\]\]` เดิม

### 1.5 `wiki-status/SKILL.md`
- [ ] Step 5 (Dashboard): เพิ่ม optional section แสดง top tags ด้วย `obsidian tags sort=count counts`
- [ ] เพิ่ม fallback: ถ้า Obsidian ไม่เปิด → ข้าม tag section ไป

---

## ส่วนที่ 2 — Commands เก่าที่ต้องแก้ไข

### 2.1 `commands/ingest.md`
- [ ] เพิ่มบรรทัด: "ใช้ obsidian-markdown syntax เมื่อสร้างหน้า wiki"
- [ ] เพิ่มบรรทัด: verify ด้วย obsidian-cli (ถ้า Obsidian เปิดอยู่)

### 2.2 `commands/research.md`
- [ ] เปลี่ยน: ใช้ `defuddle` แทน WebFetch สำหรับ URL

### 2.3 `commands/query.md`
- [ ] เพิ่มบรรทัด: ใช้ `obsidian search` เสริม index.md (ถ้า Obsidian เปิดอยู่)
- [ ] เพิ่มบรรทัด: ใช้ obsidian-markdown เมื่อสร้าง synthesis page

### 2.4 `commands/synthesis.md`
- [ ] เพิ่มบรรทัด: ใช้ **obsidian-markdown** skill เมื่อสร้างหน้า

### 2.5 `commands/new-concept.md`
- [ ] เพิ่มบรรทัด: ใช้ **obsidian-markdown** skill เมื่อสร้างหน้า

### 2.6 `commands/new-book.md`
- [ ] เพิ่มบรรทัด: ใช้ **obsidian-markdown** skill เมื่อสร้างหน้า

### 2.7 `commands/update.md`
- [ ] เพิ่มบรรทัด: ใช้ **obsidian-markdown** skill เมื่อแก้ไขหน้า

### 2.8 `commands/init.md`
- [ ] Step 1 (directory structure): เพิ่ม skill directories ใหม่
  ```
  .claude/skills/defuddle/
  .claude/skills/json-canvas/
  .claude/skills/obsidian-bases/
  .claude/skills/obsidian-cli/
  .claude/skills/obsidian-markdown/
  ```
- [ ] Step 8 (skills stubs): เพิ่ม 5 skills ใหม่ในรายการ

---

## ส่วนที่ 3 — Commands ใหม่ที่ต้องสร้าง

### 3.1 `commands/clip.md` → `/clip [url]`
Workflow: `defuddle parse <url> --md` → บันทึกลง `raw/clips/[slug].md` → ถาม "ingest เลยไหม?"

- [ ] สร้างไฟล์ `commands/clip.md`
- [ ] Steps: defuddle → สร้างชื่อไฟล์จาก title → Write ลง raw/clips/ → offer `/ingest`

### 3.2 `commands/canvas.md` → `/canvas [topic]`
Workflow: อ่าน wiki concepts ที่เกี่ยวข้อง → สร้าง `.canvas` visual knowledge map

- [ ] สร้างไฟล์ `commands/canvas.md`
- [ ] Steps: อ่าน index + concepts → ใช้ json-canvas skill → บันทึก `wiki/canvas/[slug].canvas`

### 3.3 `commands/base.md` → `/base [name]`
Workflow: สร้าง `.base` ไฟล์ที่ดึง wiki pages มาแสดงเป็น table/cards view

- [ ] สร้างไฟล์ `commands/base.md`
- [ ] Presets: `sources` (table), `concepts` (cards), `books` (table), `all` (full dashboard)
- [ ] Steps: ใช้ obsidian-bases skill → บันทึก `wiki/bases/[name].base`

---

## ส่วนที่ 4 — Documentation

### 4.1 `.claude/README.md`
- [ ] เพิ่ม Skills table: 5 skills ใหม่
- [ ] เพิ่ม Commands section: `/clip`, `/canvas`, `/base`
- [ ] อัปเดต Recommended Workflows

### 4.2 `.claude/README-th.md`
- [ ] เหมือนกับ README.md ฉบับภาษาไทย

---

## ลำดับการทำ (Priority)

```
Priority 1 — กระทบ workflow หลักทันที
  1.1 wiki-ingest/SKILL.md
  1.2 wiki-research/SKILL.md
  2.1 commands/ingest.md
  2.2 commands/research.md

Priority 2 — เพิ่ม capability ใหม่
  1.3 wiki-query/SKILL.md
  2.3 commands/query.md
  3.1 commands/clip.md   ← new

Priority 3 — polish + เพิ่ม commands ใหม่
  1.4 wiki-lint/SKILL.md
  1.5 wiki-status/SKILL.md
  2.4–2.8 commands (synthesis, new-concept, new-book, update)
  3.2 commands/canvas.md  ← new
  3.3 commands/base.md    ← new

Priority 4 — documentation
  2.8 commands/init.md
  4.1 README.md
  4.2 README-th.md
```

---

## สถิติ

| ประเภท | จำนวน |
|--------|-------|
| Skills ที่ต้องแก้ | 5 |
| Commands ที่ต้องแก้ | 8 |
| Commands ใหม่ | 3 |
| README ที่ต้องแก้ | 2 |
| **รวม** | **18 items** |
