---
title: "OpenRAG — Document Update Guide"
type: source
source_file: raw/notes/openrag/docs-lean/data-update-guide.md
published: 2026-03-17
tags: [openrag, document-update, hash, deduplication, weekly-update]
related: [wiki/concepts/openrag-platform.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/openrag/docs-lean/data-update-guide.md|Original file]]

## สรุป

คู่มือการ update ข้อมูลใน OpenRAG — อธิบาย document ID system, 4 update scenarios, weekly workflow และข้อควรระวัง

## ประเด็นสำคัญ

### Document ID System (SHA256 Hash)

```
document_id = SHA256(file content) → base64 → ตัด 24 ตัวอักษร
```

- ไฟล์เดิมเนื้อหาเดิม → hash เหมือนเดิม → **ระบบ skip ทันที**
- ไฟล์ชื่อเดิมแต่เนื้อหาใหม่ → hash ต่าง → **ลบเก่า + index ใหม่**
- ไฟล์ชื่อใหม่เนื้อหาเดิม → hash เหมือน → **ระบบ skip**

chunk ID: `{file_hash}_{chunk_number}` เช่น `a1b2c3d4_0`

### Update Mechanism: Delete All → Re-index All

**ไม่มี partial update** — ทุกครั้งที่ไฟล์เปลี่ยน ระบบจะ:
1. `check_document_exists(hash)` — ถ้า hash เหมือนเดิม stop
2. `check_filename_exists(filename)` — ถ้าพบ ลบ chunks เก่าทั้งหมด
3. `delete_document_by_filename()` — ลบด้วย `delete_by_query`
4. Process + embed + index ใหม่ทั้งไฟล์

ตำแหน่ง code: `src/models/processors.py` lines 730–740

### replace_duplicates Parameter

```python
# Default: True (แนะนำ)
status = await client.documents.ingest("report.pdf")
# → ลบเก่าอัตโนมัติ แล้ว index ใหม่

# False: ป้องกันการเขียนทับ → throw Error ถ้าชื่อซ้ำ
```

### 4 Update Scenarios

| Scenario | วิธีทำ | ผลลัพธ์ |
|----------|--------|---------|
| Update ไฟล์เดิม (เนื้อหาเปลี่ยน) | อัปโหลดชื่อเดิมทับ | ลบเก่า + index ใหม่ |
| เพิ่มไฟล์ใหม่ (ไม่แทนที่เก่า) | อัปโหลดชื่อใหม่ | index เพิ่มเข้าไป |
| ลบเก่า + เพิ่มใหม่ (เปลี่ยนชื่อ) | delete เก่า → ingest ใหม่ | มีแค่ไฟล์ใหม่ |
| Bulk update หลายไฟล์ | loop อัปโหลดทีละไฟล์ | อัปเดตทีละไฟล์ |

### ข้อควรระวัง

- **อย่าเปลี่ยนชื่อไฟล์โดยไม่ตั้งใจ** → ของเก่ายังอยู่ใน index ข้อมูลปน
- **ถ้าไฟล์มี 100 หน้า แก้แค่หน้าเดียว** → ต้อง embed ใหม่ทั้ง 100 หน้า
- **แนะนำ**: แยกไฟล์ตาม section ที่ update บ่อย → ประหยัด embedding cost

### Weekly Automation

```bash
# Cron Job
0 2 * * 1 python weekly_update.py

# Python SDK
status = await client.documents.ingest("weekly_report.pdf")
# status: "completed" | "unchanged" | "failed"
```

## ข้อมูลที่น่าสนใจ

- ถ้าไฟล์เนื้อหาไม่เปลี่ยน → `status.status = "unchanged"` → ไม่เสีย API call embedding เลย
- Langflow storage ไม่โตเพราะลบไฟล์หลัง ingest อัตโนมัติ (delete_after_ingest=True)
- ไฟล์ PDF 10MB → OpenSearch เก็บแค่ ~2-5MB (text chunks + vectors)

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/openrag-platform|OpenRAG Platform]] — ระบบ indexing และ document management
