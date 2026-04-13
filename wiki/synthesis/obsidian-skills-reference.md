---
title: "Obsidian & Claude Skills Reference"
type: synthesis
tags: [obsidian, claude-code, skills, reference, tools]
sources: []
related: []
created: 2026-04-14
updated: 2026-04-14
---

## คำถาม / โจทย์

Skill ชุดใหม่ที่ติดตั้งมา (`defuddle`, `json-canvas`, `obsidian-bases`, `obsidian-cli`, `obsidian-markdown`) แต่ละตัวทำอะไรได้บ้าง?

---

## สรุปคำตอบ

Skill ทั้ง 5 เป็น **ส่วนขยายของ Claude Code** ที่ให้ Claude เข้าใจและสร้าง/แก้ไขไฟล์รูปแบบต่างๆ ใน Obsidian ได้อย่างถูกต้อง รวมถึงดึงเนื้อหาจากเว็บมาในรูปแบบ Markdown ที่สะอาด

---

## การวิเคราะห์แต่ละ Skill

---

### 1. `defuddle` — ดึงเนื้อหาเว็บเป็น Markdown

**ใช้เมื่อ:** ผู้ใช้ให้ URL มาแล้วต้องการให้ Claude อ่านหรือวิเคราะห์เนื้อหานั้น

**ทำอะไรได้:**
- ดึงเนื้อหาจากหน้าเว็บ แล้วตัด navigation, โฆษณา, และ clutter ออก
- แปลงเป็น Markdown ที่อ่านได้และใช้ token น้อยกว่า WebFetch ปกติ
- ดึง metadata เฉพาะ เช่น `title`, `description`, `domain`

**คำสั่งหลัก:**

```bash
defuddle parse <url> --md              # แปลงเป็น Markdown
defuddle parse <url> --md -o file.md   # บันทึกลงไฟล์
defuddle parse <url> -p title          # ดึงเฉพาะ title
defuddle parse <url> --json            # ผลลัพธ์เป็น JSON
```

**หมายเหตุ:** ไม่ใช้กับ URL ที่ลงท้าย `.md` (ใช้ WebFetch โดยตรงแทน)

---

### 2. `json-canvas` — สร้างและแก้ไขไฟล์ Canvas

**ใช้เมื่อ:** ต้องการสร้าง mind map, flowchart, หรือ canvas ภาพใน Obsidian (ไฟล์ `.canvas`)

**ทำอะไรได้:**
- สร้าง/แก้ไขไฟล์ `.canvas` ตาม JSON Canvas Spec 1.0
- จัดการ **Nodes** 4 ประเภท:
  - `text` — กล่องข้อความ (รองรับ Markdown)
  - `file` — embed ไฟล์จาก vault
  - `link` — embed หน้าเว็บ
  - `group` — กลุ่มภาพ (container)
- สร้าง **Edges** (เส้นเชื่อม) พร้อม label, สีสัน, และลูกศรทิศทาง
- กำหนดสีด้วย preset `"1"`–`"6"` หรือ hex เช่น `#FF0000`

**โครงสร้างไฟล์พื้นฐาน:**

```json
{
  "nodes": [
    { "id": "abc123", "type": "text", "x": 0, "y": 0,
      "width": 400, "height": 200, "text": "# Hello" }
  ],
  "edges": [
    { "id": "edge01", "fromNode": "abc123", "toNode": "xyz456", "label": "leads to" }
  ]
}
```

**ข้อควรระวัง:** ต้องใช้ `\n` ใน JSON string สำหรับขึ้นบรรทัดใหม่ ห้ามใช้ `\\n`

---

### 3. `obsidian-bases` — สร้าง Database View สำหรับ Notes

**ใช้เมื่อ:** ต้องการสร้างตาราง/การ์ด/รายการ ที่ดึงข้อมูลจาก notes หลายๆ ไฟล์ใน vault (ไฟล์ `.base`)

**ทำอะไรได้:**
- สร้าง/แก้ไขไฟล์ `.base` (YAML format) ที่ทำงานเหมือน database view
- กรอง notes ด้วย **filters** (by tag, folder, property, date)
- คำนวณค่าด้วย **formulas** เช่น วันนับถึง deadline, สถานะ emoji
- แสดงผลใน **4 view types:**
  - `table` — ตาราง
  - `cards` — การ์ดภาพ (gallery)
  - `list` — รายการ
  - `map` — แผนที่ (ต้องมี lat/lng)
- ใส่ **summaries** เช่น Sum, Average, Count ที่ท้ายคอลัมน์

**ตัวอย่าง — Task Tracker:**

```yaml
filters:
  and:
    - file.hasTag("task")

formulas:
  days_until_due: 'if(due, (date(due) - today()).days, "")'
  priority_label: 'if(priority == 1, "🔴 High", "🟢 Low")'

views:
  - type: table
    name: "Active Tasks"
    filters:
      and:
        - 'status != "done"'
    order:
      - file.name
      - status
      - formula.priority_label
      - formula.days_until_due
```

**Properties ที่ใช้ได้:** `file.name`, `file.mtime`, `file.tags`, `file.folder` และ frontmatter ทั้งหมดของ note

---

### 4. `obsidian-cli` — ควบคุม Obsidian จาก Terminal

**ใช้เมื่อ:** ต้องการให้ Claude อ่าน/สร้าง/แก้ไข notes, ค้นหา vault, หรือ dev plugin โดยตรงผ่าน command line

**ต้องการ:** Obsidian เปิดอยู่และติดตั้ง `obsidian` CLI แล้ว

**ทำอะไรได้:**

| หมวด | ตัวอย่าง command |
|------|-----------------|
| อ่าน note | `obsidian read file="My Note"` |
| สร้าง note | `obsidian create name="New Note" content="# Hello"` |
| เพิ่มเนื้อหา | `obsidian append file="My Note" content="New line"` |
| ค้นหา | `obsidian search query="RAG" limit=10` |
| Daily note | `obsidian daily:append content="- [ ] Task"` |
| Set property | `obsidian property:set name="status" value="done" file="Note"` |
| ดู tasks | `obsidian tasks daily todo` |
| ดู tags | `obsidian tags sort=count counts` |
| Backlinks | `obsidian backlinks file="My Note"` |

**สำหรับ Plugin Development:**

```bash
obsidian plugin:reload id=my-plugin    # โหลด plugin ใหม่
obsidian dev:errors                    # ดู error log
obsidian dev:screenshot path=ss.png   # ถ่ายภาพหน้าจอ
obsidian dev:console level=error      # ดู console
obsidian eval code="app.vault.getFiles().length"  # run JS
```

**Options ทั่วไป:**
- `--copy` — copy output ไป clipboard
- `silent` — ไม่เปิดไฟล์ใน Obsidian
- `vault="Vault Name"` — เลือก vault ที่ต้องการ

---

### 5. `obsidian-markdown` — เขียน Obsidian Flavored Markdown

**ใช้เมื่อ:** ต้องการสร้างหรือแก้ไขไฟล์ `.md` ที่ใช้ syntax เฉพาะของ Obsidian

**ทำอะไรได้:**
- เขียน **Wikilinks** แบบ Obsidian (`[[Note Name]]`, `[[Note#Heading]]`)
- Embed เนื้อหาด้วย `![[...]]` สำหรับ note, รูปภาพ, PDF, audio, video
- สร้าง **Callouts** ด้วย syntax `> [!type]` มีกว่า 12 ประเภท เช่น `note`, `warning`, `tip`, `bug`, `danger`
- เขียน **Properties** (frontmatter YAML) พร้อม type ที่ถูกต้อง
- ใช้ **Tags** inline (`#tag`, `#nested/tag`)
- เขียน **Comments** ที่ซ่อนใน reading view (`%% ... %%`)
- Highlight ด้วย `==text==`
- สูตรคณิตศาสตร์ด้วย LaTeX (`$...$`, `$$...$$`)
- Diagram ด้วย Mermaid

**ตัวอย่าง Callout:**

```markdown
> [!warning] ข้อควรระวัง
> เนื้อหาสำคัญที่ต้องการเน้น

> [!faq]- คำถามที่พบบ่อย (ย่อได้)
> กดเพื่อขยาย
```

**ตัวอย่าง Embed:**

```markdown
![[Note Name]]              # embed ทั้ง note
![[Note Name#หัวข้อ]]       # embed เฉพาะหัวข้อ
![[image.png|400]]          # embed รูปกำหนดความกว้าง
![[doc.pdf#page=3]]         # embed PDF หน้าที่ 3
```

---

## สรุปเปรียบเทียบ

| Skill | ไฟล์ที่ทำงานด้วย | จุดเด่น |
|-------|-----------------|---------|
| `defuddle` | URL → `.md` | ดึงเว็บเป็น Markdown สะอาด |
| `json-canvas` | `.canvas` (JSON) | สร้าง visual canvas/mind map |
| `obsidian-bases` | `.base` (YAML) | Database view ของ notes |
| `obsidian-cli` | vault (ทุกไฟล์) | ควบคุม Obsidian จาก terminal |
| `obsidian-markdown` | `.md` | เขียน Obsidian syntax ได้ถูกต้อง |

---

## หัวข้อที่ควรศึกษาต่อ

- วิธีใช้ `obsidian-bases` ร่วมกับ wiki เพื่อสร้าง dashboard ของ sources/concepts ทั้งหมด
- วิธีสร้าง `.canvas` สำหรับ knowledge map ของหัวข้อ RAG
- Defuddle สามารถใช้ใน `/ingest` workflow เพื่อ clip เว็บโดยตรงได้
