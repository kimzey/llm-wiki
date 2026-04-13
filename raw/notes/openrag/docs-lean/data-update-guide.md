# คู่มือการ Update ข้อมูลใน OpenRAG

> ตอบคำถาม: ต้องทำอะไรเมื่อข้อมูลต้อง update ทุกสัปดาห์, ต้องเคลียของเก่ามั้ย, ต้อง embed ใหม่ทั้งหมดมั้ย, data เก่า-ใหม่จะปนกันมั้ย

---

## สารบัญ

1. [ทำความเข้าใจระบบก่อน](#1-ทำความเข้าใจระบบก่อน)
2. [ระบบระบุเอกสาร — ใช้อะไรเป็น ID?](#2-ระบบระบุเอกสาร--ใช้อะไรเป็น-id)
3. [ตอบทุกคำถาม](#3-ตอบทุกคำถาม)
4. [Scenarios การ Update 4 แบบ](#4-scenarios-การ-update-4-แบบ)
5. [Weekly Update Workflow (วิธีแนะนำ)](#5-weekly-update-workflow-วิธีแนะนำ)
6. [ตัวอย่าง Code จริง](#6-ตัวอย่าง-code-จริง)
7. [Automation Script](#7-automation-script)
8. [สิ่งที่ระวัง](#8-สิ่งที่ระวัง)

---

## 1. ทำความเข้าใจระบบก่อน

### ระบบ Index ของ OpenRAG

เมื่อ ingest ไฟล์ 1 ไฟล์ ระบบจะเก็บไว้ใน OpenSearch แบบนี้:

```
ไฟล์ report.pdf (10 หน้า)
    │
    ▼
document_id: "a1b2c3d4..."  ← SHA256 hash ของ content ไฟล์
    │
    ├── chunk ID: "a1b2c3d4_0"   (หน้า 1, text)
    ├── chunk ID: "a1b2c3d4_1"   (หน้า 1, table)
    ├── chunk ID: "a1b2c3d4_2"   (หน้า 2, text)
    ├── chunk ID: "a1b2c3d4_3"   (หน้า 3, text)
    ...
    └── chunk ID: "a1b2c3d4_22"  (หน้า 10, text)
```

**แต่ละ chunk เก็บ field เหล่านี้:**

```json
{
  "_id": "a1b2c3d4_0",
  "document_id": "a1b2c3d4...",
  "filename": "report.pdf",
  "page": 1,
  "text": "เนื้อหาหน้า 1...",
  "chunk_embedding_text_embedding_3_small": [0.012, -0.045, ...],
  "embedding_model": "text-embedding-3-small",
  "indexed_time": "2026-03-17T10:00:00Z",
  "owner": "user_abc"
}
```

**สิ่งสำคัญ**: ทุก chunk ของไฟล์เดียวกันจะมี `filename` และ `document_id` เหมือนกัน — นี่คือ key สำคัญสำหรับการ delete และ update

---

## 2. ระบบระบุเอกสาร — ใช้อะไรเป็น ID?

OpenRAG ใช้ **2 ระดับ** ในการระบุเอกสาร:

### ระดับที่ 1: Content Hash (SHA256)

```
document_id = SHA256(เนื้อหาของไฟล์) → base64 → ตัด 24 ตัวอักษร
```

- **ไฟล์เดิม เนื้อหาเดิม** → hash เหมือนเดิมทุกครั้ง
- **ไฟล์ใหม่ เนื้อหาเปลี่ยน** → hash ต่างออกไป

ตำแหน่ง code: `src/utils/hash_utils.py`

### ระดับที่ 2: Filename (metadata field)

- `filename` เก็บเป็น field ธรรมดาใน OpenSearch
- ใช้สำหรับ **ค้นหา** และ **ลบ** ทีหลัง
- ไม่ใช่ primary key — แค่ text field ที่ query ได้

### ผลลัพธ์จริงที่ต้องรู้

| สถานการณ์ | Hash | ผลที่เกิด |
|-----------|------|----------|
| อัปโหลดไฟล์เดิม (ไม่เปลี่ยนเนื้อหา) | เหมือนเดิม | **ข้ามทันที** ไม่ index ซ้ำ |
| อัปโหลดไฟล์ชื่อเดิม แต่เนื้อหาใหม่ | ต่างออกไป | **ลบเก่า → index ใหม่** |
| อัปโหลดไฟล์ชื่อใหม่ เนื้อหาเดิม | เหมือนเดิม | **ข้าม** เพราะ hash ซ้ำ |
| อัปโหลดไฟล์ใหม่ทั้งหมด | ต่างออกไป | **Index ใหม่ปกติ** |

---

## 3. ตอบทุกคำถาม

### ❓ ต้องเคลียของเก่ามั้ย?

**ตอบ: ไม่ต้องทำเอง** — ระบบจัดการให้อัตโนมัติ เมื่ออัปโหลดไฟล์ชื่อเดิม:

```
ระบบทำอัตโนมัติ:
1. ตรวจว่า filename นี้มีอยู่แล้วมั้ย?
2. ถ้ามี → ลบ chunks เก่าทั้งหมดก่อน (delete_by_query)
3. index chunks ใหม่ทั้งหมด
```

ควบคุมได้ด้วย parameter `replace_duplicates`:

```python
# replace_duplicates=True (DEFAULT) — แนะนำ สำหรับ weekly update
# → ลบเก่า แล้ว index ใหม่อัตโนมัติ
status = await client.documents.ingest("report_week12.pdf")

# replace_duplicates=False — ป้องกันการเขียนทับโดยไม่ตั้งใจ
# → ถ้าชื่อซ้ำ = Error (ต้องลบด้วยตนเองก่อน)
```

---

### ❓ มันจะ Update ยังไง?

**ตอบ: ไม่มี "partial update"** — ระบบทำแบบ **Delete All → Re-index All** เสมอ

```
ไม่มีการ merge หรืออัพเดทบางส่วน

เมื่ออัปโหลดไฟล์ชื่อเดิม:
BEFORE: report.pdf → 23 chunks (เนื้อหาเก่า)
         ↓ delete_by_query (filename = "report.pdf") → ลบ 23 chunks
AFTER:  report.pdf → 31 chunks (เนื้อหาใหม่)
```

**ขั้นตอนจริงใน code:**

```
1. check_document_exists(hash)         → hash ต่างออกไป ผ่าน
2. check_filename_exists("report.pdf") → พบไฟล์เก่า
3. delete_document_by_filename("report.pdf") → ลบ 23 chunks เก่า
4. process_text / docling chunking     → แบ่ง chunk ใหม่
5. embedding API call                  → embed ทุก chunk ใหม่
6. opensearch.index() x31             → index 31 chunks ใหม่
```

ตำแหน่ง code: `src/models/processors.py` lines 730–740

---

### ❓ ต้อง Embed ใหม่ทุกอันเลยมั้ย?

**ตอบ: ขึ้นกับว่าเนื้อหาเปลี่ยนมั้ย**

```
Case A: เนื้อหาไม่เปลี่ยน (แค่ชื่อไฟล์ต่าง)
→ Hash เหมือนเดิม → ระบบ SKIP ทันที → ไม่ embed ซ้ำ ✓

Case B: เนื้อหาเปลี่ยนบางส่วน (เช่น แก้ไข 2 หน้า)
→ Hash ต่าง → ลบทั้งหมด → Embed ใหม่ทั้งไฟล์ (ทุก chunk)
→ ไม่มีระบบ partial re-embed ✗

Case C: ไฟล์ใหม่ทั้งหมด
→ Hash ใหม่ → Embed ใหม่ทั้งหมดตามปกติ ✓
```

**สรุป**: ถ้าไฟล์เปลี่ยน แม้แต่ตัวเดียว = embed ใหม่ทุก chunk ของไฟล์นั้น ไม่มีวิธีเลี่ยง เพราะ OpenRAG ไม่รองรับ partial update

---

### ❓ Data เก่ากับใหม่จะปนกันมั้ย?

**ตอบ: ไม่ปน** ถ้าใช้ชื่อไฟล์เดิม

```
ใช้ชื่อเดิม "report.pdf":
→ ลบเก่าก่อน → index ใหม่ → ไม่ปน ✓

แต่ถ้าเปลี่ยนชื่อ "report_v1.pdf" → "report_v2.pdf":
→ report_v1.pdf ยังอยู่ใน index
→ report_v2.pdf ถูก index เพิ่ม
→ ข้อมูลปน ✗ ต้องลบ report_v1.pdf เองก่อน
```

**กฎง่ายๆ: ใช้ชื่อไฟล์เดิมเสมอเมื่อ update** — ระบบจะจัดการให้

---

### ❓ การทำ Index เป็นยังไง?

ทุก chunk ถูก index เป็น document แยกใน OpenSearch:

```
รูปแบบ index name: "documents" (default)

รูปแบบ document ID: "{content_hash}_{chunk_number}"
ตัวอย่าง: "a1b2c3d4e5f6_0", "a1b2c3d4e5f6_1", ...

การ index chunk ใหม่ = opensearch.index(id=chunk_id, body=chunk_doc)
ถ้า ID ซ้ำ → overwrite (แต่ทำ delete ก่อนอยู่แล้ว)
```

**ไม่มีการ "update" document ใน OpenSearch** — มีแค่ delete แล้ว index ใหม่เท่านั้น

---

## 4. Scenarios การ Update 4 แบบ

### Scenario A: Update ไฟล์เดิม (เนื้อหาเปลี่ยน)

> ใช้บ่อยสุด — เช่น รายงานประจำสัปดาห์, ข้อมูล spec ที่แก้ไข

```
weekly_report.pdf (v1, 1 Jan)
          ↓ แก้ไขเนื้อหา
weekly_report.pdf (v2, 8 Jan)
```

**สิ่งที่ต้องทำ**: อัปโหลดไฟล์ชื่อเดิมทับ

```python
# Python SDK
status = await client.documents.ingest("weekly_report.pdf")
# ระบบจะ: ลบ v1 ทั้งหมด → embed v2 ใหม่ → index v2

print(f"Deleted old + Indexed new: {status.successful_files} files")
```

```typescript
// TypeScript SDK
const status = await client.documents.ingest({ filePath: "./weekly_report.pdf" });
// ชื่อเดิม → replace_duplicates=true → auto-replace
```

**ผลลัพธ์**: ข้อมูลใหม่ทั้งหมด ไม่มีข้อมูลเก่าหลงเหลือ ✓

---

### Scenario B: เพิ่มไฟล์ใหม่ (ไม่แทนที่เก่า)

> เช่น เพิ่ม product sheet รายการใหม่ทุกสัปดาห์

```
products_jan.pdf  ← มีอยู่แล้ว
products_feb.pdf  ← เพิ่มใหม่
```

**สิ่งที่ต้องทำ**: อัปโหลดไฟล์ชื่อใหม่ปกติ

```python
status = await client.documents.ingest("products_feb.pdf")
# ชื่อใหม่ → ไม่มี conflict → index เพิ่มเข้าไป
```

**ผลลัพธ์**: ทั้งสองไฟล์อยู่ใน index พร้อมกัน Search จะเจอทั้งคู่ ✓

---

### Scenario C: ลบไฟล์เก่า + เพิ่มไฟล์ใหม่

> เช่น เปลี่ยนชื่อไฟล์ตาม version หรือ period

```
report_2026_Q1.pdf  ← ต้องการเอาออก
report_2026_Q2.pdf  ← เพิ่มใหม่
```

**สิ่งที่ต้องทำ**: ลบเก่าก่อน แล้ว ingest ใหม่

```python
# Step 1: ลบไฟล์เก่า
result = await client.documents.delete("report_2026_Q1.pdf")
print(f"Deleted {result.deleted_chunks} chunks")

# Step 2: ingest ใหม่
status = await client.documents.ingest("report_2026_Q2.pdf")
print(f"Indexed: {status.successful_files} files")
```

**ผลลัพธ์**: มีแค่ Q2 ใน index ✓

---

### Scenario D: Update หลายไฟล์พร้อมกัน (Bulk Update)

> เช่น ทุกวันจันทร์ update ไฟล์ทั้งหมด 20 ไฟล์

**สิ่งที่ต้องทำ**: loop อัปโหลดทีละไฟล์ (ชื่อเดิมทั้งหมด)

```python
import asyncio
from pathlib import Path

async def weekly_bulk_update(client, directory: str):
    files = list(Path(directory).glob("*.pdf"))
    print(f"อัปโหลด {len(files)} ไฟล์...")

    for i, file_path in enumerate(files, 1):
        print(f"[{i}/{len(files)}] {file_path.name}...")
        status = await client.documents.ingest(file_path)
        if status.status == "completed":
            print(f"  ✓ OK")
        else:
            print(f"  ✗ ล้มเหลว: {status.status}")

    print("เสร็จ!")
```

---

## 5. Weekly Update Workflow (วิธีแนะนำ)

### ภาพรวม Workflow

```
ทุกสัปดาห์ (เช่น ทุกวันจันทร์ 02:00 AM)
        │
        ▼
┌─────────────────────────────────┐
│  Step 1: เตรียมไฟล์ใหม่       │
│  - รวบรวมไฟล์ที่ต้อง update   │
│  - ใช้ชื่อไฟล์เดิมเสมอ        │
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│  Step 2: อัปโหลดทีละไฟล์      │
│  - ingest(file, wait=True)     │
│  - ระบบลบเก่า+index ใหม่เอง   │
│  - ตรวจสอบ status ทุกไฟล์     │
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│  Step 3: (Optional) ลบไฟล์    │
│  ที่ไม่ต้องการแล้วออก          │
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│  Step 4: ตรวจสอบผลลัพธ์       │
│  - search test query           │
│  - ดู sources ว่า timestamp ใหม่│
└─────────────────────────────────┘
```

### ประเภทของข้อมูลและกลยุทธ์ที่เหมาะสม

| ประเภทข้อมูล | กลยุทธ์ | ชื่อไฟล์ |
|-------------|---------|---------|
| รายงานรายสัปดาห์ | อัปโหลดทับ | `weekly_report.pdf` (ชื่อเดิม) |
| ข้อมูล spec / manual | อัปโหลดทับ | `product_manual.pdf` (ชื่อเดิม) |
| ข่าวสาร/บทความ | เพิ่มใหม่ | `news_2026_w12.pdf` (ชื่อใหม่ตาม week) |
| ข้อมูลสะสม (archive) | เพิ่มใหม่ | `archive_march_2026.pdf` |
| ข้อมูลที่ retire แล้ว | ลบออก | ใช้ `client.documents.delete()` |

---

## 6. ตัวอย่าง Code จริง

### Python: Weekly Update Script

```python
import asyncio
import os
from pathlib import Path
from datetime import datetime
from openrag_sdk import OpenRAGClient
from openrag_sdk.exceptions import OpenRAGError


async def weekly_update(
    update_dir: str,      # โฟลเดอร์ไฟล์ใหม่
    delete_files: list[str] = None,  # ไฟล์ที่ต้องการลบออก
):
    """
    Weekly update routine สำหรับ OpenRAG

    - ไฟล์ในโฟลเดอร์ update_dir จะถูก ingest (ชื่อเดิม = แทนที่อัตโนมัติ)
    - ไฟล์ใน delete_files จะถูกลบออกจาก index
    """
    results = {"updated": [], "deleted": [], "skipped": [], "failed": []}

    async with OpenRAGClient(
        api_key=os.environ["OPENRAG_API_KEY"],
        base_url=os.environ.get("OPENRAG_URL", "http://localhost:3000"),
    ) as client:

        # --- Step 1: ลบไฟล์ที่ไม่ต้องการ ---
        if delete_files:
            print(f"\n[1/3] ลบไฟล์ที่ไม่ต้องการ ({len(delete_files)} ไฟล์)...")
            for filename in delete_files:
                try:
                    result = await client.documents.delete(filename)
                    if result.success:
                        print(f"  ✓ ลบ {filename} ({result.deleted_chunks} chunks)")
                        results["deleted"].append(filename)
                    else:
                        print(f"  ? {filename} ไม่พบ (อาจลบไปแล้ว)")
                except OpenRAGError as e:
                    print(f"  ✗ ลบ {filename} ล้มเหลว: {e}")
                    results["failed"].append({"file": filename, "error": str(e)})

        # --- Step 2: อัปโหลดไฟล์ใหม่ ---
        files = sorted(Path(update_dir).glob("**/*"))
        files = [f for f in files if f.is_file() and f.suffix in
                 {".pdf", ".docx", ".txt", ".md", ".pptx", ".xlsx"}]

        print(f"\n[2/3] อัปโหลดไฟล์ใหม่ ({len(files)} ไฟล์)...")
        for i, file_path in enumerate(files, 1):
            print(f"  [{i}/{len(files)}] {file_path.name}...", end=" ")
            try:
                status = await client.documents.ingest(
                    file_path,
                    poll_interval=2.0,
                    timeout=300.0,
                )
                if status.status == "completed":
                    print(f"✓ ({status.successful_files} indexed)")
                    results["updated"].append(file_path.name)
                elif status.status == "failed":
                    print(f"✗ ล้มเหลว")
                    results["failed"].append({"file": file_path.name, "error": "ingest failed"})
                else:
                    # "unchanged" = เนื้อหาเหมือนเดิม ระบบ skip
                    print(f"~ ข้าม (ไม่มีการเปลี่ยนแปลง)")
                    results["skipped"].append(file_path.name)
            except OpenRAGError as e:
                print(f"✗ Error: {e}")
                results["failed"].append({"file": file_path.name, "error": str(e)})

        # --- Step 3: สรุปผล ---
        print(f"\n[3/3] สรุป:")
        print(f"  ✓ Updated  : {len(results['updated'])} ไฟล์")
        print(f"  ✓ Deleted  : {len(results['deleted'])} ไฟล์")
        print(f"  ~ Skipped  : {len(results['skipped'])} ไฟล์ (เนื้อหาเหมือนเดิม)")
        print(f"  ✗ Failed   : {len(results['failed'])} ไฟล์")
        if results["failed"]:
            for f in results["failed"]:
                print(f"     - {f['file']}: {f['error']}")

        return results


# ตัวอย่างการเรียกใช้
asyncio.run(weekly_update(
    update_dir="./data/weekly",
    delete_files=["old_policy_v1.pdf", "deprecated_guide.pdf"],
))
```

---

### TypeScript: Weekly Update Script

```typescript
import { OpenRAGClient, OpenRAGError } from "openrag-sdk";
import { readdirSync, statSync } from "fs";
import { join, extname, basename } from "path";

const SUPPORTED_EXTENSIONS = new Set([
  ".pdf", ".docx", ".txt", ".md", ".pptx", ".xlsx",
]);

async function weeklyUpdate(
  updateDir: string,
  deleteFiles: string[] = []
) {
  const client = new OpenRAGClient();
  const results = { updated: [], deleted: [], skipped: [], failed: [] } as Record<string, string[]>;

  try {
    // Step 1: ลบไฟล์ที่ไม่ต้องการ
    if (deleteFiles.length > 0) {
      console.log(`\n[1/3] ลบไฟล์ที่ไม่ต้องการ (${deleteFiles.length} ไฟล์)...`);
      for (const filename of deleteFiles) {
        try {
          const result = await client.documents.delete(filename);
          if (result.success) {
            console.log(`  ✓ ลบ ${filename} (${result.deleted_chunks} chunks)`);
            results.deleted.push(filename);
          }
        } catch (e) {
          console.error(`  ✗ ลบ ${filename} ล้มเหลว:`, e);
          results.failed.push(filename);
        }
      }
    }

    // Step 2: อัปโหลดไฟล์ใหม่
    const files = readdirSync(updateDir)
      .filter((f) => SUPPORTED_EXTENSIONS.has(extname(f).toLowerCase()))
      .filter((f) => statSync(join(updateDir, f)).isFile());

    console.log(`\n[2/3] อัปโหลดไฟล์ใหม่ (${files.length} ไฟล์)...`);
    for (let i = 0; i < files.length; i++) {
      const filename = files[i];
      const filePath = join(updateDir, filename);
      process.stdout.write(`  [${i + 1}/${files.length}] ${filename}... `);

      try {
        const status = await client.documents.ingest({ filePath });
        if (status.status === "completed") {
          console.log(`✓`);
          results.updated.push(filename);
        } else {
          console.log(`✗ ${status.status}`);
          results.failed.push(filename);
        }
      } catch (e) {
        console.log(`✗ Error: ${e}`);
        results.failed.push(filename);
      }
    }

    // Step 3: สรุป
    console.log(`\n[3/3] สรุป:`);
    console.log(`  ✓ Updated : ${results.updated.length} ไฟล์`);
    console.log(`  ✓ Deleted : ${results.deleted.length} ไฟล์`);
    console.log(`  ✗ Failed  : ${results.failed.length} ไฟล์`);
    if (results.failed.length > 0) {
      results.failed.forEach((f) => console.log(`     - ${f}`));
    }

    return results;
  } finally {
    // ไม่มี close สำหรับ TS client
  }
}

// Usage
weeklyUpdate("./data/weekly", ["old_policy.pdf"]);
```

---

## 7. Automation Script

### ตั้ง Cron Job (Linux/macOS)

```bash
# เพิ่มใน crontab (crontab -e)
# ทำงานทุกวันจันทร์ เวลา 02:00 AM
0 2 * * 1 /usr/bin/python3 /opt/openrag/scripts/weekly_update.py >> /var/log/openrag_update.log 2>&1
```

### Docker-based Schedule (docker-compose.yml)

```yaml
services:
  openrag-updater:
    image: python:3.11-slim
    environment:
      - OPENRAG_API_KEY=${OPENRAG_API_KEY}
      - OPENRAG_URL=http://openrag:3000
    volumes:
      - ./data/weekly:/data/weekly:ro
      - ./scripts:/scripts:ro
    command: >
      sh -c "pip install openrag-sdk &&
             while true; do
               python /scripts/weekly_update.py;
               sleep 604800;
             done"
    depends_on:
      - openrag
```

### REST API โดยตรง (ไม่ใช้ SDK)

```bash
#!/bin/bash
# weekly_update.sh

OPENRAG_URL="http://localhost:3000"
API_KEY="orag_your_key_here"
UPDATE_DIR="./data/weekly"

echo "=== Weekly OpenRAG Update ==="
echo "$(date)"

for file in "$UPDATE_DIR"/*.pdf; do
  filename=$(basename "$file")
  echo -n "Uploading $filename... "

  # Upload ไฟล์
  response=$(curl -s -X POST \
    "$OPENRAG_URL/api/v1/documents/ingest" \
    -H "X-API-Key: $API_KEY" \
    -F "file=@$file")

  task_id=$(echo "$response" | python3 -c "import sys,json; print(json.load(sys.stdin)['task_id'])")
  echo "task=$task_id"

  # รอ จนเสร็จ
  for i in $(seq 1 60); do
    sleep 5
    status=$(curl -s "$OPENRAG_URL/api/v1/tasks/$task_id" \
      -H "X-API-Key: $API_KEY" | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])")
    if [ "$status" = "completed" ] || [ "$status" = "failed" ]; then
      echo "  → $status"
      break
    fi
  done
done

echo "=== Done ==="
```

---

## 8. สิ่งที่ระวัง

### ⚠️ อย่าเปลี่ยนชื่อไฟล์โดยไม่ตั้งใจ

```
✗ BAD: อัปโหลด report_week12_FINAL.pdf  ← ชื่อใหม่
       → ของเก่า report_week12.pdf ยังอยู่ใน index
       → ข้อมูลปน มี 2 เวอร์ชัน

✓ GOOD: อัปโหลด report_week12.pdf  ← ชื่อเดิม
        → ของเก่าถูกลบ, ของใหม่ถูก index
```

### ⚠️ เนื้อหาเหมือนเดิม = ไม่ embed ใหม่ (ประหยัด cost)

```python
# ถ้าไฟล์ไม่มีการเปลี่ยนแปลงจริงๆ
status = await client.documents.ingest("same_content.pdf")
# → ระบบตรวจ hash → เหมือนเดิม → skip → ไม่เสีย API call embedding
# status.status จะ return "unchanged" ไม่ใช่ "completed"
```

### ⚠️ ไม่มี Partial Update

```
ถ้าไฟล์มี 100 หน้า แก้แค่หน้า 1 หน้า
→ ต้อง embed ใหม่ทั้ง 100 หน้า
→ ไม่มีทางเลี่ยง เพราะ hash ของทั้งไฟล์เปลี่ยน

เทคนิคแก้: แยกเอกสารเป็นไฟล์เล็กๆ ตาม section
- section_intro.pdf    (อัปเดตน้อย)
- section_products.pdf (อัปเดตบ่อย)
- section_pricing.pdf  (อัปเดตทุกสัปดาห์)
→ ประหยัด embedding cost ได้มาก
```

### ⚠️ Timeout สำหรับไฟล์ใหญ่

```python
# ไฟล์ใหญ่ (เช่น PDF 500 หน้า) อาจใช้เวลานาน
status = await client.documents.ingest(
    "large_document.pdf",
    poll_interval=5.0,    # เช็คทุก 5 วินาที
    timeout=1800.0,       # รอสูงสุด 30 นาที
)
```

### ⚠️ ลำดับการทำงานสำคัญ

```
✓ CORRECT:
1. ลบไฟล์เก่าก่อน (ถ้าเปลี่ยนชื่อ)
2. ingest ไฟล์ใหม่

✗ WRONG:
1. ingest ไฟล์ใหม่ก่อน
2. ลบไฟล์เก่า
→ ช่วงระหว่างกัน: ข้อมูลทั้ง 2 เวอร์ชันอยู่ใน index พร้อมกัน
```

---

## สรุปสั้น

| คำถาม | คำตอบ |
|-------|-------|
| ต้องเคลียของเก่ามั้ย? | **ไม่ต้อง** — ระบบลบเองอัตโนมัติเมื่ออัปโหลดชื่อเดิม |
| มันจะ update ยังไง? | **Delete All → Re-index All** ไม่มี partial update |
| ต้อง embed ใหม่ทุกอันมั้ย? | **เฉพาะไฟล์ที่เนื้อหาเปลี่ยน** ไฟล์เดิมจะ skip อัตโนมัติ |
| data เก่ากับใหม่จะปนกันมั้ย? | **ไม่ปน** ถ้าใช้ชื่อไฟล์เดิม |
| การทำ index เป็นยังไง? | ลบ chunks เก่าด้วย `delete_by_query` แล้ว index ใหม่ทีละ chunk |
| วิธีที่ดีที่สุด? | **ใช้ชื่อไฟล์เดิมเสมอ** + เรียก `ingest()` ปกติ |

---

*Generated: 2026-03-17 | OpenRAG Data Update Guide*
