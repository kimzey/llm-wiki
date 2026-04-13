# Document Best Practices — เตรียมเอกสารให้ RAG ทำงานดีที่สุด

## ทำไมเอกสารบางอันถามแล้วตอบไม่ดี?

RAG มีคุณภาพขึ้นกับ 3 สิ่ง:
```
1. คุณภาพของเอกสาร (Garbage in = Garbage out)
2. Chunk strategy (แบ่งยังไง)
3. Query ของ user (ถามชัดแค่ไหน)

ส่วนใหญ่ปัญหามาจาก → คุณภาพเอกสาร + chunk strategy
```

---

## ประเภทเอกสาร → คาดหวังผลลัพธ์ได้อย่างไร

### ✅ Tier 1: ดีมาก (ผลลัพธ์ 95%+)

```
เอกสารที่ Docling + RAG ทำงานได้ดีที่สุด:

PDF ที่สร้างจากโปรแกรม (Digital Native):
  - Word → Save as PDF
  - Google Docs → Export PDF
  - LaTeX → PDF
  - เหตุผล: มี text layer ชัดเจน ไม่ต้อง OCR

DOCX จาก Microsoft Word:
  - Structure ชัดเจน (heading, paragraph, table)
  - ง่ายสำหรับ Docling มาก

HTML / Web pages:
  - Structure ดี ถ้า semantic HTML
  - ดีสำหรับ documentation sites
```

### ⚠️ Tier 2: ดี แต่ต้องระวัง

```
Scanned PDF คุณภาพสูง (300 DPI+):
  - OCR ทำงานได้ดี ~85-95%
  - อาจผิดพลาดในตัวเลขหรือชื่อพิเศษ
  - แก้: เปิด ocr_enabled: true ใน config.yaml

PPTX / PowerPoint:
  - ข้อความใน slide ดึงได้ดี
  - แต่ context อาจน้อย (slides มักมีแค่ bullet points)
  - แก้: เพิ่ม speaker notes ลงใน slide

XLSX / Excel:
  - ข้อมูลตารางดึงได้ดี
  - แต่ formula และ calculated values อาจไม่ตรง
  - แก้: Export เป็น CSV หรือ PDF แทน
```

### ❌ Tier 3: มีปัญหา ต้องแก้ไขก่อน

```
Scanned PDF คุณภาพต่ำ:
  - DPI ต่ำกว่า 150: OCR แม่นยำต่ำมาก
  - เอกสารเบี้ยว: OCR อ่านผิด
  - แก้: Scan ใหม่ที่ 300 DPI+, ปรับ rotation

เอกสารที่ encrypt / password protected:
  - Docling อ่านไม่ได้เลย
  - แก้: ลบ password ก่อน แล้วค่อย upload

PDF ที่เป็น vector graphics ล้วน:
  - เช่น brochure ที่ design ใน Illustrator
  - ไม่มี text layer เลย
  - แก้: ขอไฟล์ต้นฉบับ (DOCX) หรือ Scan

ลายมือเขียน:
  - OCR ทั่วไปแม่นยำต่ำมาก
  - แก้: พิมพ์ดิจิทัลแทน หรือใช้ Google Vision API
```

---

## วิธีเตรียมเอกสารให้ดี

### 1. โครงสร้างเอกสาร — ใช้ Heading

```
❌ BAD: ไม่มี heading structure
  ลาป่วย พนักงานมีสิทธิลา 30 วัน ต้องมีใบรับรองแพทย์...
  ลากิจ พนักงานมีสิทธิลา 3 วัน ต้องได้รับอนุมัติ...

✅ GOOD: มี Heading ชัดเจน
  # Leave Policy

  ## 1. Sick Leave
  พนักงานมีสิทธิลาป่วย 30 วันต่อปี
  ต้องมีใบรับรองแพทย์เมื่อลาเกิน 3 วัน

  ## 2. Personal Leave
  พนักงานมีสิทธิลากิจ 3 วันต่อปี
  ต้องได้รับอนุมัติจากหัวหน้าก่อน

เหตุผล: Docling ใช้ heading เพื่อแบ่ง section
         chunk จะตรง section → ค้นหาแม่นยำกว่า
```

### 2. ตาราง — ใส่ Header ให้ชัด

```
❌ BAD:
  Widget, $10, 100 units
  Gadget, $25, 50 units

✅ GOOD:
  | Product | Price | Stock |
  |---------|-------|-------|
  | Widget  | $10   | 100   |
  | Gadget  | $25   | 50    |

เหตุผล: Docling ใช้ TableFormer แยก row/column
         ถ้ามี header → แยกได้ถูกต้อง
```

### 3. หลีกเลี่ยง Layout ซับซ้อน

```
❌ หลีกเลี่ยง:
  - 3 columns layout (magazine-style)
  - Text ที่อยู่ใน text box / shape
  - Watermark ที่ซ้อนทับเนื้อหา
  - Header/Footer ที่มีเนื้อหาสำคัญมาก

✅ แนะนำ:
  - Single column
  - ข้อความในเนื้อหาหลัก
  - Header/Footer เป็นแค่ page number และชื่อเอกสาร
```

### 4. ภาษา — ใช้ให้สอดคล้องกัน

```
✅ เอกสารภาษาไทยทั้งหมด → ถามภาษาไทย → ดีมาก
✅ เอกสารภาษาอังกฤษทั้งหมด → ถามภาษาอังกฤษ → ดีมาก

⚠️ เอกสารปนกัน (Thai + English) → embedding อาจสับสน
   แก้: แยก index หรือ แยก knowledge filter ตามภาษา

⚠️ คำศัพท์เฉพาะทาง (ชื่อผลิตภัณฑ์, รหัส, โค้ด)
   → Embedding เข้าใจได้ แต่ต้อง exact match ถ้าถามด้วยรหัส
   → แนะนำ: อธิบาย context ด้วยเสมอ
   ตัวอย่างที่ดี: "สินค้า Widget Pro (SKU-1234) ราคา $10"
```

---

## Naming Convention — ตั้งชื่อไฟล์

```
❌ BAD:
  scan001.pdf
  เอกสาร.pdf
  final_v3_FINAL2.pdf
  Untitled.pdf

✅ GOOD:
  hr_leave_policy_2024.pdf
  product_catalog_q3_2024.pdf
  it_security_guidelines_v2.pdf
  finance_budget_2024_q4.xlsx

เหตุผล:
  - ชื่อไฟล์ปรากฏใน citation
  - User เห็นชื่อไฟล์รู้ว่ามาจากไหน
  - ง่ายต่อการ manage และ delete
```

---

## Content Tips ที่ช่วย RAG

### เพิ่ม Context ให้ตาราง

```
❌ ตารางแบบไม่มี context:
  | Q1   | Q2   | Q3   | Q4   |
  |------|------|------|------|
  | $1M  | $1.2M| $1.5M| $1.8M|

✅ ตารางแบบมี context:
  ตาราง: รายได้ประจำปี 2024 (หน่วย: ล้านบาท)
  | ไตรมาส | รายได้ |
  |--------|--------|
  | Q1     | $1M    |
  | Q2     | $1.2M  |
  | Q3     | $1.5M  |
  | Q4     | $1.8M  |
  รายได้รวมปี 2024: $5.5M เพิ่มขึ้น 15% จากปี 2023
```

### เพิ่มคำอธิบายหลังภาพ

```
❌ PDF ที่มีแต่ภาพ + chart ไม่มีคำอธิบาย
   → RAG ไม่เห็นข้อมูลในภาพ (ถ้าไม่เปิด picture_descriptions)

✅ เพิ่ม Alt text หรือ Caption:
   [ภาพ: กราฟแสดงการเติบโตของยอดขาย Q1-Q4 2024]
   ยอดขาย Q1: 1M, Q2: 1.2M, Q3: 1.5M, Q4: 1.8M
   แนวโน้ม: เติบโตต่อเนื่อง 20-25% ต่อไตรมาส

หรือเปิด picture_descriptions: true ใน config.yaml
→ AI จะอธิบายภาพอัตโนมัติ (แต่เพิ่ม cost)
```

### Acronyms และ คำย่อ

```
❌ เนื้อหามีแต่ตัวย่อ:
  "ตาม SLA ของ TH-BKK-DC01 กำหนด RTO 4 ชม. และ RPO 1 ชม."

✅ อธิบายครั้งแรกที่ใช้:
  "ตาม Service Level Agreement (SLA) ของ Data Center กรุงเทพ (TH-BKK-DC01)
   กำหนด Recovery Time Objective (RTO) ไว้ที่ 4 ชั่วโมง
   และ Recovery Point Objective (RPO) ไว้ที่ 1 ชั่วโมง"

เหตุผล: User อาจถามด้วยคำเต็ม แต่เอกสารมีแค่ตัวย่อ
         → vector ไม่ match → ค้นหาไม่เจอ
```

---

## Document Organization Strategy

### แบ่งเอกสารตามหมวดหมู่ + Access Control

```
โครงสร้างที่แนะนำ:

📁 HR Documents
   hr_leave_policy_2024.pdf          → allowed_groups: all-employees
   hr_salary_structure_2024.pdf      → allowed_groups: hr-team, management
   hr_performance_review_template.pdf → allowed_groups: hr-team

📁 IT Documents
   it_security_policy.pdf            → allowed_groups: all-employees
   it_network_architecture.pdf       → allowed_groups: it-team
   it_server_runbook.pdf             → allowed_groups: it-team, devops-team

📁 Finance Documents
   finance_budget_2024.pdf           → allowed_groups: finance-team, management
   finance_expense_policy.pdf        → allowed_groups: all-employees
   finance_q3_report.pdf             → allowed_groups: management

📁 Product Documents
   product_catalog_2024.pdf          → allowed_groups: all-employees
   product_technical_spec.pdf        → allowed_groups: engineering-team
   product_pricing_internal.pdf      → allowed_groups: sales-team, management
```

### ใช้ Knowledge Filters แทน Group ซับซ้อน

```bash
# แทนที่จะตั้งค่า allowed_groups ซับซ้อนในแต่ละไฟล์
# สร้าง Knowledge Filter ครั้งเดียว:

# HR Bot Filter
POST /knowledge-filter
{
  "name": "HR Knowledge Base",
  "filter": {"terms": {"allowed_groups": ["hr-team", "all-employees"]}}
}

# Management Bot Filter
POST /knowledge-filter
{
  "name": "Management Dashboard",
  "filter": {"terms": {"allowed_groups": ["management"]}}
}

# แล้วใช้ filter_id ตอน chat แทน
```

---

## Chunk Strategy ตามประเภทเนื้อหา

### Policy / Procedure Documents

```yaml
# เนื้อหา: นโยบาย, ขั้นตอน, ระเบียบ
# ลักษณะ: prose ยาว มี section หลายส่วน

knowledge:
  chunk_size: 1000       # ค่า default
  chunk_overlap: 200
```

### FAQ / Q&A Documents

```yaml
# เนื้อหา: คำถาม-คำตอบ
# ลักษณะ: แต่ละ Q&A เป็น unit ที่สมบูรณ์ในตัวเอง

knowledge:
  chunk_size: 400        # เล็กลง → แต่ละ chunk = 1 Q&A
  chunk_overlap: 50      # overlap น้อยลง

# TIP: เตรียม FAQ ให้ชัด:
  Q: ลาป่วยได้กี่วัน?
  A: พนักงานมีสิทธิลาป่วยได้ 30 วันต่อปี...

  Q: ต้องมีใบรับรองแพทย์ไหม?
  A: ต้องมีใบรับรองแพทย์เมื่อลาเกิน 3 วัน...
```

### Technical Documentation

```yaml
# เนื้อหา: API docs, architecture, specs
# ลักษณะ: มี code, ต้องการ context ยาว

knowledge:
  chunk_size: 1500       # ใหญ่ขึ้น → ได้ context มากขึ้น
  chunk_overlap: 300
```

### Product Catalog / Pricing

```yaml
# เนื้อหา: รายการสินค้า, ราคา, specs
# ลักษณะ: structured data, short items

knowledge:
  chunk_size: 500        # เล็ก → แต่ละ chunk = สินค้าไม่กี่ชิ้น
  chunk_overlap: 100
```

---

## การ Update เอกสาร

```bash
# เมื่อเอกสารมีการอัปเดต (เช่น นโยบายใหม่ปี 2025):

# ขั้นตอน:
# 1. ลบเอกสารเก่า
curl -X POST "http://localhost:8080/documents/delete-by-filename" \
  -H "Authorization: Bearer API_KEY" \
  -d '{"filename": "hr_leave_policy_2024.pdf"}'

# 2. Upload เอกสารใหม่
curl -X POST "http://localhost:8080/v1/documents/ingest" \
  -H "Authorization: Bearer API_KEY" \
  -F "file=@hr_leave_policy_2025.pdf" \
  -F "allowed_groups=all-employees"

# ⚠️ สำคัญ: อย่าลืมลบเก่าก่อน
# ถ้ามีทั้งปี 2024 และ 2025 → AI อาจตอบสับสน
```

---

## ทดสอบ RAG Quality

### วิธีทดสอบว่า RAG ทำงานดีไหม:

```
1. ถาม "ทดสอบ" ด้วยคำถามที่รู้คำตอบอยู่แล้ว
   เช่น: "ลาป่วยได้กี่วัน?" → ควรตอบ "30 วัน"

2. ตรวจสอบ Citation
   → ชื่อไฟล์ถูกต้องไหม?
   → หน้าถูกต้องไหม?

3. ถามคำถามที่ไม่มีคำตอบในเอกสาร
   → ควรบอกว่า "ไม่พบข้อมูล" ไม่ควร hallucinate

4. ถามคำถามเดียวกันหลายแบบ:
   "ลาป่วย" = "sick leave" = "การลาเจ็บป่วย"
   → ควรได้ข้อมูลเดียวกัน (test semantic search)
```

### Signs ว่า RAG ทำงานไม่ดี:

```
❌ ตอบผิด ทั้งที่เอกสารมีคำตอบ
   → ตรวจ chunk size, ลอง embed query เทียบกับ chunks

❌ ตอบไม่พบ ทั้งที่เอกสารมีข้อมูล
   → ตรวจ access control, ตรวจ embedding model

❌ ตอบด้วยข้อมูลที่ไม่อยู่ในเอกสาร (hallucination)
   → แก้ system prompt: "ตอบเฉพาะจาก context ที่ให้มา"

❌ Citation ผิด (ชี้ไปไฟล์ผิด)
   → ตรวจ chunk overlap, อาจมี chunks ที่ปนกัน
```

---

## Checklist ก่อน Go-Live

```
เอกสาร:
[ ] ไฟล์ทุกอันตั้งชื่อสื่อความหมาย
[ ] ลบไฟล์ duplicate / version เก่าออก
[ ] ตรวจว่า PDF อ่านได้ (ไม่ encrypted)
[ ] เอกสาร scanned มี DPI ≥ 300

Access Control:
[ ] ตั้ง allowed_groups ทุกไฟล์
[ ] ทดสอบว่า user เห็นเฉพาะเอกสารที่ควรเห็น
[ ] ทดสอบว่า user ไม่เห็นเอกสาร confidential

RAG Quality:
[ ] ทดสอบถามคำถามทุก topic หลัก
[ ] ตรวจ citation ว่าถูกต้อง
[ ] ทดสอบถามเป็นภาษาไทยและภาษาอังกฤษ
[ ] ทดสอบถามสิ่งที่ไม่อยู่ในเอกสาร

System:
[ ] Docling health check ผ่าน
[ ] OpenSearch index มี documents
[ ] Backend API ตอบสนอง
[ ] Frontend load ได้ปกติ
```

---

*กลับไป: [cost-and-models.md](./cost-and-models.md)*
*index ทั้งหมด: [README.md](./README.md)*
