---
name: wiki-lint
description: Health check for the research wiki. Finds orphan pages, missing cross-links, contradictions, concept gaps, stale content, and suggests new sources to find.
origin: user
---

# Wiki Lint Workflow

Vault root: `/Users/kimzey/Desktop/local-valut/`

## Steps

### 1. สำรวจ wiki ทั้งหมด
```
Read index.md
Glob wiki/**/*.md → list หน้าจริงทั้งหมด
เปรียบเทียบ → หาหน้าที่ไม่อยู่ใน index (missing entries)
```

### 2. ตรวจสอบ 7 จุด

#### A. Orphan Pages — หน้าที่ไม่มีใครลิงก์มาหา
Read ทุกหน้า → grep หา `[[filename]]` → หน้าไหนไม่ถูก mention เลย?

#### B. Broken Links — ลิงก์ที่ชี้ไปหน้าที่ไม่มีอยู่
Grep pattern `\[\[wiki/.*\]\]` ในทุกหน้า → เช็คว่าไฟล์ปลายทางมีอยู่จริง

#### C. Missing Concept Pages — Concepts ที่ถูก mention แต่ไม่มีหน้าของตัวเอง
Grep คำในหน้า sources/synthesis ที่น่าจะเป็น concept → เช็คว่ามีหน้าใน wiki/concepts/ ไหม

#### D. Contradictions — ข้อมูลขัดแย้งกัน
อ่าน concept pages ที่มีหลาย sources → หาคำที่บ่งบอกความขัดแย้ง เช่น "ตรงข้าม", "แย้ง", "ต่างจาก"

#### E. Stale Content — ข้อมูลที่อาจล้าสมัย
ดู frontmatter `updated:` → หน้าที่ไม่ได้อัปเดตนานกว่า 60 วัน + มี source ใหม่กว่า

#### F. Empty Sections — Sections ที่ว่างเปล่าในหน้าสำคัญ
Read concept pages → หา sections ที่มีแต่ heading ไม่มีเนื้อหา

#### G. Data Gaps — หัวข้อที่ควรค้นหาเพิ่มเติม
วิเคราะห์ concept pages → ถ้ามี concept ที่มี sources น้อยกว่า 2 → แนะนำสิ่งที่ควร ingest เพิ่ม

### 3. Output รายงาน

```markdown
# Wiki Health Report — [YYYY-MM-DD]

## สรุป
- Total pages: N
- Issues found: N

## A. Orphan Pages
- [ ] wiki/concepts/foo.md — ไม่มีใครลิงก์มา

## B. Broken Links
- [ ] wiki/sources/bar.md → [[wiki/concepts/missing]] ไม่มีไฟล์นี้

## C. Concept Pages ที่ยังขาด
- [ ] "X" — ถูก mention ใน 3 หน้าแต่ไม่มีหน้าของตัวเอง

## D. ข้อมูลขัดแย้ง
- [ ] wiki/concepts/baz.md — source A และ B ให้ข้อมูลต่างกัน

## E. เนื้อหาที่อาจล้าสมัย
- [ ] wiki/concepts/qux.md — ไม่ได้อัปเดต 90 วัน

## F. Sections ว่างเปล่า
- [ ] wiki/books/foo.md — section "คำวิจารณ์" ยังว่าง

## G. แนะนำ Sources ที่ควร ingest เพิ่ม
- "X topic" — มีแค่ 1 source ควรหาเพิ่มอีก 2-3 ชิ้น
```

### 4. Append log.md
```
## [YYYY-MM-DD] lint | พบ N issues
- Orphans: N, Broken links: N, Missing concepts: N
```

### 5. ถาม user
"ต้องการให้แก้ไข issues ไหนทันที?"
