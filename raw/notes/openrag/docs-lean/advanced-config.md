# Advanced Config — ตั้งค่า LLM, System Prompt, Models, Chunk

## ไฟล์ config หลักมี 2 ชั้น

```
ชั้นที่ 1: .env
  └── ค่า infrastructure (passwords, ports, URLs, feature flags)
  └── ไม่ค่อยเปลี่ยนหลัง setup

ชั้นที่ 2: config/config.yaml
  └── ค่า AI (model, system prompt, chunk size, embedding)
  └── เปลี่ยนได้ผ่าน UI Settings หรือแก้ไฟล์โดยตรง
  └── เปลี่ยนได้โดยไม่ต้อง restart
```

---

## config/config.yaml — โครงสร้างทั้งหมด

```yaml
agent:
  model: gpt-4.1-mini          # LLM สำหรับ chat
  provider: openai             # openai | anthropic | watsonx | ollama
  system_prompt: |             # พฤติกรรมของ AI
    You are a helpful assistant...

knowledge:
  embedding_model: text-embedding-3-small
  chunk_size: 1000
  chunk_overlap: 200
  ocr_enabled: false           # OCR สำหรับภาพใน PDF
  picture_descriptions: false  # อธิบายภาพด้วย AI

providers:
  openai:
    api_key: sk-...
    edited: true
  anthropic:
    api_key: ""
    edited: false
  watsonx:
    api_key: ""
    project_id: ""
    endpoint: ""
    edited: false
  ollama:
    endpoint: ""
    edited: false

onboarding:
  current_step: 4
```

---

## Section 1: agent — ตั้งค่า LLM

### เลือก Provider และ Model

```yaml
agent:
  provider: openai       # เปลี่ยนตรงนี้
  model: gpt-4o          # เปลี่ยนตาม provider
```

**Models ที่รองรับ (ครบทุกตัว):**

```
OpenAI:
  gpt-5
  gpt-4.5-preview
  gpt-4o                   ← แนะนำ (balance ดี)
  gpt-4o-mini              ← แนะนำ (ถูก + เร็ว)
  gpt-4.1
  gpt-4.1-mini             ← default ปัจจุบัน
  gpt-4.1-nano
  gpt-4-turbo
  gpt-3.5-turbo
  o1, o1-mini, o1-pro
  o3, o3-mini
  o4-mini

Anthropic (Claude):
  claude-opus-4-5-20251101
  claude-sonnet-4-5-20250929    ← แนะนำ (claude default)
  claude-haiku-4-5-20251001     ← เร็ว + ถูก
  claude-3-7-sonnet-20250219
  claude-3-5-sonnet-20241022
  claude-3-5-haiku-20241022
  claude-3-opus-20240229

IBM WatsonX:
  ibm/granite-3-8b-instruct
  ibm/granite-3-2b-instruct
  meta-llama/llama-3-70b-instruct
  meta-llama/llama-3-8b-instruct
  mistralai/mistral-large
  (+ อื่นๆ ตาม WatsonX catalog)

Ollama (Local):
  llama3.1, llama3.2
  mistral, mixtral
  gemma2
  qwen2.5
  deepseek-r1
  (ใช้ได้ทุก model ที่ pull ลง Ollama)
```

---

### System Prompt — กำหนดพฤติกรรม AI

นี่คือส่วนสำคัญที่สุดที่ควบคุมว่า AI จะตอบแบบไหน

**Default System Prompt (ย่อ):**
```yaml
agent:
  system_prompt: |
    You are a helpful assistant designed to help users find information
    from the documents in the knowledge base.

    You have access to a search tool that retrieves relevant passages
    from indexed documents. When answering questions:
    - Always use the search tool to find relevant information
    - Cite your sources by mentioning the document name and page
    - If information is not found, say so clearly
    - Do not make up information not present in the documents
```

**ตัวอย่าง Custom System Prompt สำหรับองค์กร:**

```yaml
# HR Bot
agent:
  system_prompt: |
    คุณคือผู้ช่วย HR ของบริษัท ABC ที่มีความรู้เรื่องนโยบายและกฎระเบียบของบริษัท
    ตอบคำถามเป็นภาษาไทยเสมอ ยกเว้นผู้ใช้ถามเป็นภาษาอื่น
    ให้ข้อมูลที่ถูกต้องและอ้างอิงจากเอกสารเสมอ
    ถ้าไม่มีข้อมูลในฐานความรู้ ให้แนะนำให้ติดต่อ HR โดยตรงที่ hr@company.com
    ไม่ตอบคำถามที่ไม่เกี่ยวกับการทำงานหรือนโยบายบริษัท

# Customer Support Bot
agent:
  system_prompt: |
    You are a customer support agent for ABC Company.
    Always be polite, professional, and helpful.
    Use the product documentation to answer questions accurately.
    If you cannot find the answer, escalate by saying:
    "Please contact our support team at support@abc.com"
    Never promise refunds or special deals not stated in the documents.

# Technical Documentation Bot
agent:
  system_prompt: |
    You are a technical assistant for the Engineering team.
    Answer questions about system architecture, APIs, and procedures.
    Always provide code examples when relevant.
    Use technical terminology appropriate for senior engineers.
    Reference specific document sections for all answers.
```

**วิธีแก้ System Prompt:**
```
Option 1: UI → Settings → System Prompt → แก้ไข → Save
Option 2: แก้ไฟล์ config/config.yaml โดยตรง (ไม่ต้อง restart)
```

---

## Section 2: knowledge — ตั้งค่า Embedding และ Chunking

```yaml
knowledge:
  embedding_model: text-embedding-3-small  # เปลี่ยนตาม provider
  chunk_size: 1000      # tokens ต่อ chunk
  chunk_overlap: 200    # tokens ที่ overlap ระหว่าง chunks
  ocr_enabled: false    # ใช้ OCR กับภาพใน PDF
  picture_descriptions: false  # อธิบายภาพด้วย Vision AI
```

### Embedding Models ตาม Provider:

```yaml
# ถ้าใช้ OpenAI:
embedding_model: text-embedding-3-small   # dimension: 1536 (default, ถูก)
embedding_model: text-embedding-3-large   # dimension: 3072 (แม่นกว่า, แพงกว่า)
embedding_model: text-embedding-ada-002   # dimension: 1536 (รุ่นเก่า)

# ถ้าใช้ IBM WatsonX:
embedding_model: ibm/slate-125m-english-rtrvr   # dimension: 768
embedding_model: ibm/slate-30m-english-rtrvr    # dimension: 384

# ถ้าใช้ Ollama (local, ฟรี):
embedding_model: nomic-embed-text    # dimension: 768
embedding_model: mxbai-embed-large   # dimension: 1024
embedding_model: all-minilm          # dimension: 384 (เล็ก เร็ว)
```

### Chunk Settings — ส่งผลโดยตรงต่อคุณภาพ RAG

```
chunk_size: 1000   (ค่า default)

เอกสารทั่วไป (Policy, FAQ, Manual):
  → chunk_size: 800-1200  ✅ ดีที่สุด

เอกสารที่มี context ยาว (Legal, Technical Spec):
  → chunk_size: 1500-2000
  → chunk_overlap: 300-400

เอกสารสั้นๆ (FAQ, Q&A format):
  → chunk_size: 400-600
  → chunk_overlap: 100

ตาราง/ข้อมูล structured:
  → chunk_size: 500
  → ควรให้แต่ละ chunk มีทั้ง header + data ของตาราง
```

### OCR และ Picture Descriptions

```yaml
# เปิด OCR — สำหรับ PDF ที่มีภาพที่มีข้อความ (เช่น scanned form)
ocr_enabled: true

# เปิด Picture Descriptions — AI อธิบายว่าภาพในเอกสารแสดงอะไร
# ต้องใช้ Vision model (OpenAI GPT-4o หรือ Claude ที่รองรับ vision)
picture_descriptions: true
# ⚠️ เปิดแล้วค่า API จะสูงขึ้น เพราะต้อง call vision model ทุกภาพ
```

---

## Section 3: providers — ตั้งค่า API Keys

```yaml
providers:
  openai:
    api_key: sk-proj-...
    edited: true      # ← true = ค่านี้จะไม่ถูก override จาก env var

  anthropic:
    api_key: sk-ant-...
    edited: true

  watsonx:
    api_key: your-ibm-api-key
    project_id: your-project-id
    endpoint: https://us-south.ml.cloud.ibm.com
    edited: true

  ollama:
    endpoint: http://host.docker.internal:11434
    edited: true
```

**Priority ของค่า config:**
```
1. config.yaml (ถ้า edited: true) ← ใช้ค่านี้ก่อน
2. .env file (OPENAI_API_KEY=...)
3. Default values

ถ้า edited: false → ระบบจะใช้ค่าจาก .env แทน
```

---

## Feature Flags ใน .env

```bash
# ===== Ingestion =====
DISABLE_INGEST_WITH_LANGFLOW=false
# false = ใช้ Langflow pipeline (default, แนะนำ)
# true  = ใช้ backend ingestion โดยตรง (ไม่ผ่าน Langflow, เร็วกว่า)

INGEST_SAMPLE_DATA=true
# true = โหลด sample documents ตอน startup (สำหรับ demo)
# false = เริ่มด้วย empty database

UPLOAD_BATCH_SIZE=25
# จำนวนไฟล์ที่ process พร้อมกัน (parallel ingestion)

DELETE_LANGFLOW_FILE_AFTER_INGESTION=true
# ลบไฟล์ออกจาก Langflow หลัง ingest เสร็จ (ประหยัด disk)

# ===== Performance =====
INGESTION_TIMEOUT=3600    # 60 นาที — timeout สำหรับ ingest ไฟล์เดียว
LANGFLOW_TIMEOUT=2400     # 40 นาที — รอ Langflow response
TASK_CLEANUP_DAYS=7       # ลบ task records ที่เก่ากว่า 7 วัน

# ===== Privacy =====
DO_NOT_TRACK=false
# true = ปิด telemetry ทั้งหมด (ไม่ส่งข้อมูลไปที่ Scarf)

# ===== Logging =====
LOG_LEVEL=INFO    # DEBUG | INFO | WARNING | ERROR
```

---

## Langfuse Tracing — ดู LLM Calls ละเอียด

Langfuse คือ observability platform สำหรับ LLM — ดูได้ว่า:
- แต่ละ query ใช้ prompt อะไร
- LLM ตอบอะไร
- ใช้ tokens กี่ตัว
- ค่าใช้จ่ายต่อ query

```bash
# .env — เปิดใช้ Langfuse
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_HOST=https://cloud.langfuse.com   # หรือ self-hosted
```

**ถ้ามี Langfuse — ดูได้:**
```
Langfuse Dashboard:
  ├── Traces — ทุก request แบบ end-to-end
  ├── Generations — LLM calls ทั้งหมด
  ├── Scores — quality scores
  └── Usage & Cost — ค่าใช้จ่ายรวม
```

---

## AWS S3 Integration

```bash
# .env — ตั้งค่า S3 สำหรับ ingest จาก S3 bucket
AWS_ACCESS_KEY_ID=AKIAXXXXXXXX
AWS_SECRET_ACCESS_KEY=xxxxxx
AWS_REGION=ap-southeast-1
S3_BUCKET_NAME=my-org-documents
```

```bash
# Ingest ไฟล์จาก S3
curl -X POST "http://localhost:8080/v1/documents/ingest" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "s3_key": "documents/hr/policy_2024.pdf",
    "allowed_groups": "all-employees"
  }'
```

---

## ปรับ Settings ผ่าน API

```bash
# ดู settings ปัจจุบัน
curl "http://localhost:8080/settings" \
  -H "Authorization: Bearer YOUR_API_KEY"

# Response:
{
  "agent": {
    "model": "gpt-4o-mini",
    "provider": "openai",
    "system_prompt": "You are a helpful assistant..."
  },
  "knowledge": {
    "embedding_model": "text-embedding-3-small",
    "chunk_size": 1000,
    "chunk_overlap": 200
  }
}

# แก้ไข settings
curl -X PUT "http://localhost:8080/settings" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "agent": {
      "model": "gpt-4o",
      "system_prompt": "คุณคือ HR Assistant..."
    },
    "knowledge": {
      "chunk_size": 1200,
      "chunk_overlap": 250
    }
  }'
```

---

## ดู Available Models ที่ระบบรองรับ

```bash
curl "http://localhost:8080/models" \
  -H "Authorization: Bearer YOUR_API_KEY"

# Response:
{
  "chat_models": [
    {"id": "gpt-4o", "provider": "openai", "available": true},
    {"id": "gpt-4o-mini", "provider": "openai", "available": true},
    {"id": "claude-sonnet-4-5", "provider": "anthropic", "available": false},
    {"id": "llama3.1", "provider": "ollama", "available": true}
  ],
  "embedding_models": [
    {"id": "text-embedding-3-small", "provider": "openai", "dimension": 1536},
    {"id": "nomic-embed-text", "provider": "ollama", "dimension": 768}
  ]
}
```

---

## Task Queue — การทำงานเบื้องหลัง

```
เมื่อ ingest ไฟล์:
  1. Backend สร้าง Task (pending)
  2. Task ถูกใส่ใน Queue
  3. Worker หยิบ Task ไปทำ (processing)
  4. Task เสร็จ → success หรือ failed

Parallel Workers:
  UPLOAD_BATCH_SIZE=25 → process 25 ไฟล์พร้อมกัน
```

**Task States:**
```
pending    → รอคิว
processing → กำลังทำงาน
success    → เสร็จแล้ว
failed     → ล้มเหลว (ดู error message)
cancelled  → ถูกยกเลิก
```

**Retry & Timeout:**
```python
# ถ้า task ทำงานนาน > INGESTION_TIMEOUT → ถือว่า timeout
# ไม่มี auto-retry — ต้อง upload ใหม่เอง

# ดู error ของ task ที่ fail:
GET /tasks/{task_id}
→ { "status": "failed", "error": "Docling timeout after 3600s" }
```

---

## Telemetry — ข้อมูลที่ส่งออก

OpenRAG ส่ง telemetry ไปที่ Scarf (privacy-respecting analytics):

**ส่งข้อมูลอะไร:**
```
✅ ส่ง: ประเภทเหตุการณ์ (startup, ingest, chat)
✅ ส่ง: OS, GPU availability
✅ ส่ง: anonymous usage counts

❌ ไม่ส่ง: เนื้อหาเอกสาร
❌ ไม่ส่ง: คำถาม/คำตอบ
❌ ไม่ส่ง: API keys
❌ ไม่ส่ง: ข้อมูลส่วนตัว
```

**ปิด Telemetry:**
```bash
# .env
DO_NOT_TRACK=true
```

---

*กลับไป: [org-rag-guide.md](./org-rag-guide.md)*
*ต่อไป: [cost-and-models.md](./cost-and-models.md)*
