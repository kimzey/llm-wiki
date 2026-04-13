# OpenRAG — วิธีนำ Data เข้าระบบ (ฉบับองค์กร)

> ระบบรองรับหลายช่องทางในการนำข้อมูลเข้า ตั้งแต่ drag & drop ไปจนถึง
> auto-sync กับ Cloud Storage และ S3 bucket สำหรับข้อมูลจำนวนมหาศาล

---

## สารบัญ

- [ภาพรวมทุกช่องทาง](#ภาพรวมทุกช่องทาง)
- [ช่องทาง 1: Upload ไฟล์ผ่าน UI](#ช่องทาง-1-upload-ไฟล์ผ่าน-ui)
- [ช่องทาง 2: Upload ทั้ง Folder (Server Path)](#ช่องทาง-2-upload-ทั้ง-folder-server-path)
- [ช่องทาง 3: S3 Bucket (AWS)](#ช่องทาง-3-s3-bucket-aws)
- [ช่องทาง 4: Google Drive Connector](#ช่องทาง-4-google-drive-connector)
- [ช่องทาง 5: OneDrive Connector](#ช่องทาง-5-onedrive-connector)
- [ช่องทาง 6: SharePoint Connector](#ช่องทาง-6-sharepoint-connector)
- [ช่องทาง 7: URL / Web Page (MCP Flow)](#ช่องทาง-7-url--web-page-mcp-flow)
- [ช่องทาง 8: REST API โดยตรง](#ช่องทาง-8-rest-api-โดยตรง)
- [ไฟล์รูปแบบใดบ้างที่รองรับ](#ไฟล์รูปแบบใดบ้างที่รองรับ)
- [Webhook Auto-Sync คืออะไร](#webhook-auto-sync-คืออะไร)
- [เปรียบเทียบช่องทางสำหรับองค์กร](#เปรียบเทียบช่องทางสำหรับองค์กร)
- [แผน Rollout สำหรับ Data จำนวนมาก](#แผน-rollout-สำหรับ-data-จำนวนมาก)

---

## ภาพรวมทุกช่องทาง

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATA INGESTION PATHS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [1] UI Upload          → drag & drop, หลายไฟล์พร้อมกัน        │
│  [2] Server Path        → folder บน server ที่ mount ไว้       │
│  [3] S3 Bucket          → AWS S3 ทั้ง bucket ได้เลย            │
│  [4] Google Drive       → auto-sync + webhook real-time        │
│  [5] OneDrive           → auto-sync + webhook real-time        │
│  [6] SharePoint         → auto-sync + webhook real-time        │
│  [7] URL/Web (MCP)      → ดึงจาก URL หรือ web page            │
│  [8] REST API           → เรียก API โดยตรง จาก script/ETL      │
│                                                                 │
│  ทุกช่องทาง → Docling (parse) → Embed → OpenSearch index       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## ช่องทาง 1: Upload ไฟล์ผ่าน UI

**เหมาะกับ:** ข้อมูลจำนวนน้อย–ปานกลาง, ทีมงานอัพโหลดด้วยตัวเอง

**API Endpoint:** `POST /api/router/upload_ingest`
**Content-Type:** `multipart/form-data`

```
Form fields:
  file[]             ← ไฟล์ 1 ไฟล์หรือหลายไฟล์พร้อมกัน
  session_id         ← (optional) conversation session
  delete_after_ingest = "true"   ← ลบ temp file หลัง index
  replace_duplicates = "true"    ← แทนที่ถ้าชื่อซ้ำ
  create_filter      = "false"   ← สร้าง knowledge filter ด้วยไหม
```

**กระบวนการ:**
1. รับไฟล์ → บันทึก temp file ใน `/tmp/`
2. สร้าง async task (non-blocking)
3. ส่ง task_id กลับทันที (status 202)
4. Backend ประมวลผล: Langflow → Docling → Embed → OpenSearch
5. ลบ temp file

**ตัวอย่าง response:**
```json
{
  "task_id": "abc123-...",
  "message": "Langflow upload task created for 3 file(s)",
  "file_count": 3
}
```

**ติดตามสถานะ:** `GET /api/tasks/{task_id}`

---

## ช่องทาง 2: Upload ทั้ง Folder (Server Path)

**เหมาะกับ:** มี data ที่ mount volume ไว้บน server อยู่แล้ว (เช่น NAS, network drive)

**API Endpoint:** `POST /api/upload/path`

```json
{
  "path": "/app/openrag-documents/hr-policies"
}
```

**หลักการทำงาน:**
- Walk ทุก subdirectory อัตโนมัติ (`os.walk`)
- สร้าง task เดียวสำหรับทุกไฟล์ใน folder
- Process parallel ตาม task system

**ตัวอย่างใช้งาน:**
```bash
# mount volume แล้ว copy ไฟล์ก่อน
cp -r /nas/company-docs /app/openrag-documents/

# จากนั้น call API
curl -X POST http://localhost:8000/api/upload/path \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"path": "/app/openrag-documents"}'
```

> **หมายเหตุ:** Path ต้องเป็น path บน container (ดู volume mount ใน `docker-compose.yml`)

---

## ช่องทาง 3: S3 Bucket (AWS)

**เหมาะกับ:** Data archive ใน AWS S3, documents จากหลาย department ที่เก็บ S3

**API Endpoint:** `POST /api/upload/bucket`

```json
{
  "s3_url": "s3://my-company-bucket/documents/hr/"
}
```

**Prerequisites:**
```yaml
# docker-compose.yml — ต้องใส่ env vars เหล่านี้
environment:
  AWS_ACCESS_KEY_ID: "AKIAIOSFODNN7EXAMPLE"
  AWS_SECRET_ACCESS_KEY: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
  AWS_DEFAULT_REGION: "ap-southeast-1"
```

**หลักการทำงาน:**
- List object ทั้งหมดใน bucket/prefix ผ่าน paginator (ไม่จำกัดจำนวนไฟล์)
- Download แต่ละไฟล์ผ่าน `S3FileProcessor`
- Process async ทุกไฟล์

**ตัวอย่าง:**
```bash
curl -X POST http://localhost:8000/api/upload/bucket \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"s3_url": "s3://acme-knowledge-base/2024/"}'
```

---

## ช่องทาง 4: Google Drive Connector

**เหมาะกับ:** องค์กรที่ใช้ Google Workspace — sync เอกสารจาก Drive อัตโนมัติ

### Setup ครั้งแรก

**1. ตั้งค่า Google OAuth App:**
- Google Cloud Console → Create OAuth 2.0 Client
- Authorized redirect URIs: `https://your-domain.com/api/connectors/google_drive/callback`

**2. ใส่ env vars:**
```yaml
environment:
  GOOGLE_DRIVE_CLIENT_ID: "xxx.apps.googleusercontent.com"
  GOOGLE_DRIVE_CLIENT_SECRET: "GOCSPX-..."
```

**3. Connect ผ่าน UI:**
- ไปที่ Settings → Connectors → Google Drive → Connect
- Login ด้วย Google account
- Grant permissions: Drive read access

### วิธี Sync

**Sync ไฟล์ที่เลือก (File Picker):**
```bash
POST /api/connectors/google_drive/sync
{
  "selected_files": [
    {"id": "1BxiMVs0XRA...", "name": "Q3-Report.pdf", "mimeType": "application/pdf"},
    {"id": "folder_id_xyz", "name": "HR Policies", "mimeType": "application/vnd.google-apps.folder"}
  ]
}
```
> โฟลเดอร์จะถูก expand เป็นไฟล์ทุกไฟล์ภายในอัตโนมัติ

**Sync ทุกไฟล์ที่ sync ไว้แล้ว (re-sync):**
```bash
POST /api/connectors/google_drive/sync
{}
```

**ดูสถานะ:**
```bash
GET /api/connectors/google_drive/status
```

### Webhook Real-time Sync

เมื่อไฟล์ใน Google Drive เปลี่ยนแปลง → Google ส่ง webhook มาที่:
```
POST /api/connectors/google_drive/webhook
```
→ ระบบ re-index เฉพาะไฟล์ที่เปลี่ยน อัตโนมัติ ไม่ต้อง sync manual

**Google Drive file types ที่รองรับ:**
| Google Format | แปลงเป็น |
|---------------|---------|
| Google Docs | .docx |
| Google Sheets | .xlsx |
| Google Slides | .pptx |
| Google Forms | .pdf |
| ไฟล์ปกติ (PDF, DOCX, etc.) | ตามเดิม |

---

## ช่องทาง 5: OneDrive Connector

**เหมาะกับ:** องค์กรที่ใช้ Microsoft 365 Personal/Business

### Setup

**1. Azure App Registration:**
- Azure Portal → App Registration → New registration
- API Permissions: `Files.Read.All`, `User.Read`
- Redirect URI: `https://your-domain.com/api/connectors/onedrive/callback`

**2. Env vars:**
```yaml
environment:
  ONEDRIVE_CLIENT_ID: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  ONEDRIVE_CLIENT_SECRET: "xxx~xxx..."
```

**3. Connect และ sync เหมือน Google Drive**

```bash
POST /api/connectors/onedrive/sync
{
  "selected_files": [{"id": "01BYE5RZ...", "name": "Annual Report.pdf"}]
}
```

---

## ช่องทาง 6: SharePoint Connector

**เหมาะกับ:** องค์กรที่ใช้ SharePoint Online สำหรับ document management

### Setup

**1. Azure App Registration (คล้าย OneDrive):**
- API Permissions: `Sites.Read.All`, `Files.Read.All`
- Redirect URI: `https://your-domain.com/api/connectors/sharepoint/callback`

**2. Env vars:**
```yaml
environment:
  SHAREPOINT_CLIENT_ID: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  SHAREPOINT_CLIENT_SECRET: "xxx~xxx..."
```

**3. Sync พร้อมระบุ SharePoint URL:**
```bash
POST /api/connectors/sharepoint/sync
{
  "selected_files": [
    {
      "id": "item_id_here",
      "name": "Company Policy.pdf",
      "downloadUrl": "https://tenant.sharepoint.com/_layouts/..."
    }
  ]
}
```

---

## ช่องทาง 7: URL / Web Page (MCP Flow)

**เหมาะกับ:** นำเข้าเนื้อหาจาก website, documentation, knowledge base ออนไลน์

**Flow ID:** `72c3d17c-2dac-4a73-b48a-6518473d7830` (openrag_url_mcp)

```bash
POST /api/v1/run/72c3d17c-2dac-4a73-b48a-6518473d7830
Authorization: Bearer $JWT

{
  "input_value": "https://docs.company.com/api-guide",
  "output_type": "chat",
  "input_type": "chat"
}
```

**ข้อจำกัด:**
- ต้องเป็น public URL หรือ URL ที่ server เข้าถึงได้
- ไม่รองรับ login-required pages
- เหมาะกับ: official docs, public knowledge base, company website

---

## ช่องทาง 8: REST API โดยตรง

**เหมาะกับ:** Script automation, ETL pipeline, CI/CD integration

### ด้วย API Key (ไม่ต้อง login)

```bash
# สร้าง API Key ก่อน (จาก UI → Settings → API Keys)
API_KEY="orag_xxxxxxxxxxxxxxxx"

# Upload ไฟล์เดียว
curl -X POST http://localhost:8000/api/router/upload_ingest \
  -H "X-API-Key: $API_KEY" \
  -F "file=@/path/to/document.pdf" \
  -F "delete_after_ingest=true" \
  -F "replace_duplicates=true"
```

### Batch Upload Script (หลาย 100 ไฟล์)

```python
import httpx
import asyncio
from pathlib import Path

API_KEY = "orag_xxxxxx"
BASE_URL = "http://localhost:8000"
DOCS_DIR = Path("/local/company-documents")

async def upload_file(client, filepath):
    with open(filepath, "rb") as f:
        response = await client.post(
            f"{BASE_URL}/api/router/upload_ingest",
            headers={"X-API-Key": API_KEY},
            files={"file": (filepath.name, f, "application/octet-stream")},
            data={
                "delete_after_ingest": "true",
                "replace_duplicates": "true",
            },
            timeout=60.0,
        )
    return response.json()

async def batch_upload(max_concurrent=5):
    files = list(DOCS_DIR.rglob("*"))
    files = [f for f in files if f.is_file()]

    print(f"Found {len(files)} files to upload")

    semaphore = asyncio.Semaphore(max_concurrent)

    async def limited_upload(client, filepath):
        async with semaphore:
            result = await upload_file(client, filepath)
            print(f"✓ {filepath.name}: task_id={result.get('task_id')}")
            return result

    async with httpx.AsyncClient() as client:
        tasks = [limited_upload(client, f) for f in files]
        results = await asyncio.gather(*tasks, return_exceptions=True)

    print(f"Done: {len([r for r in results if not isinstance(r, Exception)])} success")

asyncio.run(batch_upload())
```

### ตรวจสอบสถานะ Task

```python
async def wait_for_task(client, task_id, timeout=300):
    import time
    start = time.time()
    while time.time() - start < timeout:
        resp = await client.get(
            f"{BASE_URL}/api/tasks/{task_id}",
            headers={"X-API-Key": API_KEY}
        )
        data = resp.json()
        status = data.get("status")
        if status == "completed":
            return data
        elif status == "failed":
            raise Exception(f"Task failed: {data.get('error')}")
        await asyncio.sleep(2)
    raise TimeoutError(f"Task {task_id} timed out")
```

---

## ไฟล์รูปแบบใดบ้างที่รองรับ

Docling (parser) รองรับ:

| ประเภท | นามสกุล | หมายเหตุ |
|--------|---------|----------|
| PDF | .pdf | รวม PDF scanned (ต้องใช้ OCR) |
| Word | .docx, .doc | |
| PowerPoint | .pptx, .ppt | |
| Excel | .xlsx, .xls | |
| Markdown | .md | |
| HTML | .html, .htm | |
| Plain Text | .txt | |
| CSV | .csv | |
| Images | .png, .jpg, .jpeg | ต้องเปิด OCR |
| XML/JSON | .xml, .json | |

**OCR สำหรับ scanned PDF:**
```yaml
# docker-compose.yml
environment:
  DOCLING_OCR_ENABLED: "true"   # เปิด OCR (ช้าลง แต่ครอบคลุมกว่า)
```

---

## Webhook Auto-Sync คืออะไร

Cloud Connectors (Google Drive, OneDrive, SharePoint) รองรับ **real-time sync** ผ่าน Webhook:

```
[ผู้ใช้แก้ไขไฟล์ใน Google Drive]
          │
          ▼
[Google ส่ง notification มาที่ OpenRAG]
  POST /api/connectors/google_drive/webhook

          │
          ▼
[OpenRAG ระบุว่าไฟล์ไหนเปลี่ยน]
          │
          ▼
[สร้าง sync task อัตโนมัติ]
  → re-index เฉพาะไฟล์นั้น

          │
          ▼
[ผู้ใช้ query → ได้ข้อมูลล่าสุดทันที]
```

**ประโยชน์:**
- ไม่ต้อง manual sync
- ข้อมูลใน RAG ทันสมัยเสมอ
- ลด overhead (sync เฉพาะไฟล์ที่เปลี่ยน)

**Setup Webhook URL:**
```
https://your-openrag-domain.com/api/connectors/{connector_type}/webhook
```
ต้องเป็น HTTPS และ accessible จาก internet สำหรับ Google/Microsoft ส่ง notification มาได้

---

## เปรียบเทียบช่องทางสำหรับองค์กร

| ช่องทาง | ขนาดข้อมูล | Automation | Real-time | ใช้งานง่าย | เหมาะกับ |
|---------|-----------|------------|-----------|------------|----------|
| UI Upload | น้อย–กลาง | ❌ Manual | ❌ | ⭐⭐⭐ | ทีมงานอัพโหลดเอง |
| Server Path | มาก | ✅ Script | ❌ | ⭐⭐ | มี NAS/network drive |
| S3 Bucket | มหาศาล | ✅ Script | ❌ | ⭐⭐ | Archive ใน AWS |
| Google Drive | กลาง–มาก | ✅ Auto | ✅ Webhook | ⭐⭐⭐ | Google Workspace org |
| OneDrive | กลาง–มาก | ✅ Auto | ✅ Webhook | ⭐⭐⭐ | Microsoft 365 Personal |
| SharePoint | มาก | ✅ Auto | ✅ Webhook | ⭐⭐ | Microsoft 365 Business |
| URL/Web | น้อย | ❌ Manual | ❌ | ⭐⭐⭐ | Public docs, websites |
| REST API | ไม่จำกัด | ✅ Full | ❌ | ⭐ | ETL pipeline, DevOps |

---

## แผน Rollout สำหรับ Data จำนวนมาก

### Phase 1: Initial Bulk Load

**สำหรับ data สะสมมาหลายปี (10,000+ ไฟล์):**

```
Option A — มี Google Drive / SharePoint:
  1. Connect Connector (ทำครั้งเดียว)
  2. เลือก folder ทั้งหมดผ่าน File Picker
  3. Sync → ระบบทำให้เอง (background tasks)
  ⏱ ~100 ไฟล์/ชั่วโมง (ขึ้นกับขนาดไฟล์และ Docling)

Option B — มี data ใน S3:
  1. ตั้งค่า AWS credentials
  2. Call POST /api/upload/bucket หนึ่งครั้ง
  3. Monitor ผ่าน task system
  ⏱ เร็วกว่า Option A (ไม่ผ่าน OAuth)

Option C — มี data บน server:
  1. Mount volume ใน docker-compose.yml
  2. Copy files เข้า /app/openrag-documents/
  3. Call POST /api/upload/path
  ⏱ เร็วที่สุด (local disk I/O)
```

### Phase 2: Ongoing Maintenance

```
Cloud Storage (Drive/SharePoint/OneDrive):
  → Webhook auto-sync → ไม่ต้องทำอะไร

Local files ใหม่:
  → UI Upload หรือ Script

Data จาก ระบบอื่น (ERP, CRM, etc.):
  → Script กำหนดเวลา (cron) เรียก REST API
  → ตัวอย่าง: export รายงาน PDF ทุกวัน → upload อัตโนมัติ
```

### ตัวอย่าง Cron Script สำหรับ Daily Export

```bash
#!/bin/bash
# daily_ingest.sh — รันทุกวันตี 2

API_KEY="orag_xxxxxxxx"
EXPORT_DIR="/var/exports/$(date +%Y%m%d)"
BASE_URL="http://localhost:8000"

# Export รายงานจาก ERP (ตัวอย่าง)
python export_erp_reports.py --output "$EXPORT_DIR"

# Upload ทั้ง folder
curl -X POST "$BASE_URL/api/upload/path" \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"path\": \"$EXPORT_DIR\"}"

echo "Daily ingest completed: $(date)"
```

### ข้อควรระวัง: Performance

| ตัวแปร | ผลกระทบ | คำแนะนำ |
|--------|---------|----------|
| ขนาดไฟล์ | PDF 100 หน้า ช้ากว่า PDF 10 หน้า 10x | แตก PDF ใหญ่เป็นหลาย chapter |
| OCR | เปิด OCR ช้าลง 3–5x | เปิดเฉพาะถ้าจำเป็น |
| Concurrent tasks | มาก task พร้อมกัน → memory สูง | จำกัด concurrent upload ที่ 5–10 |
| OpenSearch | index ใหญ่ → query ช้า | เพิ่ม replica / shard ถ้า > 1M docs |
| Embedding model | model ใหญ่ = dimension มาก = index ใหญ่ | เลือก model ให้เหมาะกับ hardware |

---

## สรุปคำแนะนำตามสถานการณ์

```
องค์กรใช้ Google Workspace:
  → ✅ Google Drive Connector (best choice)
  → Initial: File Picker เลือก folder
  → Ongoing: Webhook auto-sync

องค์กรใช้ Microsoft 365:
  → ✅ SharePoint Connector
  → Initial + Ongoing เหมือน Google Drive

มี Legacy documents บน server/NAS:
  → ✅ Volume mount + POST /api/upload/path
  → ทำครั้งเดียว

มี Archive ใน AWS S3:
  → ✅ POST /api/upload/bucket
  → ทำครั้งเดียว + script สำหรับไฟล์ใหม่

ต้องการ Custom ETL / Integration กับระบบอื่น:
  → ✅ REST API + API Key
  → เขียน Python/Node.js script เรียก API

ต้องการ RAG จาก Website / Documentation:
  → ✅ URL Flow (MCP)
  → Manual หรือ script วนลูป URL
```
