---
name: wiki-query
description: Answer research questions by searching the wiki. Reads index, drills into relevant pages, synthesizes Thai-primary answers with citations, and optionally files the answer as a synthesis page.
origin: user
---

# Wiki Query Workflow

Vault root: `/Users/kimzey/Desktop/local-valut/`

## Steps

### 1. อ่าน index.md
Read `index.md` — หา pages ที่น่าจะเกี่ยวข้องกับคำถาม

### 2. อ่าน pages ที่เกี่ยวข้อง
Read ทุกหน้าที่ระบุใน index แล้วดูว่า related links พาไปหาหน้าอื่นอีกไหม
ลำดับความสำคัญ: concepts > synthesis > books > sources

### 3. ตอบคำถาม
- ตอบเป็นภาษาไทยเป็นหลัก
- cite ทุก claim ด้วย `[[wiki/concepts/foo]]` หรือ `(source: wiki/sources/bar.md)`
- ถ้าไม่มีข้อมูลใน wiki → บอกตรงๆ ว่า "ยังไม่มีข้อมูลใน wiki — ต้องการ ingest source เพิ่มไหม?"

### 4. รูปแบบ output ตามประเภทคำถาม
| ประเภทคำถาม | รูปแบบ |
|------------|--------|
| อธิบาย concept | prose + bullet points |
| เปรียบเทียบ | markdown table |
| สรุปหนังสือ | structured summary |
| วิเคราะห์เชิงลึก | essay + synthesis page |
| หา pattern | list + examples |

### 5. ถามว่าจะบันทึกไหม
ทุกคำตอบที่ไม่ใช่ข้อมูลพื้นฐาน ให้ถาม:
"ต้องการบันทึกคำตอบนี้เป็นหน้า synthesis ไหม?"
ถ้าใช่ → สร้าง `wiki/synthesis/[slug].md` + อัปเดต index.md

### 6. Append log.md
```
## [YYYY-MM-DD] query | [คำถามสั้นๆ]
- Pages consulted: [list]
- Synthesis created: yes/no
```

## หมายเหตุ
- คำตอบที่ดีจะ compound กันในอนาคต — ยิ่ง file เป็น synthesis มาก wiki ยิ่งฉลาดขึ้น
- อย่า hallucinate — ถ้าไม่รู้จาก wiki ให้บอก
