---
title: "OpenRAG — Data Ingestion Channels"
type: source
source_file: raw/notes/openrag/docs-lean/openrag-data-ingestion-guide.md
published: 2026-03-17
tags: [openrag, ingestion, google-drive, s3, onedrive, sharepoint, webhook, api]
related: [wiki/concepts/openrag-platform.md, wiki/concepts/docling-document-parser.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/frameworks/openrag/docs-lean/openrag-data-ingestion-guide.md|Original file]]

## สรุป

8 ช่องทางนำข้อมูลเข้า OpenRAG — ตั้งแต่ UI Upload ธรรมดาไปจนถึง Cloud Connectors + Webhook auto-sync

## ประเด็นสำคัญ

### 8 Ingestion Channels

| ช่องทาง | API Endpoint | เหมาะกับ | Real-time |
|---------|-------------|---------|-----------|
| UI Upload (drag & drop) | `POST /api/router/upload_ingest` | ข้อมูลน้อย-กลาง | ❌ |
| Server Path (folder) | `POST /api/upload/path` | มี NAS/network drive | ❌ |
| S3 Bucket (AWS) | `POST /api/upload/bucket` | Archive ใน AWS | ❌ |
| Google Drive | `POST /api/connectors/google_drive/sync` | Google Workspace | ✅ Webhook |
| OneDrive | `POST /api/connectors/onedrive/sync` | Microsoft 365 Personal | ✅ Webhook |
| SharePoint | `POST /api/connectors/sharepoint/sync` | Microsoft 365 Business | ✅ Webhook |
| URL / Web (MCP) | Flow ID: `72c3d17c-...` | Public docs, websites | ❌ |
| REST API (API Key) | `POST /api/router/upload_ingest` | ETL, automation | ❌ |

ทุกช่องทาง → Docling (parse) → Embed → OpenSearch index

### รูปแบบไฟล์ที่รองรับ

PDF, DOCX/DOC, PPTX/PPT, XLSX/XLS, MD, HTML, TXT, CSV, PNG/JPG/JPEG (OCR), XML/JSON

### Webhook Auto-Sync (Cloud Connectors)

```
[User แก้ไขไฟล์ใน Google Drive]
       ↓ Google ส่ง notification
POST /api/connectors/google_drive/webhook
       ↓
OpenRAG re-index เฉพาะไฟล์ที่เปลี่ยน อัตโนมัติ
```

- ต้องเป็น HTTPS และเข้าถึงจาก internet ได้
- URL: `https://your-domain.com/api/connectors/{connector_type}/webhook`

### Google Drive Setup

1. Google Cloud Console → OAuth 2.0 Client
2. Redirect URI: `https://your-domain.com/api/connectors/google_drive/callback`
3. Env: `GOOGLE_DRIVE_CLIENT_ID`, `GOOGLE_DRIVE_CLIENT_SECRET`
4. Connect ผ่าน Settings → Connectors → Google Drive → Login Google

Google Docs/Sheets/Slides/Forms ถูก export เป็น DOCX/XLSX/PPTX/PDF อัตโนมัติก่อน ingest

### S3 Setup

```bash
POST /api/upload/bucket
{ "s3_url": "s3://my-bucket/documents/" }
# ต้องมี AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_DEFAULT_REGION
```

List objects ทั้งหมดใน bucket/prefix ผ่าน paginator (ไม่จำกัดจำนวนไฟล์)

### REST API Batch Upload (Python)

```python
import asyncio, httpx
from pathlib import Path

async def batch_upload(api_key, base_url, docs_dir, max_concurrent=5):
    semaphore = asyncio.Semaphore(max_concurrent)
    async with httpx.AsyncClient() as client:
        tasks = [upload_file(client, api_key, base_url, f, semaphore)
                 for f in Path(docs_dir).rglob("*") if f.is_file()]
        await asyncio.gather(*tasks, return_exceptions=True)
```

### Performance Guidelines

| ตัวแปร | คำแนะนำ |
|--------|---------|
| ขนาดไฟล์ใหญ่ | แตก PDF ใหญ่เป็นหลาย chapter |
| OCR | เปิดเฉพาะถ้าจำเป็น (ช้าลง 3-5x) |
| Concurrent tasks | จำกัดที่ 5-10 concurrent upload |
| OpenSearch > 1M docs | เพิ่ม replica / shard |

### Phase ทำงานสำหรับ Initial Bulk Load

```
A: มี Google Drive/SharePoint → Connect → File Picker → Sync (~100 ไฟล์/ชั่วโมง)
B: มีใน S3 → call /api/upload/bucket (เร็วกว่า A)
C: มี data บน server → Mount volume → call /api/upload/path (เร็วที่สุด)
```

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/openrag-platform|OpenRAG Platform]] — platform overview + ingestion flow
- [[wiki/concepts/docling-document-parser|Docling]] — document parser ที่ทุก channel ผ่าน
