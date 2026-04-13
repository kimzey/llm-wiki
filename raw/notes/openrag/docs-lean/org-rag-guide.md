# คู่มือทำ RAG สำหรับองค์กร — ตั้งแต่ศูนย์จนใช้งานได้จริง

---

## สารบัญ

1. [ภาพรวมสิ่งที่ต้องทำ](#1-ภาพรวมสิ่งที่ต้องทำ)
2. [ขั้นตอน Setup ระบบ](#2-ขั้นตอน-setup-ระบบ)
3. [วิธีนำ DATA เข้าระบบ](#3-วิธีนำ-data-เข้าระบบ)
4. [วิธีใช้ RAG](#4-วิธีใช้-rag)
5. [API สำหรับ External Integration](#5-api-สำหรับ-external-integration)
6. [Data-Level Access Control](#6-data-level-access-control)
7. [สิ่งที่ต้องรู้เพิ่มเติม](#7-สิ่งที่ต้องรู้เพิ่มเติม)

---

## 1. ภาพรวมสิ่งที่ต้องทำ

```
องค์กรต้องการ RAG — Checklist:

[ ] 1. ติดตั้ง OpenRAG (docker compose up)
[ ] 2. ตั้งค่า .env (Password, API Keys)
[ ] 3. รัน Docling แยก (document parser)
[ ] 4. เลือก LLM Provider (OpenAI / Claude / Ollama)
[ ] 5. นำเอกสารเข้าระบบ (Ingest)
[ ] 6. ตั้งค่า Access Control ว่าใครเห็นอะไร
[ ] 7. สร้าง API Key สำหรับ app ภายนอก (ถ้าต้องการ)
[ ] 8. ทดสอบถามคำถาม
```

---

## 2. ขั้นตอน Setup ระบบ

### 2.1 Prerequisites

```bash
# ต้องมีในเครื่อง:
- Docker + Docker Compose
- Git
- RAM อย่างน้อย 8GB (แนะนำ 16GB+)
- Disk อย่างน้อย 20GB
```

### 2.2 Clone และ Setup

```bash
# 1. Clone repo
git clone https://github.com/IBM/openrag.git
cd openrag

# 2. Copy env template
cp .env.example .env
```

### 2.3 ตั้งค่า .env ที่จำเป็น

เปิดไฟล์ `.env` แล้วแก้ไขค่าเหล่านี้:

```bash
# ===== REQUIRED — ต้องตั้งทุกอัน =====

# OpenSearch password (ต้องซับซ้อน: ตัวใหญ่ + ตัวเล็ก + ตัวเลข + อักขระพิเศษ)
OPENSEARCH_PASSWORD=MyOrg@Secure2024!
OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyOrg@Secure2024!

# Secret key สำหรับ JWT (สุ่มได้)
SECRET_KEY=my-organization-super-secret-key-min-32-chars

# LLM Provider — เลือกอย่างน้อย 1 อัน
OPENAI_API_KEY=sk-proj-...          # ใช้ OpenAI (แนะนำสำหรับเริ่มต้น)
# ANTHROPIC_API_KEY=sk-ant-...      # หรือ Claude
# OLLAMA_ENDPOINT=http://...        # หรือ Local Model (ฟรี)

# Embedding model (ต้องสอดคล้องกับ LLM provider)
SELECTED_EMBEDDING_MODEL=text-embedding-3-small
```

### 2.4 รัน Services

```bash
# รัน OpenRAG ทั้งหมด
docker compose up -d

# ดู logs
docker compose logs -f openrag-backend

# ตรวจสอบว่า services พร้อม
curl http://localhost:8080/health
# Expected: {"status": "ok"}
```

### 2.5 รัน Docling (แยก — ต้องรันก่อนใช้งาน)

```bash
# Option 1: Docker (ง่ายที่สุด)
docker run -d \
  --name docling \
  -p 5001:5001 \
  -e DOCLING_OCR_ENGINE=easyocr \
  quay.io/docling-project/docling-serve:latest

# Option 2: Docker + GPU (เร็วกว่า 5-10x)
docker run -d \
  --name docling \
  --gpus all \
  -p 5001:5001 \
  quay.io/docling-project/docling-serve:latest-gpu

# ตรวจสอบ
curl http://localhost:5001/health
# Expected: {"status": "ok"}
```

### 2.6 เข้าใช้งาน

```
Frontend UI:         http://localhost:3000
Langflow UI:         http://localhost:7860
OpenSearch Dashboards: http://localhost:5601
Backend API Docs:    http://localhost:8080/docs
```

---

## 3. วิธีนำ DATA เข้าระบบ

มี 5 วิธีในการนำข้อมูลเข้า ขึ้นอยู่กับ use case:

---

### วิธีที่ 1: Upload ผ่าน Web UI (ง่ายที่สุด)

**เหมาะสำหรับ:** ผู้ใช้ทั่วไป, ไฟล์ไม่มาก

```
1. เปิด http://localhost:3000
2. Login
3. คลิก "Upload Document"
4. เลือกไฟล์ (PDF, DOCX, PPTX, ฯลฯ)
5. ตั้งค่า permissions (ใครเห็นได้บ้าง)
6. คลิก Upload
7. รอ processing (progress bar)
8. เสร็จ → ไฟล์พร้อมถาม
```

**รองรับไฟล์:** PDF, DOCX, PPTX, XLSX, HTML, TXT, PNG, JPG

---

### วิธีที่ 2: Upload ผ่าน REST API (Programmatic)

**เหมาะสำหรับ:** การ automate, batch upload, integration

```bash
# Step 1: สร้าง API Key ก่อน (ผ่าน Dashboard)
# Settings → API Keys → Create

# Step 2: Upload + Ingest
curl -X POST "http://localhost:8080/v1/documents/ingest" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -F "file=@/path/to/document.pdf" \
  -F "allowed_groups=all-employees"

# Response:
{
  "task_id": "task-abc-123",
  "status": "pending",
  "filename": "document.pdf"
}

# Step 3: ตรวจสอบ status
curl "http://localhost:8080/v1/tasks/task-abc-123" \
  -H "Authorization: Bearer YOUR_API_KEY"

# Response เมื่อเสร็จ:
{
  "task_id": "task-abc-123",
  "status": "success",
  "result": {
    "chunks_indexed": 45,
    "processing_time": 38.2
  }
}
```

---

### วิธีที่ 3: Batch Upload หลายไฟล์พร้อมกัน

**เหมาะสำหรับ:** นำเอกสารทั้งหมดขององค์กรเข้าระบบครั้งแรก

```bash
#!/bin/bash
# batch_ingest.sh — script สำหรับ upload ทีละหลายไฟล์

API_URL="http://localhost:8080/v1/documents/ingest"
API_KEY="YOUR_API_KEY"
DOC_FOLDER="/path/to/documents"
ALLOWED_GROUP="all-employees"

for file in "$DOC_FOLDER"/*.pdf; do
  filename=$(basename "$file")
  echo "Uploading: $filename"

  response=$(curl -s -X POST "$API_URL" \
    -H "Authorization: Bearer $API_KEY" \
    -F "file=@$file" \
    -F "allowed_groups=$ALLOWED_GROUP")

  task_id=$(echo $response | python3 -c "import sys,json; print(json.load(sys.stdin)['task_id'])")
  echo "  Task ID: $task_id"

  # รอให้ task เสร็จ
  while true; do
    status=$(curl -s "http://localhost:8080/v1/tasks/$task_id" \
      -H "Authorization: Bearer $API_KEY" | \
      python3 -c "import sys,json; print(json.load(sys.stdin)['status'])")

    if [ "$status" = "success" ]; then
      echo "  ✅ Done: $filename"
      break
    elif [ "$status" = "failed" ]; then
      echo "  ❌ Failed: $filename"
      break
    fi
    sleep 5
  done
done

echo "Batch upload complete!"
```

```bash
chmod +x batch_ingest.sh
./batch_ingest.sh
```

---

### วิธีที่ 4: Sync จาก Cloud Storage

**เหมาะสำหรับ:** องค์กรที่เก็บเอกสารใน Google Drive / SharePoint / OneDrive

#### Google Drive:
```bash
# 1. ตั้งค่าใน .env
GOOGLE_OAUTH_CLIENT_ID=your-client-id
GOOGLE_OAUTH_CLIENT_SECRET=your-client-secret

# 2. Connect ผ่าน UI: Settings → Connectors → Google Drive
# หรือผ่าน API:
curl -X POST "http://localhost:8080/connectors/google_drive/sync" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"folder_id": "1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs"}'

# 3. ตรวจสอบ
curl "http://localhost:8080/connectors/google_drive/status" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

#### SharePoint / OneDrive:
```bash
# .env
MICROSOFT_GRAPH_OAUTH_CLIENT_ID=your-client-id
MICROSOFT_GRAPH_OAUTH_CLIENT_SECRET=your-client-secret

# Sync
curl -X POST "http://localhost:8080/connectors/sharepoint/sync" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

---

### วิธีที่ 5: Ingest จาก URL

**เหมาะสำหรับ:** เอกสารที่อยู่บน Web, internal wiki, documentation sites

```bash
# Ingest เนื้อหาจาก URL โดยตรง
curl -X POST "http://localhost:8080/langflow/ingest" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://internal.company.com/wiki/hr-policy",
    "allowed_groups": ["all-employees"]
  }'
```

---

### สรุปเปรียบเทียบวิธีนำ DATA เข้า

| วิธี | เหมาะกับ | ง่าย | Auto |
|------|---------|------|------|
| Web UI | ผู้ใช้ทั่วไป, ไฟล์น้อย | ✅ | ❌ |
| REST API | Developer, integration | ⚠️ | ✅ |
| Batch Script | เอกสารเยอะ, ทำครั้งเดียว | ⚠️ | ✅ |
| Cloud Sync | เก็บใน Drive/SharePoint | ⚠️ | ✅ |
| URL Ingest | Web content, Wiki | ✅ | ✅ |

---

## 4. วิธีใช้ RAG

### 4.1 ถามผ่าน Web UI

```
1. เปิด http://localhost:3000
2. Login
3. ไปที่ "Chat" หรือ "Ask"
4. พิมพ์คำถาม เช่น "นโยบายลาป่วยของบริษัทคือกี่วัน?"
5. ระบบจะ:
   - ค้นหาเอกสารที่เกี่ยวข้อง
   - ตอบคำถาม พร้อม citation (บอกว่ามาจากไฟล์ไหน หน้าไหน)
```

**Tips:**
- ถามเป็นภาษาเดียวกับเอกสาร = ผลดีที่สุด
- ถามเป็น natural language ได้เลย
- ระบบรองรับ follow-up questions (ถามต่อเนื่องได้)

---

### 4.2 ถามผ่าน API (สำหรับ App ที่ integrate)

```bash
# Basic Chat
curl -X POST "http://localhost:8080/v1/chat" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "นโยบายลาป่วยคือกี่วัน?",
    "session_id": "my-session-001"
  }'

# Response:
{
  "chat_id": "chat-xyz-789",
  "response": "ตามนโยบายบริษัท พนักงานมีสิทธิลาป่วยได้ 30 วันต่อปี โดยต้องมีใบรับรองแพทย์หากลาเกิน 3 วัน",
  "citations": [
    {
      "filename": "hr_policy_2024.pdf",
      "page": 3,
      "text": "Employees are entitled to 30 days of sick leave..."
    }
  ],
  "session_id": "my-session-001"
}
```

---

### 4.3 ถามแบบ Streaming (ตอบทีละคำ เหมือน ChatGPT)

```python
import httpx

async def stream_chat(message: str, session_id: str):
    async with httpx.AsyncClient() as client:
        async with client.stream(
            "POST",
            "http://localhost:8080/langflow",
            headers={"Authorization": "Bearer YOUR_API_KEY"},
            json={
                "message": message,
                "session_id": session_id
            },
            timeout=120
        ) as response:
            async for chunk in response.aiter_text():
                print(chunk, end="", flush=True)

# ใช้งาน
import asyncio
asyncio.run(stream_chat(
    "สรุปนโยบาย HR ของบริษัทให้หน่อย",
    "session-001"
))
```

---

### 4.4 ค้นหาเอกสาร (Search ล้วนๆ ไม่ generate)

```bash
# ค้นหา raw chunks ที่เกี่ยวข้อง
curl -X POST "http://localhost:8080/v1/search" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "sick leave policy",
    "k": 5
  }'

# Response:
{
  "results": [
    {
      "text": "Employees are entitled to 30 days of sick leave...",
      "filename": "hr_policy_2024.pdf",
      "page": 3,
      "score": 0.943
    },
    ...
  ]
}
```

---

### 4.5 ใช้ Knowledge Filter (จำกัดขอบเขตค้นหา)

```bash
# สร้าง filter เฉพาะ HR documents
curl -X POST "http://localhost:8080/knowledge-filter" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "HR Only",
    "description": "เฉพาะเอกสาร HR",
    "filter": {
      "terms": {
        "allowed_groups": ["hr-team", "all-employees"]
      }
    }
  }'
# Response: { "filter_id": "kf-hr-001" }

# ใช้ filter ใน chat
curl -X POST "http://localhost:8080/v1/chat" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "message": "ลาคลอดได้กี่วัน?",
    "session_id": "sess-001",
    "knowledge_filter_id": "kf-hr-001"
  }'
```

---

### 4.6 ดูประวัติการสนทนา

```bash
# ดู session ทั้งหมด
curl "http://localhost:8080/langflow/history" \
  -H "Authorization: Bearer YOUR_API_KEY"

# ดู chat เฉพาะ session
curl "http://localhost:8080/v1/chat/chat-xyz-789" \
  -H "Authorization: Bearer YOUR_API_KEY"

# ลบ session
curl -X DELETE "http://localhost:8080/sessions/my-session-001" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

---

## 5. API สำหรับ External Integration

### 5.1 Authentication — สร้าง API Key

```
วิธีสร้าง:
1. เข้า Frontend: http://localhost:3000
2. ไปที่ Settings → API Keys
3. คลิก "Generate New Key"
4. ตั้งชื่อ เช่น "HR Chatbot Integration"
5. Copy key → เก็บไว้ให้ดี (แสดงครั้งเดียว)

รูปแบบ: sk-openrag-xxxxxxxxxxxxxxxx
ใช้ใน Header: Authorization: Bearer sk-openrag-xxxxxxxx
```

### 5.2 Public API v1 — Endpoints ทั้งหมด

Base URL: `http://your-openrag-server:8080`

#### Chat API

```http
POST /v1/chat
Authorization: Bearer {api_key}
Content-Type: application/json

{
  "message": "คำถามของ user",
  "session_id": "unique-session-id",         // optional — สำหรับ conversation history
  "knowledge_filter_id": "kf-xxx",           // optional — จำกัด scope
  "model": "gpt-4o"                          // optional — เลือก LLM
}

Response 200:
{
  "chat_id": "chat-abc-123",
  "response": "คำตอบจาก AI...",
  "citations": [
    {
      "filename": "document.pdf",
      "page": 5,
      "text": "relevant excerpt..."
    }
  ],
  "session_id": "unique-session-id"
}
```

```http
GET /v1/chat/{chat_id}
Authorization: Bearer {api_key}

Response 200:
{
  "chat_id": "chat-abc-123",
  "message": "คำถาม",
  "response": "คำตอบ",
  "citations": [...],
  "created_at": "2024-11-15T10:30:00Z"
}
```

```http
DELETE /v1/chat/{chat_id}
Authorization: Bearer {api_key}

Response 200: { "deleted": true }
```

#### Search API

```http
POST /v1/search
Authorization: Bearer {api_key}
Content-Type: application/json

{
  "query": "search query",
  "k": 10,                                   // จำนวนผลลัพธ์ (default: 10)
  "knowledge_filter_id": "kf-xxx"            // optional
}

Response 200:
{
  "results": [
    {
      "text": "relevant text chunk...",
      "filename": "document.pdf",
      "page": 3,
      "score": 0.943,
      "document_id": "doc-abc-chunk-001"
    }
  ]
}
```

#### Document Ingestion API

```http
POST /v1/documents/ingest
Authorization: Bearer {api_key}
Content-Type: multipart/form-data

file: (binary file)
allowed_users: user1@company.com,user2@company.com    // optional
allowed_groups: hr-team,all-employees                 // optional

Response 202:
{
  "task_id": "task-abc-123",
  "status": "pending",
  "filename": "document.pdf"
}
```

```http
DELETE /v1/documents
Authorization: Bearer {api_key}
Content-Type: application/json

{
  "filename": "document.pdf"
}

Response 200: { "deleted": true, "chunks_removed": 45 }
```

#### Task Status API

```http
GET /v1/tasks/{task_id}
Authorization: Bearer {api_key}

Response 200:
{
  "task_id": "task-abc-123",
  "status": "success",           // pending | processing | success | failed | cancelled
  "result": {
    "filename": "document.pdf",
    "chunks_indexed": 45,
    "processing_time": 38.2
  },
  "error": null
}
```

---

### 5.3 ตัวอย่าง Integration กับ Application จริง

#### Python SDK-style wrapper:

```python
import httpx
from typing import Optional

class OpenRAGClient:
    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url.rstrip("/")
        self.headers = {
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json"
        }

    async def chat(
        self,
        message: str,
        session_id: Optional[str] = None,
        filter_id: Optional[str] = None
    ) -> dict:
        async with httpx.AsyncClient(timeout=120) as client:
            payload = {"message": message}
            if session_id:
                payload["session_id"] = session_id
            if filter_id:
                payload["knowledge_filter_id"] = filter_id

            response = await client.post(
                f"{self.base_url}/v1/chat",
                headers=self.headers,
                json=payload
            )
            response.raise_for_status()
            return response.json()

    async def search(self, query: str, k: int = 10) -> list:
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.base_url}/v1/search",
                headers=self.headers,
                json={"query": query, "k": k}
            )
            return response.json()["results"]

    async def ingest_file(
        self,
        file_path: str,
        allowed_groups: list[str] = None
    ) -> str:
        """Returns task_id"""
        async with httpx.AsyncClient(timeout=300) as client:
            with open(file_path, "rb") as f:
                files = {"file": f}
                data = {}
                if allowed_groups:
                    data["allowed_groups"] = ",".join(allowed_groups)

                response = await client.post(
                    f"{self.base_url}/v1/documents/ingest",
                    headers={"Authorization": self.headers["Authorization"]},
                    files=files,
                    data=data
                )
            return response.json()["task_id"]

    async def wait_for_task(self, task_id: str) -> dict:
        import asyncio
        async with httpx.AsyncClient() as client:
            while True:
                response = await client.get(
                    f"{self.base_url}/v1/tasks/{task_id}",
                    headers=self.headers
                )
                result = response.json()
                if result["status"] in ["success", "failed"]:
                    return result
                await asyncio.sleep(5)

# ใช้งาน:
import asyncio

async def main():
    client = OpenRAGClient(
        base_url="http://localhost:8080",
        api_key="sk-openrag-xxxxxxxx"
    )

    # Ingest เอกสาร
    task_id = await client.ingest_file(
        "hr_policy.pdf",
        allowed_groups=["all-employees"]
    )
    result = await client.wait_for_task(task_id)
    print(f"Ingested: {result['result']['chunks_indexed']} chunks")

    # ถามคำถาม
    answer = await client.chat(
        message="ลาป่วยได้กี่วัน?",
        session_id="hr-bot-session-001"
    )
    print(f"Answer: {answer['response']}")
    print(f"Source: {answer['citations'][0]['filename']}")

asyncio.run(main())
```

#### Node.js / TypeScript:

```typescript
class OpenRAGClient {
  constructor(
    private baseUrl: string,
    private apiKey: string
  ) {}

  private get headers() {
    return {
      Authorization: `Bearer ${this.apiKey}`,
      "Content-Type": "application/json",
    };
  }

  async chat(message: string, sessionId?: string) {
    const res = await fetch(`${this.baseUrl}/v1/chat`, {
      method: "POST",
      headers: this.headers,
      body: JSON.stringify({ message, session_id: sessionId }),
    });
    return res.json();
  }

  async search(query: string, k = 10) {
    const res = await fetch(`${this.baseUrl}/v1/search`, {
      method: "POST",
      headers: this.headers,
      body: JSON.stringify({ query, k }),
    });
    const data = await res.json();
    return data.results;
  }
}

// ใช้งาน
const rag = new OpenRAGClient("http://localhost:8080", "sk-openrag-xxx");

const answer = await rag.chat("นโยบาย HR คืออะไร?", "session-001");
console.log(answer.response);
console.log(answer.citations);
```

---

### 5.4 Webhook / Event Integration

ระบบ Task ทำงาน async — ต้อง poll status หรือ implement polling loop:

```python
# Polling loop สำหรับ production
async def ingest_with_callback(file_path: str, on_complete):
    task_id = await client.ingest_file(file_path)

    while True:
        task = await client.get_task(task_id)

        if task["status"] == "success":
            await on_complete(task["result"])
            break
        elif task["status"] == "failed":
            raise Exception(f"Ingestion failed: {task.get('error')}")

        await asyncio.sleep(10)
```

---

## 6. Data-Level Access Control

### 6.1 ระบบ Permissions มี 3 ระดับ

```
ทุก Document ที่ Index จะมี:

owner:           "hr.manager@company.com"    ← เจ้าของ (เห็นได้เสมอ)
allowed_users:   ["alice@co.com", "bob@co.com"]  ← รายชื่อ user ที่เห็นได้
allowed_groups:  ["hr-team", "finance"]          ← กลุ่มที่เห็นได้

Logic: ถ้าเป็น owner OR อยู่ใน allowed_users OR อยู่ใน allowed_groups
       → เห็นเอกสาร
```

### 6.2 การตั้งค่า Permissions ตอน Ingest

#### ผ่าน Web UI:
```
Upload Document → ตั้งค่า Access:
  ☐ Public (ทุกคนในระบบเห็นได้)
  ☐ Private (เฉพาะตัวเอง)
  ☐ Specific Users: user1@co.com, user2@co.com
  ☐ Groups: hr-team, management
```

#### ผ่าน API:
```bash
# ตัวอย่าง 1: เอกสารสำหรับทุกคน
curl -X POST "http://localhost:8080/v1/documents/ingest" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -F "file=@company_handbook.pdf" \
  -F "allowed_groups=all-employees"

# ตัวอย่าง 2: เอกสาร HR เฉพาะทีม HR
curl -X POST "http://localhost:8080/v1/documents/ingest" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -F "file=@salary_structure.pdf" \
  -F "allowed_groups=hr-team,management"

# ตัวอย่าง 3: เอกสาร confidential เฉพาะบุคคล
curl -X POST "http://localhost:8080/v1/documents/ingest" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -F "file=@performance_review_alice.pdf" \
  -F "allowed_users=alice@company.com,hr.manager@company.com"

# ตัวอย่าง 4: เฉพาะ owner คนเดียว (private)
curl -X POST "http://localhost:8080/v1/documents/ingest" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -F "file=@my_personal_notes.pdf"
  # ไม่ตั้ง allowed_users/groups = เฉพาะตัวเองเห็น
```

---

### 6.3 ตัวอย่าง Scenario จริง — การแบ่ง Permissions

```
บริษัท ABC มีแผนก: HR, Finance, Engineering, Management

เอกสาร "Company Handbook"
  → allowed_groups: ["all-employees"]
  → ทุกคนในบริษัทเห็นได้

เอกสาร "Salary Structure 2024"
  → allowed_groups: ["hr-team", "management"]
  → เฉพาะ HR และ Management เห็น

เอกสาร "Q3 Financial Report"
  → allowed_groups: ["finance-team", "management"]
  → เฉพาะ Finance และ Management เห็น

เอกสาร "Engineering Architecture Doc"
  → allowed_groups: ["engineering-team"]
  → เฉพาะ Engineering เห็น

เอกสาร "Performance Review - John"
  → allowed_users: ["john@abc.com", "john.manager@abc.com", "hr@abc.com"]
  → เฉพาะ John, Manager ของ John, และ HR เห็น
```

---

### 6.4 Groups — การตั้งค่าและจัดการ

Groups ไม่ได้ถูก manage โดย OpenRAG โดยตรง — ระบบ match จาก JWT token ที่ส่งมา:

**Option A: ใช้กับ OAuth (Google/Microsoft)**
```
Groups มาจาก Google Workspace Groups หรือ Microsoft AD Groups
เมื่อ user login ด้วย Google/Microsoft
→ JWT token จะมี groups ที่ user อยู่
→ OpenSearch filter ใช้ groups เหล่านี้
```

**Option B: Custom groups ผ่าน API key**
```python
# เมื่อเรียก API ด้วย API key
# ต้องส่ง user context พร้อมกัน
curl -X POST "http://localhost:8080/v1/chat" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "X-User-Id: john@company.com" \         # ← user ที่กำลังถาม
  -H "X-User-Groups: hr-team,all-employees" \ # ← groups ของ user นี้
  -d '{"message": "ลาป่วยได้กี่วัน?"}'
```

---

### 6.5 ตรวจสอบ Access Control ใน OpenSearch

```bash
# ดูว่า user นี้เห็นเอกสารอะไรบ้าง (ผ่าน OpenSearch Dashboards)
# http://localhost:5601 → Dev Tools

POST /documents/_search
{
  "query": {
    "bool": {
      "should": [
        {"term": {"owner": "john@company.com"}},
        {"term": {"allowed_users": "john@company.com"}},
        {"terms": {"allowed_groups": ["engineering-team", "all-employees"]}}
      ]
    }
  },
  "_source": ["filename", "owner", "allowed_groups"],
  "size": 20
}
```

---

### 6.6 Row-Level Security — ทำงานยังไงเบื้องหลัง

```python
# /src/services/search_service.py
# เมื่อ user ค้นหา — backend เพิ่ม filter อัตโนมัติ:

def build_access_filter(user: str, groups: list[str]) -> dict:
    return {
        "bool": {
            "should": [
                {"term": {"owner": user}},
                {"term": {"allowed_users": user}},
                {"terms": {"allowed_groups": groups}}
            ],
            "minimum_should_match": 1
        }
    }

# Query ที่ส่งไป OpenSearch:
query = {
    "bool": {
        "must": {
            "knn": {
                "embeddings": {"vector": query_vector, "k": 10}
            }
        },
        "filter": build_access_filter(
            user="john@company.com",
            groups=["engineering-team", "all-employees"]
        )
    }
}
```

**ผล:** John ค้นหาอะไรก็ตาม จะเห็นเฉพาะเอกสารที่ตัวเองมีสิทธิ์เข้าถึง — **ไม่ต้องทำอะไรเพิ่ม** ระบบจัดการให้เองอัตโนมัติ

---

## 7. สิ่งที่ต้องรู้เพิ่มเติม

### 7.1 เลือก LLM Provider อย่างไร?

| Provider | ค่าใช้จ่าย | Privacy | คุณภาพ | Setup |
|---------|----------|---------|--------|-------|
| **OpenAI** (GPT-4o) | ต้องจ่าย | ข้อมูลออกนอก | ดีมาก | ง่าย |
| **Claude** (Anthropic) | ต้องจ่าย | ข้อมูลออกนอก | ดีมาก | ง่าย |
| **WatsonX** (IBM) | ต้องจ่าย | มี on-prem option | ดี | ปานกลาง |
| **Ollama** (Local) | ฟรี | ข้อมูลไม่ออก | ปานกลาง | ต้อง GPU |

**แนะนำสำหรับองค์กร:**
- ข้อมูลไม่ sensitive: OpenAI GPT-4o-mini (ถูก + เร็ว)
- ข้อมูล sensitive / ต้องการ privacy: Ollama + Llama 3.1 (local)
- Enterprise: IBM WatsonX (on-prem option)

---

### 7.2 เลือก Embedding Model อย่างไร?

```bash
# Default — ดีที่สุดสำหรับเริ่มต้น
SELECTED_EMBEDDING_MODEL=text-embedding-3-small
# Dimension: 1536, ราคา: ถูก, คุณภาพ: ดีมาก

# ถ้าต้องการ accuracy สูงกว่า (แต่แพงกว่า 2x)
SELECTED_EMBEDDING_MODEL=text-embedding-3-large
# Dimension: 3072

# ถ้าใช้ Ollama (ฟรี, local)
SELECTED_EMBEDDING_MODEL=nomic-embed-text
# Dimension: 768
```

**⚠️ สำคัญ:** เมื่อตั้งค่า embedding model แล้ว อย่าเปลี่ยน
ถ้าเปลี่ยน = ต้อง re-index เอกสารทั้งหมดใหม่

---

### 7.3 Chunk Size — ส่งผลต่ออะไร?

```
Chunk Size คือขนาดของ "ชิ้นส่วนข้อความ" ที่แบ่งจากเอกสาร

เล็กเกินไป (< 500 tokens):
  ✅ ค้นหาแม่นยำ
  ❌ ขาดบริบท ตอบไม่ครบ
  ❌ Index ใหญ่มาก

ใหญ่เกินไป (> 2000 tokens):
  ✅ บริบทครบ
  ❌ ค้นหาไม่แม่นยำ
  ❌ LLM อาจ miss ข้อมูลสำคัญ

แนะนำ: 800-1200 tokens (ค่า default ของ OpenRAG)

ปรับได้ใน Langflow:
  → เปิด Langflow UI → ingestion_flow
  → คลิก TextSplitter component
  → แก้ chunk_size
```

---

### 7.4 จัดการ Tasks และ Queue

```bash
# ดู tasks ทั้งหมด (ต้องการ admin หรือ internal API)
curl "http://localhost:8080/tasks" \
  -H "Authorization: Bearer YOUR_API_KEY"

# Response:
{
  "tasks": [
    {"task_id": "task-001", "status": "success", "filename": "a.pdf"},
    {"task_id": "task-002", "status": "processing", "filename": "b.pdf"},
    {"task_id": "task-003", "status": "pending", "filename": "c.pdf"}
  ]
}

# ยกเลิก task
curl -X POST "http://localhost:8080/tasks/task-003/cancel" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

---

### 7.5 ลบเอกสารออกจากระบบ

```bash
# ลบตาม filename
curl -X POST "http://localhost:8080/documents/delete-by-filename" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"filename": "old_policy_2023.pdf"}'

# ลบผ่าน public API
curl -X DELETE "http://localhost:8080/v1/documents" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"filename": "old_policy_2023.pdf"}'
```

---

### 7.6 Monitor ระบบ

```bash
# Health check ทุก service
curl http://localhost:8080/health
curl http://localhost:5001/health

# ดู available models
curl "http://localhost:8080/models" \
  -H "Authorization: Bearer YOUR_API_KEY"

# ดู current settings
curl "http://localhost:8080/settings" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

**OpenSearch Dashboards (http://localhost:5601):**
```
admin / {OPENSEARCH_PASSWORD}

ดูสิ่งที่สำคัญ:
- Index Management → ดูจำนวน documents ใน index
- Dev Tools → ทดสอบ query โดยตรง
- Cluster Health → ดูสถานะ cluster
```

---

### 7.7 Backup & Recovery

```bash
# Backup OpenSearch data
docker exec opensearch \
  curl -X PUT "https://localhost:9200/_snapshot/backup" \
  -u admin:YourPassword \
  --insecure \
  -H "Content-Type: application/json" \
  -d '{"type": "fs", "settings": {"location": "/backup"}}'

# Backup Langflow flows
POST http://localhost:8080/flows/backup
# บันทึก flows ปัจจุบัน

# Data volumes ที่ต้อง backup:
# - ./opensearch-data/    ← OpenSearch index data
# - ./keys/               ← Langflow API key
# - ./flows/              ← Flow definitions
# - ./data/               ← Application data
# - ./.env                ← Configuration
```

---

### 7.8 Security Checklist สำหรับ Production

```
[ ] เปลี่ยน OPENSEARCH_PASSWORD เป็น password แข็งแกร่ง
[ ] เปลี่ยน SECRET_KEY เป็น random string ยาว 64+ chars
[ ] ตั้ง LANGFLOW_AUTO_LOGIN=False
[ ] ใช้ HTTPS (reverse proxy เช่น nginx)
[ ] จำกัด port ที่เปิดออกสู่ internet (เฉพาะ 3000 และ 8080)
[ ] ไม่ expose OpenSearch (9200) และ Langflow (7860) สู่ internet
[ ] Rotate API keys เป็นประจำ
[ ] Enable audit logging
[ ] ตั้ง CORS ให้เฉพาะ domain ที่อนุญาต
```

---

### 7.9 Scaling สำหรับ Production

```yaml
# docker-compose.override.yml สำหรับ production
services:
  opensearch:
    environment:
      - OPENSEARCH_JAVA_OPTS=-Xms4g -Xmx4g  # เพิ่ม RAM

  openrag-backend:
    deploy:
      replicas: 2  # รัน backend 2 instances

  langflow:
    environment:
      - LANGFLOW_WORKERS=4  # เพิ่ม workers
```

---

### 7.10 ตัวอย่าง Use Cases ในองค์กร

```
1. HR Knowledge Base
   → เอกสาร: HR Policy, Employee Handbook, Benefits Guide
   → Users: ทุกพนักงาน
   → Bot: "HR Assistant" ใน LINE/Slack

2. Legal & Compliance
   → เอกสาร: กฎหมาย, ข้อบังคับ, สัญญา
   → Users: Legal team, Management
   → ใช้: ค้นหาข้อกฎหมาย, ตรวจสอบ compliance

3. IT Support Knowledge Base
   → เอกสาร: Technical docs, Runbooks, FAQs
   → Users: IT team, Support team
   → ใช้: ช่วย support tickets, troubleshooting

4. Sales Intelligence
   → เอกสาร: Product catalog, Pricing, Case studies
   → Users: Sales team
   → ใช้: ตอบคำถาม prospect แบบ real-time

5. Customer Support
   → เอกสาร: Product manual, FAQ, Policy
   → ผ่าน: Public API → integrate กับ website chatbot
   → Users: ลูกค้าภายนอก
```

---

## Quick Reference

```bash
# ============ SETUP ============
cp .env.example .env && nano .env
docker compose up -d
docker run -d -p 5001:5001 quay.io/docling-project/docling-serve:latest

# ============ INGEST ============
# Upload 1 file
curl -X POST localhost:8080/v1/documents/ingest \
  -H "Authorization: Bearer API_KEY" \
  -F "file=@doc.pdf" -F "allowed_groups=all-employees"

# ============ CHAT ============
curl -X POST localhost:8080/v1/chat \
  -H "Authorization: Bearer API_KEY" \
  -d '{"message": "your question", "session_id": "sess-001"}'

# ============ SEARCH ============
curl -X POST localhost:8080/v1/search \
  -H "Authorization: Bearer API_KEY" \
  -d '{"query": "search term", "k": 5}'

# ============ HEALTH ============
curl localhost:8080/health
curl localhost:5001/health
```

---

*เอกสารนี้ครอบคลุม: Setup → Ingest → RAG → API → Access Control → Production*
*Phase docs อื่นๆ อยู่ที่: [phase1-overview.md](./phase1-overview.md)*
