---
name: wiki-ingest
description: Ingest a new source into the research wiki. Reads the source, extracts key concepts, creates/updates all relevant wiki pages, and updates index + log. Thai-primary output.
origin: user
---

# Wiki Ingest Workflow

Vault root: `/Users/kimzey/Desktop/local-valut/`

## Steps (ทำตามลำดับ ห้ามข้าม)

### 1. อ่าน schema

Read `CLAUDE.md` — ดูรูปแบบ frontmatter และ page templates ที่ถูกต้อง

### 2. อ่าน source

Read source file ทั้งหมด รวมถึงรูปภาพถ้ามี

### 4. สร้าง Source Summary Page

สร้าง `wiki/sources/[slug].md` ตาม template ใน CLAUDE.md

- slug = ชื่อไฟล์ต้นฉบับ หรือ title แปลงเป็น lowercase-hyphens
- ใส่ frontmatter ครบ (title, type, source_file, url, published, tags, related, created, updated)
- **จำเป็น**: ต้องมีลิงก์กลับไปที่ raw source file ในเนื้อหา เพื่อให้อ่านรายละเอียดเต็มได้
  - ใช้รูปแบบ: `**อ่านเต็ม**: [[../../raw/path/to/file.md|ไฟล์ต้นฉบับ]]`
  - หรือถ้าเป็น external URL: `**อ่านเต็ม**: [URL](url)`
  - วางไว้ด้านบนสุดของเนื้อหา หลังจาก frontmatter

### 5. อัปเดต Concept Pages

สำหรับแต่ละ concept ที่พบใน source:

```
Glob wiki/concepts/ → เช็คว่ามีหน้าอยู่แล้วไหม
  ถ้ามี  → Read หน้านั้น → เพิ่มข้อมูลใหม่, ระบุถ้าขัดแย้ง
  ถ้าไม่มี → สร้างหน้าใหม่ตาม template ใน CLAUDE.md
```

ทุก concept page ต้องมี:

- ลิงก์กลับไปที่ source: `[[wiki/sources/[slug]]]`
- ถ้าข้อมูลใหม่ขัดแย้งกับเดิม → เพิ่ม section "ข้อถกเถียง / มุมมองที่ต่างกัน" ห้ามลบของเดิม

### 6. อัปเดต Book Page (ถ้า source คือหนังสือหรือบท)

Read/Create `wiki/books/[slug].md` ตาม template ใน CLAUDE.md
เพิ่มสรุปบทใหม่ใต้ section "สรุปแต่ละบท"

### 7. อัปเดต index.md

Read `index.md` → เพิ่ม row ใหม่ใน table ที่เกี่ยวข้องทุกตาราง

### 8. Append log.md

```
## [YYYY-MM-DD] ingest | [source title]
- หน้าที่สร้างใหม่: [list]
- หน้าที่อัปเดต: [list]
- Concepts: [list]
```

### 9. รายงานผล

```
ingest เสร็จแล้ว
สร้างใหม่: N หน้า
อัปเดต: N หน้า
รวม: [list of all touched files]
```

## กฎห้ามทำ

- ห้ามลบข้อมูลเดิมออกจาก concept page — ให้ merge เสมอ
- ห้ามแก้ไขไฟล์ใน `raw/` — อ่านได้อย่างเดียว
- ห้ามสร้างหน้าโดยไม่มี frontmatter

## ข้อปฏิบัติจำเป็น

- **ทุก Source Summary Page ต้องมีลิงก์กลับไปที่ raw source**
  - ใช้ blockquote หรือ heading แรกหลัง frontmatter
  - รูปแบบ: `**อ่านเต็ม**: [[../../raw/path/to/file.md|ไฟล์ต้นฉบับ]]` หรือ `[ชื่อบทความ](url)`
  - ทำให้อ่านรายละเอียดเต็มได้โดยตรง
