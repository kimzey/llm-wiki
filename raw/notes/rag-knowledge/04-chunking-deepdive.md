# Chunking Deep Dive — ทุกวิธีที่มี + ข้อมูลจริงในแต่ละขั้นตอน

---

## สารบัญ

1. [Chunking คืออะไร ทำไมต้องทำ](#1-chunking-คืออะไร)
2. [ข้อมูลก่อน Chunk vs หลัง Chunk หน้าตาเป็นยังไง](#2-ข้อมูลก่อน-vs-หลัง-chunk)
3. [วิธี Chunking ทั้งหมด 10 วิธี พร้อม Output จริง](#3-วิธี-chunking-ทั้งหมด)
4. [Chunk แล้วเก็บยังไง — โครงสร้างใน Database](#4-chunk-แล้วเก็บยังไง)
5. [Chunk แล้วเชื่อมกันยังไง — Linking & Relationships](#5-chunk-แล้วเชื่อมกันยังไง)
6. [ปัจจุบัน 2025 เค้าทำกันยังไง](#6-ปัจจุบัน-2025)
7. [เลือกวิธีไหนดี — Decision Matrix](#7-เลือกวิธีไหน)
8. [Common Mistakes](#8-common-mistakes)

---

## 1. Chunking คืออะไร

### ทำไมต้อง Chunk

```
สมมติมีเอกสาร "คู่มือพนักงาน Sellsuki" ยาว 50 หน้า

ปัญหาถ้าไม่ chunk:

  1. Embedding Model มี limit
     text-embedding-3-small รับได้สูงสุด 8,191 tokens
     เอกสาร 50 หน้า ≈ 25,000 tokens → ใส่ไม่ได้!

  2. ยิ่งข้อความยาว embedding ยิ่ง "เบลอ"
     Embed ทั้งเอกสาร 50 หน้า → vector กลายเป็น "ค่าเฉลี่ย" ของทุกเรื่อง
     → ค้นหา "ลาพักร้อน" → vector ไม่ใกล้เพราะเอกสารพูดเรื่องอื่นด้วย 100 เรื่อง

  3. ส่งให้ LLM ทั้ง 50 หน้า = เปลืองมาก
     GPT-4o-mini: 25,000 tokens × $0.15/1M = $0.00375 ต่อ request
     ถ้า chunk ส่งแค่ 3 chunks (1,500 tokens) = $0.000225 ← ถูกกว่า 16 เท่า

  4. LLM "หลง" ถ้า context ยาวเกิน
     งานวิจัย "Lost in the Middle" (2023): LLM มักลืมข้อมูลกลาง context
     ข้อมูลต้นๆ และท้ายๆ ถูกจำ แต่ตรงกลางหาย
     → ส่งน้อยแต่ตรงประเด็น ดีกว่าส่งเยอะแต่กระจาย


Chunking แก้ปัญหาทั้งหมด:
  50 หน้า → ตัดเป็น 120 chunks เล็กๆ
  แต่ละ chunk ≈ 500-1000 ตัวอักษร
  แต่ละ chunk มี embedding ของตัวเอง
  ค้นหา → ได้แค่ chunks ที่เกี่ยวข้อง → ส่งให้ LLM
```

### องค์ประกอบของ Chunk ที่ดี

```
Chunk ที่ดีต้อง:

✅ 1. Self-contained — อ่านเดี่ยวๆ แล้วเข้าใจ
      ❌ "ต้องแจ้งล่วงหน้า 3 วัน" (ไม่รู้ว่าพูดถึงอะไร)
      ✅ "การลาพักร้อน: ต้องแจ้งล่วงหน้า 3 วัน"

✅ 2. Focused — พูดเรื่องเดียว
      ❌ "ลาพักร้อน 10 วัน... [500 คำ]... WiFi password คือ abc123"
      ✅ "ลาพักร้อน 10 วัน ต้องแจ้งล่วงหน้า ต้องหัวหน้าอนุมัติ"

✅ 3. Right size — ไม่สั้นไม่ยาวเกิน
      ❌ "ลาได้" (สั้นไป ไม่มีบริบท)
      ❌ [3000 คำรวมทุกเรื่อง] (ยาวไป embedding เบลอ)
      ✅ 200-1000 ตัวอักษร (sweet spot)

✅ 4. Has context — รู้ว่ามาจากไหน
      ✅ metadata: {source: "hr_policy.md", section: "ลาพักร้อน"}
```

---

## 2. ข้อมูลก่อน vs หลัง Chunk

### เอกสารต้นฉบับ (ก่อน Chunk)

```markdown
# คู่มือพนักงาน Sellsuki 2024

## 1. นโยบายการลา

### 1.1 ลาพักร้อน
พนักงานประจำมีสิทธิ์ลาพักร้อนปีละ 10 วัน โดยต้องแจ้งล่วงหน้า
อย่างน้อย 3 วันทำการ การลาพักร้อนต้องได้รับอนุมัติจากหัวหน้างาน
โดยตรง หากลาเกิน 3 วันติดต่อกันต้องแจ้งล่วงหน้า 1 สัปดาห์
พนักงานที่ทำงานไม่ครบ 1 ปี จะได้สิทธิ์ตามสัดส่วน

### 1.2 ลาป่วย
พนักงานมีสิทธิ์ลาป่วยปีละ 30 วัน โดยได้รับค่าจ้าง หากลาป่วย
เกิน 3 วันติดต่อกัน ต้องมีใบรับรองแพทย์ประกอบ กรณีป่วยหนัก
อาจขอลาพิเศษเพิ่มได้โดยพิจารณาเป็นรายกรณี

### 1.3 ลากิจ
พนักงานมีสิทธิ์ลากิจปีละ 5 วัน สำหรับธุระส่วนตัวที่จำเป็น เช่น
งานแต่งงาน งานศพ ย้ายบ้าน ต้องแจ้งล่วงหน้าอย่างน้อย 1 วัน
ยกเว้นกรณีฉุกเฉิน

## 2. สวัสดิการ

### 2.1 ประกันสุขภาพ
บริษัทจัดให้มีประกันสุขภาพกลุ่มสำหรับพนักงานทุกคน ครอบคลุม
ค่ารักษาพยาบาลผู้ป่วยนอก (OPD) สูงสุด 2,000 บาท/ครั้ง
ผู้ป่วยใน (IPD) สูงสุด 100,000 บาท/ปี ทันตกรรม 3,000 บาท/ปี

### 2.2 กองทุนสำรองเลี้ยงชีพ
บริษัทสมทบ 5% ของเงินเดือน พนักงานสมทบ 3-15% ตามความสมัครใจ
เริ่มสิทธิ์เมื่อทำงานครบ 1 ปี
```

### หลัง Chunk — ข้อมูลจริงที่ถูกเก็บ

```
หลังจาก chunk + embed + enrich แล้ว
แต่ละ chunk จะถูกเก็บเป็น record ใน database:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📦 Chunk #1
┌────────────────────────────────────────────────────────────────┐
│ id:           1                                                │
│ content:      "พนักงานประจำมีสิทธิ์ลาพักร้อนปีละ 10 วัน       │
│                โดยต้องแจ้งล่วงหน้าอย่างน้อย 3 วันทำการ        │
│                การลาพักร้อนต้องได้รับอนุมัติจากหัวหน้างาน      │
│                โดยตรง หากลาเกิน 3 วันติดต่อกันต้องแจ้ง         │
│                ล่วงหน้า 1 สัปดาห์ พนักงานที่ทำงานไม่ครบ 1 ปี   │
│                จะได้สิทธิ์ตามสัดส่วน"                          │
│                                                                │
│ summary:      "ลาพักร้อน 10วัน/ปี แจ้ง3วัน เกิน3วันแจ้ง1สัปดาห์│
│                ไม่ครบปีได้ตามสัดส่วน"                           │
│                                                                │
│ embedding:    [0.034, -0.128, 0.891, 0.023, ..., 0.045]       │
│               └──────────── 768 ตัวเลข ──────────────┘        │
│                                                                │
│ content_hash: "a3f2b8c1d4e5...f0a1"  (SHA256)                 │
│                                                                │
│ ── Metadata ──────────────────────────────────────────────────│
│ source:       "handbook_2024.md"                               │
│ category:     "hr"                                             │
│ department:   "HR"                                             │
│ doc_type:     "policy"                                         │
│ title:        "ลาพักร้อน"                                      │
│ header_1:     "นโยบายการลา"                                    │
│ header_2:     "ลาพักร้อน"                                      │
│ chunk_index:  0                                                │
│ chunk_total:  6                                                │
│ parent_doc_id: "doc_handbook_2024"                              │
│ prev_chunk_id: null                                            │
│ next_chunk_id: 2                                               │
│ last_updated: "2024-01-15"                                     │
│ created_at:   "2024-12-01T10:30:00Z"                           │
└────────────────────────────────────────────────────────────────┘

📦 Chunk #2
┌────────────────────────────────────────────────────────────────┐
│ id:           2                                                │
│ content:      "พนักงานมีสิทธิ์ลาป่วยปีละ 30 วัน โดยได้รับ     │
│                ค่าจ้าง หากลาป่วยเกิน 3 วันติดต่อกัน ต้องมี     │
│                ใบรับรองแพทย์ประกอบ กรณีป่วยหนักอาจขอลาพิเศษ    │
│                เพิ่มได้โดยพิจารณาเป็นรายกรณี"                  │
│                                                                │
│ summary:      "ลาป่วย 30วัน/ปี ได้ค่าจ้าง เกิน3วันใช้ใบแพทย์  │
│                ป่วยหนักขอพิเศษได้"                              │
│                                                                │
│ embedding:    [0.012, 0.445, -0.234, 0.087, ..., 0.091]       │
│                                                                │
│ content_hash: "b7d3e9f2a1c8...d4b2"                            │
│                                                                │
│ source:       "handbook_2024.md"                               │
│ category:     "hr"                                             │
│ title:        "ลาป่วย"                                         │
│ header_1:     "นโยบายการลา"                                    │
│ header_2:     "ลาป่วย"                                         │
│ chunk_index:  1                                                │
│ parent_doc_id: "doc_handbook_2024"                              │
│ prev_chunk_id: 1                                               │
│ next_chunk_id: 3                                               │
└────────────────────────────────────────────────────────────────┘

📦 Chunk #3
┌────────────────────────────────────────────────────────────────┐
│ id:           3                                                │
│ content:      "พนักงานมีสิทธิ์ลากิจปีละ 5 วัน สำหรับธุระ      │
│                ส่วนตัวที่จำเป็น เช่น งานแต่งงาน งานศพ ย้ายบ้าน │
│                ต้องแจ้งล่วงหน้าอย่างน้อย 1 วัน ยกเว้นกรณี     │
│                ฉุกเฉิน"                                        │
│                                                                │
│ summary:      "ลากิจ 5วัน/ปี แจ้ง1วัน ฉุกเฉินได้"              │
│                                                                │
│ embedding:    [-0.091, 0.234, 0.567, -0.012, ..., 0.034]      │
│                                                                │
│ content_hash: "c4a8b1d6e3f7...a5c3"                            │
│                                                                │
│ source:       "handbook_2024.md"                               │
│ category:     "hr"                                             │
│ title:        "ลากิจ"                                          │
│ header_1:     "นโยบายการลา"                                    │
│ header_2:     "ลากิจ"                                          │
│ chunk_index:  2                                                │
│ parent_doc_id: "doc_handbook_2024"                              │
│ prev_chunk_id: 2                                               │
│ next_chunk_id: 4                                               │
└────────────────────────────────────────────────────────────────┘

... (Chunk #4: ประกันสุขภาพ, Chunk #5: กองทุนสำรองเลี้ยงชีพ, ...)
```

---

## 3. วิธี Chunking ทั้งหมด 10 วิธี พร้อม Output จริง

### สมมติ Document ต้นฉบับ (ใช้ทดสอบทุกวิธี)

```
เอกสาร: "leave_policy.md" (389 ตัวอักษร)

"# นโยบายการลา
## ลาพักร้อน
พนักงานประจำลาพักร้อนได้ 10 วัน/ปี ต้องแจ้งล่วงหน้า 3 วัน
หากลาเกิน 3 วันต้องแจ้ง 1 สัปดาห์ ต้องหัวหน้าอนุมัติ
## ลาป่วย
พนักงานลาป่วยได้ 30 วัน/ปี ได้รับค่าจ้าง
ลาเกิน 3 วันต้องมีใบรับรองแพทย์ ป่วยหนักขอพิเศษได้"
```

---

### วิธี 1: Fixed Size Chunking

```
วิธี:   ตัดทุกๆ N ตัวอักษร ไม่สนเรื่อง content เลย
ง่าย:   ★★★★★
คุณภาพ: ★★☆☆☆

Settings: chunk_size=150, overlap=30

Output:
┌─────────────────────────────────────────────────────────────┐
│ Chunk 1 (0-150):                                            │
│ "# นโยบายการลา\n## ลาพักร้อน\nพนักงานประจำลาพักร้อนได้     │
│  10 วัน/ปี ต้องแจ้งล่วงหน้า 3 วัน\nหากลาเกิน 3 วันต้อง"   │
│                                        ↑ ตัดกลางประโยค!     │
├─────────────────────────────────────────────────────────────┤
│ Chunk 2 (120-270):                                          │
│ "วันต้องแจ้ง 1 สัปดาห์ ต้องหัวหน้าอนุมัติ\n## ลาป่วย\n    │
│  พนักงานลาป่วยได้ 30 วัน/ปี ได้รับค่าจ้าง\nลาเกิน 3 วัน"  │
│       ↑ overlap 30 chars                    ↑ ตัดอีก!       │
├─────────────────────────────────────────────────────────────┤
│ Chunk 3 (240-389):                                          │
│ "น 3 วันต้องมีใบรับรองแพทย์ ป่วยหนักขอพิเศษได้"            │
└─────────────────────────────────────────────────────────────┘

❌ ปัญหา:
  - ตัดกลางประโยค "ต้อง|แจ้ง"
  - Chunk 2 ผสมทั้ง "ลาพักร้อน" และ "ลาป่วย" ใน chunk เดียว
  - ไม่มี context ว่ากำลังพูดถึงอะไร

ใช้เมื่อ: แทบไม่มีใครใช้แล้วในปัจจุบัน
```

### วิธี 2: Recursive Character Splitting

```
วิธี:   พยายามตัดที่ "\n\n" ก่อน → "\n" → "." → " " → ตัวอักษร
ง่าย:   ★★★★☆
คุณภาพ: ★★★★☆

Settings: chunk_size=200, overlap=30
Separators: ["\n\n", "\n", ". ", " "]

ขั้นตอน:
  1. ลองตัดที่ "\n\n" (paragraph break)
     → ได้ 2 ส่วน: [นโยบายลาพักร้อน, นโยบายลาป่วย]
  2. แต่ละส่วนยาวไม่เกิน 200 → เสร็จ

Output:
┌─────────────────────────────────────────────────────────────┐
│ Chunk 1:                                                     │
│ "## ลาพักร้อน                                                │
│  พนักงานประจำลาพักร้อนได้ 10 วัน/ปี ต้องแจ้งล่วงหน้า 3 วัน  │
│  หากลาเกิน 3 วันต้องแจ้ง 1 สัปดาห์ ต้องหัวหน้าอนุมัติ"     │
├─────────────────────────────────────────────────────────────┤
│ Chunk 2:                                                     │
│ "## ลาป่วย                                                   │
│  พนักงานลาป่วยได้ 30 วัน/ปี ได้รับค่าจ้าง                    │
│  ลาเกิน 3 วันต้องมีใบรับรองแพทย์ ป่วยหนักขอพิเศษได้"        │
└─────────────────────────────────────────────────────────────┘

✅ ดีขึ้นมาก:
  - ตัดที่ paragraph break ไม่ตัดกลางประโยค
  - แต่ละ chunk พูดเรื่องเดียว
  - แต่ไม่มี header_1 "นโยบายการลา" ติดมาด้วย

ใช้เมื่อ: เอกสารที่ไม่มี structure ชัดเจน (plain text, meeting notes)
         ← ใช้บ่อยที่สุดในปัจจุบัน
```

### วิธี 3: Markdown/HTML Header Splitting

```
วิธี:   ตัดตามโครงสร้าง headers (# ## ###)
ง่าย:   ★★★★☆
คุณภาพ: ★★★★★

Output:
┌─────────────────────────────────────────────────────────────┐
│ Chunk 1:                                                     │
│ content: "พนักงานประจำลาพักร้อนได้ 10 วัน/ปี ต้องแจ้ง       │
│           ล่วงหน้า 3 วัน หากลาเกิน 3 วันต้องแจ้ง 1 สัปดาห์  │
│           ต้องหัวหน้าอนุมัติ"                                │
│                                                              │
│ metadata:                                                    │
│   header_1: "นโยบายการลา"     ← H1 ติดมาด้วย!               │
│   header_2: "ลาพักร้อน"       ← H2 ติดมาด้วย!               │
├─────────────────────────────────────────────────────────────┤
│ Chunk 2:                                                     │
│ content: "พนักงานลาป่วยได้ 30 วัน/ปี ได้รับค่าจ้าง           │
│           ลาเกิน 3 วันต้องมีใบรับรองแพทย์ ป่วยหนักขอพิเศษได้"│
│                                                              │
│ metadata:                                                    │
│   header_1: "นโยบายการลา"                                    │
│   header_2: "ลาป่วย"                                         │
└─────────────────────────────────────────────────────────────┘

✅ ดีมาก:
  - แต่ละ chunk มี topic ชัดเจน
  - metadata เก็บ hierarchy (รู้ว่าอยู่ section ไหน)
  - ค้นหาได้แม่น + filter ด้วย header ได้

ใช้เมื่อ: Markdown docs, HTML pages, เอกสารที่มี headers ← แนะนำ
```

### วิธี 4: Hierarchical (File → Header → Sub-chunk)

```
วิธี:   แบ่งตาม file ก่อน → ตาม header → ถ้ายังยาวก็ recursive split
ง่าย:   ★★★☆☆
คุณภาพ: ★★★★★

ขั้นตอน:
  1. File level:   leave_policy.md (1 file)
  2. Header level: "ลาพักร้อน", "ลาป่วย" (2 sections)
  3. Sub-chunk:    ถ้า section > 1000 chars → recursive split
                   ถ้า section ≤ 1000 chars → เก็บเลย

Output: (เหมือนวิธี 3 แต่จัดการเอกสารใหญ่ได้ดีกว่า)

กรณี section ยาวมาก (สมมติลาพักร้อนมี 2000 ตัวอักษร):
┌─────────────────────────────────────────────────────────────┐
│ Chunk 1a:                                                    │
│ content: "พนักงานประจำลาพักร้อนได้ 10 วัน/ปี... [ส่วนแรก]"  │
│ metadata:                                                    │
│   header_1: "นโยบายการลา"                                    │
│   header_2: "ลาพักร้อน"                                      │
│   sub_chunk_index: 0    ← บอกว่าเป็น sub-chunk ที่ 0         │
│   sub_chunk_total: 2    ← จาก section นี้มี 2 sub-chunks     │
├─────────────────────────────────────────────────────────────┤
│ Chunk 1b:                                                    │
│ content: "...พนักงานทดลองงานไม่มีสิทธิ์ลา... [ส่วนที่สอง]"  │
│ metadata:                                                    │
│   header_1: "นโยบายการลา"                                    │
│   header_2: "ลาพักร้อน"                                      │
│   sub_chunk_index: 1                                         │
│   sub_chunk_total: 2                                         │
└─────────────────────────────────────────────────────────────┘

ใช้เมื่อ: Documentation ที่มี structure, คู่มือ, wiki
         ← Elysia ใช้วิธีนี้
```

### วิธี 5: Sentence Splitting

```
วิธี:   ตัดเป็นประโยค แล้วรวมหลายประโยคเป็น 1 chunk
ง่าย:   ★★★★☆
คุณภาพ: ★★★☆☆

ขั้นตอน:
  1. แบ่งเป็นประโยค (split by . หรือ sentence tokenizer)
  2. รวม N ประโยคเป็น 1 chunk

Output (3 ประโยคต่อ chunk):
┌─────────────────────────────────────────────────────────────┐
│ Chunk 1:                                                     │
│ "พนักงานประจำลาพักร้อนได้ 10 วัน/ปี.                        │
│  ต้องแจ้งล่วงหน้า 3 วัน.                                     │
│  หากลาเกิน 3 วันต้องแจ้ง 1 สัปดาห์."                        │
├─────────────────────────────────────────────────────────────┤
│ Chunk 2:                                                     │
│ "ต้องหัวหน้าอนุมัติ.                                         │
│  พนักงานลาป่วยได้ 30 วัน/ปี.                                 │
│  ได้รับค่าจ้าง."                                             │
├─────────────────────────────────────────────────────────────┤
│ Chunk 3:                                                     │
│ "ลาเกิน 3 วันต้องมีใบรับรองแพทย์.                            │
│  ป่วยหนักขอพิเศษได้."                                       │
└─────────────────────────────────────────────────────────────┘

❌ ปัญหา:
  - Chunk 2 ผสมลาพักร้อน + ลาป่วย
  - ภาษาไทยตัดประโยคยาก (ไม่มี . ชัดเจน)

ใช้เมื่อ: ภาษาอังกฤษที่มี period ชัดเจน
         ไม่แนะนำสำหรับภาษาไทย
```

### วิธี 6: Sentence Window

```
วิธี:   แต่ละ chunk = 1 ประโยค แต่เก็บ "ประโยครอบข้าง" ไว้เป็น context
ง่าย:   ★★★☆☆
คุณภาพ: ★★★★☆

หลักการ:
  Embed เฉพาะประโยคเดียว (precise search)
  แต่ตอนส่งให้ LLM → ส่ง window (ประโยครอบข้าง) ด้วย

Output (window_size=1 คือเก็บ 1 ประโยคก่อน/หลัง):
┌─────────────────────────────────────────────────────────────┐
│ Chunk 2:                                                     │
│ embed_text: "ต้องแจ้งล่วงหน้า 3 วัน"  ← embed แค่นี้        │
│                                                              │
│ window_text: "พนักงานประจำลาพักร้อนได้ 10 วัน/ปี.           │
│               [ต้องแจ้งล่วงหน้า 3 วัน.]                      │
│               หากลาเกิน 3 วันต้องแจ้ง 1 สัปดาห์."           │
│                                    ↑ ส่ง window ให้ LLM       │
└─────────────────────────────────────────────────────────────┘

ข้อดี:  search แม่น (embed สั้น) + LLM ได้บริบทครบ (window ยาว)
ข้อเสีย: ต้องเก็บข้อมูล 2 ส่วน (embed_text + window_text)

ใช้เมื่อ: ต้องการ precision สูงมากในการ search
         เช่น legal docs, technical specs
```

### วิธี 7: Semantic Chunking

```
วิธี:   ใช้ embedding ตัดที่ "ความหมายเปลี่ยน"
ง่าย:   ★★☆☆☆
คุณภาพ: ★★★★★

ขั้นตอน:
  1. แบ่งเป็นประโยค
  2. Embed แต่ละประโยค
  3. วัด cosine similarity ระหว่างประโยคติดกัน
  4. ตรงไหน similarity ลดลงมาก = "ความหมายเปลี่ยน" = จุดตัด

ตัวอย่าง:
  ประโยค 1: "ลาพักร้อนได้ 10 วัน"           embed → [0.82, ...]
  ประโยค 2: "ต้องแจ้งล่วงหน้า 3 วัน"        embed → [0.80, ...]
  similarity(1,2) = 0.93  ← สูง → ยังเรื่องเดียวกัน
  
  ประโยค 3: "ต้องหัวหน้าอนุมัติ"            embed → [0.78, ...]
  similarity(2,3) = 0.88  ← ยังสูงอยู่
  
  ประโยค 4: "ลาป่วยได้ 30 วัน"              embed → [0.45, ...]
  similarity(3,4) = 0.52  ← ลดลงมาก! → ตัดตรงนี้!
  
  ประโยค 5: "ต้องมีใบรับรองแพทย์"           embed → [0.47, ...]
  similarity(4,5) = 0.91  ← สูง → เรื่องเดียวกัน

Output:
┌─────────────────────────────────────────────────────────────┐
│ Chunk 1 (semantic group: "ลาพักร้อน"):                       │
│ "ลาพักร้อนได้ 10 วัน ต้องแจ้งล่วงหน้า 3 วัน                 │
│  ต้องหัวหน้าอนุมัติ"                                        │
├─────────────────────────────────────────────────────────────┤
│ Chunk 2 (semantic group: "ลาป่วย"):                          │
│ "ลาป่วยได้ 30 วัน ต้องมีใบรับรองแพทย์                       │
│  ป่วยหนักขอพิเศษได้"                                        │
└─────────────────────────────────────────────────────────────┘

✅ แม่นที่สุด — AI ตัดสินใจว่า "เรื่องเปลี่ยน" ตรงไหน
❌ ช้า + แพง — ต้อง embed ทุกประโยคก่อน (ตอน index)
❌ ภาษาไทย — ตัดประโยคยาก

ใช้เมื่อ: เอกสารไม่มี structure เลย เช่น meeting transcripts, notes
```

### วิธี 8: Parent-Child (Small-to-Big)

```
วิธี:   สร้าง 2 ระดับ: child chunks (เล็ก, search) + parent chunks (ใหญ่, context)
ง่าย:   ★★★☆☆
คุณภาพ: ★★★★★

หลักการ:
  Child chunk  = เล็ก (200 chars) → embed ไว้ search (แม่น)
  Parent chunk = ใหญ่ (1000 chars) → ส่งให้ LLM (มีบริบทครบ)
  
  Search → หา child ที่ตรง → ดึง parent ของมันมาส่ง LLM

Output:
┌─────────────────────────────────────────────────────────────┐
│ PARENT Chunk P1 (ทั้ง section "ลาพักร้อน"):                  │
│ id: "parent_1"                                               │
│ content: "พนักงานประจำลาพักร้อนได้ 10 วัน/ปี ต้องแจ้ง       │
│           ล่วงหน้า 3 วัน หากลาเกิน 3 วันต้องแจ้ง 1 สัปดาห์  │
│           ต้องหัวหน้าอนุมัติ พนักงานทดลองงานไม่มีสิทธิ์      │
│           พนักงานที่ทำงานไม่ครบ 1 ปี ได้ตามสัดส่วน           │
│           สะสมข้ามปีไม่ได้..."                                │
│ ← ไม่ได้ embed (ไม่ใช้ search)                               │
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ CHILD Chunk C1:                                      │   │
│   │ id: "child_1"                                        │   │
│   │ parent_id: "parent_1"  ← ชี้ไปหา parent              │   │
│   │ content: "ลาพักร้อนได้ 10 วัน/ปี ต้องแจ้ง 3 วัน"    │   │
│   │ embedding: [0.82, -0.15, ...]  ← embed ไว้ search    │   │
│   ├─────────────────────────────────────────────────────┤   │
│   │ CHILD Chunk C2:                                      │   │
│   │ id: "child_2"                                        │   │
│   │ parent_id: "parent_1"                                │   │
│   │ content: "ลาเกิน 3 วันแจ้ง 1 สัปดาห์ หัวหน้าอนุมัติ" │   │
│   │ embedding: [0.78, 0.23, ...]                         │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

Search flow:
  1. "ลาพักร้อนต้องแจ้งกี่วัน?" → embed → search children
  2. Hit: child_1 (similarity 0.95)
  3. ดึง parent ของ child_1 → parent_1
  4. ส่ง parent_1 (full section) ให้ LLM ← มีบริบทครบ

✅ search แม่น (child เล็ก) + LLM ได้บริบทครบ (parent ใหญ่)

ใช้เมื่อ: เอกสารที่แต่ละ section มีหลาย detail ที่เกี่ยวข้องกัน
```

### วิธี 9: Agentic Chunking

```
วิธี:   ให้ LLM ตัดสินใจเองว่าจะตัดตรงไหน
ง่าย:   ★☆☆☆☆
คุณภาพ: ★★★★★

ขั้นตอน:
  1. ส่งเอกสารให้ LLM
  2. LLM: "เอกสารนี้ควรแบ่งเป็น 3 ส่วน:
           ส่วน 1: ลาพักร้อน (บรรทัด 3-6)
           ส่วน 2: ลาป่วย (บรรทัด 7-9)
           ส่วน 3: ลากิจ (บรรทัด 10-12)"
  3. ตัดตามที่ LLM บอก
  4. ให้ LLM สร้าง title + summary ให้แต่ละ chunk ด้วย

Output:
┌─────────────────────────────────────────────────────────────┐
│ Chunk 1:                                                     │
│ title: "สิทธิ์ลาพักร้อนและเงื่อนไข"   ← LLM ตั้งชื่อให้     │
│ content: "พนักงานประจำลาพักร้อนได้ 10 วัน/ปี..."            │
│ summary: "ลาพักร้อน 10 วัน แจ้ง 3 วัน หัวหน้าอนุมัติ"      │
│ propositions: [                       ← LLM แยกเป็นข้อๆ     │
│   "พนักงานประจำลาพักร้อนได้ 10 วัน/ปี",                     │
│   "ต้องแจ้งล่วงหน้าอย่างน้อย 3 วัน",                        │
│   "ลาเกิน 3 วันต้องแจ้ง 1 สัปดาห์",                         │
│   "ต้องหัวหน้าอนุมัติ",                                      │
│ ]                                                            │
└─────────────────────────────────────────────────────────────┘

✅ คุณภาพดีที่สุดเท่าที่จะเป็นไปได้
❌ แพงมาก (ต้องเรียก LLM ทุก section/page)
❌ ช้ามาก

ใช้เมื่อ: เอกสารสำคัญมากจำนวนน้อย ที่ต้องการคุณภาพสูงสุด
         เช่น สัญญา กฎหมาย
```

### วิธี 10: Late Chunking (2024-2025 ใหม่)

```
วิธี:   Embed ทั้งเอกสารก่อน แล้วค่อยตัดเป็น chunks
         (ต่างจากปกติที่ตัดก่อนแล้วค่อย embed)
ง่าย:   ★★☆☆☆
คุณภาพ: ★★★★★+

หลักการ:
  ปกติ: ตัด → embed แต่ละ chunk → chunk ไม่รู้บริบทของเอกสาร
  Late: embed ทั้งเอกสาร → ได้ token-level embeddings → ตัด
        → แต่ละ chunk ยังมีข้อมูลจากเอกสารทั้งหมดฝังอยู่!

ตัวอย่าง:
  ปกติ:
    "ต้องแจ้งล่วงหน้า 3 วัน" → embed → [0.45, 0.23, ...]
    ← ไม่รู้ว่า "แจ้ง" อะไร แจ้งเรื่องอะไร

  Late Chunking:
    embed ทั้งเอกสาร "ลาพักร้อน...ต้องแจ้งล่วงหน้า 3 วัน..."
    → token embeddings ทุก token รู้ context
    → ตัด "ต้องแจ้งล่วงหน้า 3 วัน" → [0.82, 0.67, ...]
    ← embedding นี้ "รู้" ว่ากำลังพูดถึงลาพักร้อน

ข้อจำกัด:
  ต้องใช้ long-context embedding model เช่น:
    - Jina embeddings-v3 (8192 tokens)
    - nomic-embed-text-v1.5 (8192 tokens)
    
  ยังใหม่มาก (paper ปลาย 2024) library support ยังน้อย

ใช้เมื่อ: cutting-edge ต้องการคุณภาพสูงสุด + มี long-context embed model
```

---

## 4. Chunk แล้วเก็บยังไง

### Database Schema

```sql
-- ==========================================
-- ตารางหลัก: เก็บ chunks
-- ==========================================
CREATE TABLE chunks (
    -- Identity
    id              BIGSERIAL PRIMARY KEY,
    
    -- Content
    content         TEXT NOT NULL,           -- เนื้อหา chunk เต็มๆ
    summary         TEXT,                    -- สรุปสั้นๆ (ลด tokens ตอน query)
    
    -- Vector
    embedding       vector(768) NOT NULL,    -- embedding vector
    
    -- Dedup / Diff
    content_hash    VARCHAR(64) NOT NULL,    -- SHA256 สำหรับ diff indexing
    
    -- Source tracking
    source          VARCHAR(500),            -- ไฟล์ต้นทาง
    source_url      TEXT,                    -- URL (ถ้ามี)
    
    -- Document hierarchy
    parent_doc_id   VARCHAR(100),            -- document ต้นทาง
    
    -- Section hierarchy
    header_1        VARCHAR(300),            -- H1 ที่อยู่เหนือ chunk นี้
    header_2        VARCHAR(300),            -- H2
    header_3        VARCHAR(300),            -- H3
    
    -- Chunk ordering
    chunk_index     INTEGER,                 -- ลำดับ chunk ใน document
    chunk_total     INTEGER,                 -- จำนวน chunks ทั้งหมดของ document
    
    -- Linked list (เชื่อม chunks)
    prev_chunk_id   BIGINT REFERENCES chunks(id),
    next_chunk_id   BIGINT REFERENCES chunks(id),
    
    -- Parent-Child (สำหรับ Small-to-Big)
    parent_chunk_id BIGINT REFERENCES chunks(id),
    chunk_level     VARCHAR(20) DEFAULT 'standard',  -- 'parent', 'child', 'standard'
    
    -- Classification
    category        VARCHAR(100),            -- hr, product, sop
    department      VARCHAR(100),            -- แผนก
    doc_type        VARCHAR(50),             -- policy, manual, faq
    title           VARCHAR(500),            -- ชื่อ section
    
    -- Versioning
    doc_version     VARCHAR(50),             -- เวอร์ชันเอกสาร
    last_updated    TIMESTAMP,               -- เอกสารอัปเดตล่าสุด
    
    -- Extra
    metadata        JSONB DEFAULT '{}',      -- metadata เพิ่มเติม
    language        VARCHAR(10) DEFAULT 'th', -- ภาษา
    
    -- Timestamps
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW()
);

-- ==========================================
-- ตารางเสริม: เก็บ documents (parent level)
-- ==========================================
CREATE TABLE documents (
    id              VARCHAR(100) PRIMARY KEY,
    title           VARCHAR(500),
    source          VARCHAR(500),
    source_url      TEXT,
    category        VARCHAR(100),
    doc_type        VARCHAR(50),
    total_chunks    INTEGER,
    file_hash       VARCHAR(64),             -- hash ของไฟล์ทั้งหมด
    last_indexed    TIMESTAMP DEFAULT NOW(),
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMP DEFAULT NOW()
);

-- ==========================================
-- Indexes
-- ==========================================

-- Vector search (HNSW)
CREATE INDEX idx_chunks_embedding 
ON chunks USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- BM25 full-text search (ParadeDB)
CALL paradedb.create_bm25_index(
    index_name => 'idx_chunks_bm25',
    table_name => 'chunks',
    key_field => 'id',
    text_fields => paradedb.field('content') 
                || paradedb.field('title') 
                || paradedb.field('summary')
);

-- Metadata filtering
CREATE INDEX idx_chunks_category ON chunks(category);
CREATE INDEX idx_chunks_source ON chunks(source);
CREATE INDEX idx_chunks_parent_doc ON chunks(parent_doc_id);
CREATE INDEX idx_chunks_hash ON chunks(content_hash);
CREATE INDEX idx_chunks_level ON chunks(chunk_level);
CREATE INDEX idx_chunks_metadata ON chunks USING gin(metadata);
```

### ข้อมูลจริงใน Database

```
SELECT id, title, category, chunk_index, prev_chunk_id, next_chunk_id, 
       LEFT(content, 50) as content_preview
FROM chunks 
WHERE parent_doc_id = 'doc_handbook_2024'
ORDER BY chunk_index;

 id │ title          │ category │ chunk_index │ prev │ next │ content_preview
────┼────────────────┼──────────┼─────────────┼──────┼──────┼──────────────────────────
  1 │ ลาพักร้อน       │ hr       │           0 │ NULL │    2 │ พนักงานประจำลาพักร้อนได้ 10 วัน/ปี...
  2 │ ลาป่วย          │ hr       │           1 │    1 │    3 │ พนักงานลาป่วยได้ 30 วัน/ปี ได้รับค่า...
  3 │ ลากิจ           │ hr       │           2 │    2 │    4 │ พนักงานมีสิทธิ์ลากิจปีละ 5 วัน...
  4 │ ประกันสุขภาพ     │ hr       │           3 │    3 │    5 │ บริษัทจัดประกันสุขภาพกลุ่ม OPD 2000...
  5 │ กองทุนสำรองฯ    │ hr       │           4 │    4 │ NULL │ บริษัทสมทบ 5% พนักงานสมทบ 3-15%...
```

---

## 5. Chunk แล้วเชื่อมกันยังไง

### 5.1 ทำไมต้องเชื่อม

```
ปัญหา: Chunk แต่ละอันเป็นอิสระ ไม่รู้บริบทรอบข้าง

User ถาม: "สรุปนโยบายการลาทั้งหมด"
Search → เจอ chunk "ลาพักร้อน" (similarity สูงสุด)
→ ตอบแค่เรื่องลาพักร้อน ❌ ไม่ครบ!

ถ้าเชื่อม chunks:
  chunk "ลาพักร้อน" → รู้ว่ามี chunks ข้างเคียง
  → ดึง "ลาป่วย" + "ลากิจ" มาด้วย
  → ตอบครบทุกประเภทการลา ✅
```

### 5.2 วิธีเชื่อม 6 แบบ

```
┌──────────────────────────────────────────────────────────────────┐
│ วิธี 1: Linked List (prev/next)                                 │
│                                                                  │
│  Chunk 1 ──next──▶ Chunk 2 ──next──▶ Chunk 3                    │
│          ◀──prev──         ◀──prev──                             │
│                                                                  │
│  Search hit: Chunk 2                                             │
│  → ดึง prev (Chunk 1) + next (Chunk 3) มาเป็น context           │
│                                                                  │
│  Implementation: prev_chunk_id, next_chunk_id columns            │
│  ใช้เมื่อ: ต้องการบริบทรอบข้าง                                   │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ วิธี 2: Parent-Child (Small-to-Big)                              │
│                                                                  │
│              Parent (full section)                                │
│              ┌────────────────────┐                               │
│              │ ลาพักร้อนทั้งหมด    │                               │
│              └──┬──────────┬──────┘                               │
│                 │          │                                      │
│          Child 1 ▼    Child 2 ▼                                  │
│       ┌──────────┐  ┌──────────┐                                 │
│       │ 10 วัน/ปี │  │ แจ้ง 3 วัน│                                 │
│       └──────────┘  └──────────┘                                 │
│                                                                  │
│  Search hit: Child 2 → ดึง Parent มาส่ง LLM                     │
│  Implementation: parent_chunk_id, chunk_level columns            │
│  ใช้เมื่อ: ต้องการ precise search + full context                 │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ วิธี 3: Document Grouping                                        │
│                                                                  │
│  Document: "handbook_2024.md"                                    │
│  ├── Chunk 1 (ลาพักร้อน)    parent_doc_id = "doc_handbook"       │
│  ├── Chunk 2 (ลาป่วย)       parent_doc_id = "doc_handbook"       │
│  ├── Chunk 3 (ลากิจ)        parent_doc_id = "doc_handbook"       │
│  └── Chunk 4 (ประกัน)       parent_doc_id = "doc_handbook"       │
│                                                                  │
│  Search hit: Chunk 1                                             │
│  → ดึง Chunks ทั้งหมดที่ parent_doc_id เดียวกัน                  │
│  → ให้ LLM เลือกว่าจะใช้อันไหน                                   │
│                                                                  │
│  Implementation: parent_doc_id column + documents table          │
│  ใช้เมื่อ: "สรุปทั้งเอกสาร" / "เอกสารนี้พูดถึงอะไรบ้าง"         │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ วิธี 4: Section Grouping (header-based)                          │
│                                                                  │
│  header_1 = "นโยบายการลา"                                        │
│  ├── Chunk (header_2 = "ลาพักร้อน")                               │
│  ├── Chunk (header_2 = "ลาป่วย")                                  │
│  └── Chunk (header_2 = "ลากิจ")                                   │
│                                                                  │
│  Search hit: "ลาพักร้อน"                                         │
│  User ถาม: "สรุปนโยบายการลา"                                     │
│  → WHERE header_1 = 'นโยบายการลา' → ได้ทั้ง 3 chunks              │
│                                                                  │
│  Implementation: header_1, header_2, header_3 columns            │
│  ใช้เมื่อ: ต้องการ chunks จาก section เดียวกัน                    │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ วิธี 5: Chunk Overlap (ง่ายที่สุด)                                │
│                                                                  │
│  Chunk 1: "ลาพักร้อน 10 วัน [ต้องแจ้ง 3 วัน]"                    │
│  Chunk 2:              "[ต้องแจ้ง 3 วัน] ลาป่วย 30 วัน"          │
│                         └── overlap ──┘                           │
│                                                                  │
│  ข้อมูลตรง overlap อยู่ทั้ง 2 chunks                               │
│  → ไม่ต้องทำอะไรเพิ่ม เชื่อมกันด้วย overlap                      │
│                                                                  │
│  Implementation: chunk_overlap parameter ตอน split               │
│  ใช้เมื่อ: recursive splitting ทั่วไป                             │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ วิธี 6: Knowledge Graph (ขั้นสูง)                                 │
│                                                                  │
│  (ลาพักร้อน) ──[มีสิทธิ์]──▶ (10 วัน/ปี)                         │
│       │                                                          │
│       ├──[ต้อง]──▶ (แจ้งล่วงหน้า 3 วัน)                          │
│       │                                                          │
│       └──[อนุมัติโดย]──▶ (หัวหน้างาน)                             │
│                                                                  │
│  (ลาป่วย) ──[มีสิทธิ์]──▶ (30 วัน/ปี)                             │
│       │                                                          │
│       └──[เงื่อนไข]──▶ (ใบรับรองแพทย์ ถ้าเกิน 3 วัน)             │
│                                                                  │
│  Implementation: Graph DB (Neo4j) หรือ JSONB relations           │
│  ใช้เมื่อ: ต้องการ reasoning ข้าม entities                        │
│           เช่น "ลาประเภทไหนต้องอนุมัติจากใคร?"                   │
└──────────────────────────────────────────────────────────────────┘
```

### 5.3 Implementation: ดึง Context รอบข้าง

```python
async def get_enriched_context(
    hit_chunk_id: int,
    strategy: str = "neighbors",  # neighbors, section, parent, document
    window_size: int = 1,
) -> list[dict]:
    """
    ได้ chunk ที่ search hit แล้ว → ดึง chunks เพิ่มเติมมาเป็น context
    """
    
    if strategy == "neighbors":
        # ── วิธี 1: ดึง prev/next chunks ──
        query = """
            WITH target AS (
                SELECT id, prev_chunk_id, next_chunk_id, chunk_index, parent_doc_id
                FROM chunks WHERE id = $1
            )
            SELECT c.* FROM chunks c
            WHERE c.parent_doc_id = (SELECT parent_doc_id FROM target)
              AND c.chunk_index BETWEEN 
                  (SELECT chunk_index FROM target) - $2
                  AND 
                  (SELECT chunk_index FROM target) + $2
            ORDER BY c.chunk_index
        """
        rows = await db.fetch(query, hit_chunk_id, window_size)
        
    elif strategy == "section":
        # ── วิธี 4: ดึงทุก chunk ใน section เดียวกัน ──
        query = """
            SELECT c2.* FROM chunks c1
            JOIN chunks c2 ON c2.header_1 = c1.header_1 
                          AND c2.parent_doc_id = c1.parent_doc_id
            WHERE c1.id = $1
            ORDER BY c2.chunk_index
        """
        rows = await db.fetch(query, hit_chunk_id)
        
    elif strategy == "parent":
        # ── วิธี 2: ดึง parent chunk ──
        query = """
            SELECT p.* FROM chunks c
            JOIN chunks p ON p.id = c.parent_chunk_id
            WHERE c.id = $1
        """
        rows = await db.fetch(query, hit_chunk_id)
        
    elif strategy == "document":
        # ── วิธี 3: ดึงทุก chunk ใน document ──
        query = """
            SELECT c2.* FROM chunks c1
            JOIN chunks c2 ON c2.parent_doc_id = c1.parent_doc_id
            WHERE c1.id = $1
            ORDER BY c2.chunk_index
        """
        rows = await db.fetch(query, hit_chunk_id)
    
    return [dict(row) for row in rows]


# ==========================================
# ตัวอย่างการใช้จริง
# ==========================================

# User: "สรุปนโยบายการลาทั้งหมด"
# Search → hit chunk "ลาพักร้อน" (id=1)

# Strategy: section → ดึงทุก chunk ที่ header_1 = "นโยบายการลา"
context_chunks = await get_enriched_context(
    hit_chunk_id=1, 
    strategy="section"
)

# ได้:
# [
#   {id:1, title:"ลาพักร้อน", content:"ลาพักร้อน 10 วัน/ปี..."},
#   {id:2, title:"ลาป่วย",    content:"ลาป่วย 30 วัน/ปี..."},
#   {id:3, title:"ลากิจ",     content:"ลากิจ 5 วัน/ปี..."},
# ]
# → ส่งทั้ง 3 chunks ให้ LLM → ตอบครบทุกประเภทการลา ✅


# User: "ลาพักร้อนต้องแจ้งล่วงหน้ากี่วัน"
# Search → hit chunk "ลาพักร้อน" (id=1)

# Strategy: neighbors → ดึง chunk ก่อน/หลัง
context_chunks = await get_enriched_context(
    hit_chunk_id=1, 
    strategy="neighbors",
    window_size=1
)
# ได้ chunk 1 + chunk 2 (เผื่อมีข้อมูลเพิ่มเติม)
```

### 5.4 Smart Context Assembly

```python
async def smart_context(question: str, search_results: list[dict]) -> str:
    """
    ตัดสินใจว่าจะดึง context เพิ่มหรือไม่ ขึ้นกับคำถาม
    """
    
    # วิเคราะห์คำถาม
    is_summary = any(kw in question for kw in ["สรุป", "ทั้งหมด", "รวม", "ทุก"])
    is_specific = any(kw in question for kw in ["กี่วัน", "เท่าไหร่", "ยังไง"])
    
    all_chunks = []
    
    for result in search_results:
        if is_summary:
            # คำถามกว้าง → ดึงทั้ง section
            chunks = await get_enriched_context(
                result["id"], strategy="section"
            )
        elif is_specific:
            # คำถามเฉพาะ → ดึงแค่ chunk เดียว + neighbors
            chunks = await get_enriched_context(
                result["id"], strategy="neighbors", window_size=1
            )
        else:
            chunks = [result]
        
        all_chunks.extend(chunks)
    
    # Deduplicate (อาจมี chunk ซ้ำจากหลาย search results)
    seen_ids = set()
    unique_chunks = []
    for chunk in all_chunks:
        if chunk["id"] not in seen_ids:
            seen_ids.add(chunk["id"])
            unique_chunks.append(chunk)
    
    # สร้าง context string
    context = "\n\n".join([
        f"[{c.get('title', 'N/A')}]\n{c.get('summary') or c['content']}"
        for c in unique_chunks
    ])
    
    return context
```

---

## 6. ปัจจุบัน 2025 เค้าทำกันยังไง

### Trends ปัจจุบัน

```
1. Hybrid Chunking เป็น default
   ไม่ใช้วิธีเดียว → ผสมหลายวิธี
   เช่น: Header-based (หลัก) + Recursive (fallback ถ้า section ยาว)
   
2. Chunk + Summary เป็นมาตรฐาน
   ทุก chunk เก็บทั้ง full content + summary
   search/context ใช้ summary → ลด tokens 40-60%
   ต้อง detail ค่อยดึง full content

3. Parent-Child ได้รับความนิยม
   LlamaIndex, LangChain มี built-in support
   ดีกว่า overlap แบบเก่า

4. Diff-based incremental indexing
   ไม่มีใคร re-index ทุกครั้ง
   hash-based diff → embed เฉพาะที่เปลี่ยน

5. Contextual Retrieval (Anthropic, 2024)
   ก่อน embed chunk → prepend context:
   "เอกสาร 'คู่มือพนักงาน' section 'นโยบายการลา' > ลาพักร้อน:
    พนักงานประจำลาพักร้อนได้ 10 วัน/ปี..."
   
   embedding จะ "รู้" ว่า chunk นี้มาจากไหน → search แม่นขึ้น 20-50%

6. Late Chunking เริ่มมีคนใช้
   ยังใหม่ แต่ผลดีมาก
   จะเป็น standard ในอนาคต

7. Proposition Chunking
   แยก facts เป็น propositions แต่ละอัน → embed
   "ลาพักร้อน 10 วัน ต้องแจ้ง 3 วัน"
   → ["พนักงานลาพักร้อนได้ 10 วัน/ปี",
      "การลาพักร้อนต้องแจ้งล่วงหน้า 3 วัน"]
   
   ดีมากสำหรับ factual Q&A
```

### Contextual Retrieval (Anthropic's approach)

```python
# ก่อน embed → เพิ่ม context header ให้ chunk

async def add_contextual_header(chunk: dict, full_document: str) -> str:
    """
    Anthropic's Contextual Retrieval technique
    ให้ LLM สร้าง "context header" สำหรับแต่ละ chunk
    เพิ่มก่อน embed เพื่อให้ embedding รู้บริบท
    """
    response = await llm.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": """ให้ context สั้นๆ (1-2 ประโยค) ว่า chunk นี้
                อยู่ในบริบทอะไรของเอกสาร เพื่อให้เข้าใจ chunk ได้ดีขึ้น
                ตอบแค่ context ไม่ต้องอธิบาย"""
            },
            {
                "role": "user",
                "content": f"""เอกสาร:
{full_document[:3000]}

Chunk:
{chunk['content']}"""
            }
        ],
        max_tokens=100,
    )
    
    context_header = response.choices[0].message.content
    return f"{context_header}\n\n{chunk['content']}"


# ก่อน:
#   content = "ต้องแจ้งล่วงหน้า 3 วัน หัวหน้าอนุมัติ"
#   embedding ไม่รู้ว่าพูดถึงอะไร

# หลัง (Contextual Retrieval):
#   content = "ข้อมูลนี้อยู่ในส่วนนโยบายลาพักร้อนของคู่มือพนักงาน Sellsuki 2024
#              กล่าวถึงเงื่อนไขการลาพักร้อน\n\n
#              ต้องแจ้งล่วงหน้า 3 วัน หัวหน้าอนุมัติ"
#   embedding "รู้" ว่าพูดถึงลาพักร้อน → search แม่นขึ้น
```

---

## 7. เลือกวิธีไหน

### Decision Matrix

```
┌──────────────────┬─────────┬─────────┬──────────┬──────────┬──────────┐
│ วิธี              │ คุณภาพ   │ ง่าย     │ เร็ว      │ ถูก       │ ภาษาไทย  │
├──────────────────┼─────────┼─────────┼──────────┼──────────┼──────────┤
│ Fixed Size       │ ★★☆☆☆  │ ★★★★★  │ ★★★★★   │ ★★★★★   │ ★★★★☆   │
│ Recursive        │ ★★★★☆  │ ★★★★☆  │ ★★★★★   │ ★★★★★   │ ★★★★☆   │
│ Header-based     │ ★★★★★  │ ★★★★☆  │ ★★★★★   │ ★★★★★   │ ★★★★★   │
│ Hierarchical     │ ★★★★★  │ ★★★☆☆  │ ★★★★☆   │ ★★★★★   │ ★★★★★   │
│ Sentence         │ ★★★☆☆  │ ★★★★☆  │ ★★★★★   │ ★★★★★   │ ★★☆☆☆   │
│ Sentence Window  │ ★★★★☆  │ ★★★☆☆  │ ★★★★☆   │ ★★★★☆   │ ★★☆☆☆   │
│ Semantic         │ ★★★★★  │ ★★☆☆☆  │ ★★★☆☆   │ ★★★☆☆   │ ★★★☆☆   │
│ Parent-Child     │ ★★★★★  │ ★★★☆☆  │ ★★★★☆   │ ★★★★☆   │ ★★★★★   │
│ Agentic          │ ★★★★★+ │ ★☆☆☆☆  │ ★☆☆☆☆   │ ★☆☆☆☆   │ ★★★★☆   │
│ Late Chunking    │ ★★★★★+ │ ★★☆☆☆  │ ★★★☆☆   │ ★★★☆☆   │ ★★★☆☆   │
└──────────────────┴─────────┴─────────┴──────────┴──────────┴──────────┘

เอกสารมี structure (Markdown/HTML/Docs):
  → Header-based + Hierarchical ✅

เอกสาร plain text (meeting notes, emails):
  → Recursive ✅ หรือ Semantic (ถ้ามี budget)

FAQ / Q&A:
  → ไม่ต้อง chunk — 1 Q&A = 1 chunk ✅

Documentation (Sellsuki docs, wiki):
  → Hierarchical + Parent-Child + Contextual Headers ✅

สัญญา / กฎหมาย:
  → Agentic chunking + Parent-Child ✅

Budget จำกัดมาก:
  → Recursive + Summary ✅

ต้องการดีที่สุด:
  → Hierarchical + Parent-Child + Contextual + Summary ✅
```

### สำหรับ Sellsuki Agent: แนะนำ

```
Combination ที่แนะนำ:

1. Primary: Header-based Hierarchical
   เอกสารส่วนใหญ่ (policy, manual) มี headers
   → ตัดตาม headers ก่อน → sub-chunk ถ้ายาว

2. + Parent-Child (สำหรับ sections ยาว)
   เก็บ parent (full section) + children (sub-chunks)
   search children → ดึง parent

3. + Summary ทุก chunk
   สร้าง summary ตอน index → ใช้ตอน query

4. + Contextual Header
   prepend context ก่อน embed → search แม่นขึ้น

5. + Linked List (prev/next)
   เชื่อม chunks → ดึงบริบทรอบข้างได้

6. FAQ ไม่ต้อง chunk
   1 Q&A = 1 chunk

7. Diff-based indexing
   hash ทุก chunk → re-index เฉพาะที่เปลี่ยน
```

---

## 8. Common Mistakes

```
❌ Mistake 1: Chunk ใหญ่เกินไป (> 2000 chars)
   → embedding เบลอ ค้นหาไม่แม่น
   ✅ Fix: 500-1000 chars

❌ Mistake 2: Chunk เล็กเกินไป (< 100 chars)
   → ไม่มีบริบท "ต้องแจ้ง 3 วัน" (แจ้งอะไร?)
   ✅ Fix: อย่างน้อย 200 chars หรือ 1 paragraph

❌ Mistake 3: ตัดกลางประโยค
   → "พนักงานมีสิทธิ์ลาพักร้อ|นปีละ 10 วัน"
   ✅ Fix: ใช้ recursive splitting (ตัดที่ paragraph/sentence)

❌ Mistake 4: ไม่มี overlap
   → ข้อมูลหายตรงรอยตัด
   ✅ Fix: overlap 15-25% ของ chunk_size

❌ Mistake 5: ไม่เก็บ metadata
   → ไม่รู้ว่า chunk มาจากไหน filter ไม่ได้
   ✅ Fix: เก็บ source, category, title, headers

❌ Mistake 6: Re-index ทั้งหมดทุกครั้ง
   → เสีย embedding cost โดยไม่จำเป็น
   ✅ Fix: content_hash + diff-based

❌ Mistake 7: ไม่ทดสอบ
   → ไม่รู้ว่า chunk strategy ดีแค่ไหน
   ✅ Fix: สร้าง eval set 50 คำถาม + ทดสอบ search quality

❌ Mistake 8: ใช้วิธีเดียวกับทุก document
   → FAQ ไม่ควร chunk เหมือน manual ยาวๆ
   ✅ Fix: เลือก strategy ตามประเภทเอกสาร

❌ Mistake 9: Chunk แล้วไม่เชื่อม
   → ถามเรื่องกว้าง ตอบได้ไม่ครบ
   ✅ Fix: linked list + section grouping

❌ Mistake 10: ไม่ทำ summary
   → ส่ง full content ให้ LLM ทุกครั้ง เปลือง tokens
   ✅ Fix: summary ตอน index → ใช้ summary ตอน query
```
