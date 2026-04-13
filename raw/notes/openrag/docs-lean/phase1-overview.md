# Phase 1: OpenRAG — ภาพรวมและสถาปัตยกรรม

## OpenRAG คืออะไร?

**OpenRAG** คือแพลตฟอร์ม Open-Source สำหรับทำ **RAG (Retrieval-Augmented Generation)** ซึ่งช่วยให้เราสามารถ:

1. **อัปโหลดเอกสาร** (PDF, DOCX, PPTX, รูปภาพ ฯลฯ)
2. **ประมวลผลและแยกเนื้อหา** จากเอกสารด้วย AI
3. **จัดเก็บเนื้อหาเป็น Vector** ใน Database
4. **ถามคำถาม** กับ AI ที่ตอบโดยอ้างอิงจากเอกสารที่เราอัปโหลด พร้อม Citation

---

## RAG (Retrieval-Augmented Generation) คืออะไร?

```
ปัญหา: LLM (ChatGPT, Claude ฯลฯ) รู้แค่ข้อมูลที่ถูก Train มา
       ไม่รู้เรื่องเอกสารภายในองค์กร หรือข้อมูลใหม่ล่าสุด

RAG แก้ปัญหา:
1. นำเอกสารของเรามาแยกย่อยเป็น Chunk เล็กๆ
2. แปลง Chunk แต่ละอันเป็น Vector (ตัวเลขที่แทนความหมาย)
3. เก็บ Vector ใน Database
4. เวลาถามคำถาม → ค้นหา Chunk ที่เกี่ยวข้อง → ส่งให้ LLM ตอบ
```

**ตัวอย่าง:**
> ถามว่า "นโยบายลาป่วยของบริษัทคือกี่วัน?"
> → ระบบค้นหาใน HR Policy PDF ที่อัปโหลดไว้
> → ดึง Chunk ที่เกี่ยวข้องมา
> → LLM ตอบพร้อมอ้างอิงจากเอกสาร

---

## Services ทั้งหมดใน OpenRAG

```
┌─────────────────────────────────────────────────────────────────┐
│                         OpenRAG Stack                           │
│                                                                 │
│  ┌─────────────┐    ┌──────────────────────────────────────┐   │
│  │  Frontend   │    │           Backend (FastAPI)           │   │
│  │  (Next.js)  │◄──►│  - API Endpoints (28 routes)         │   │
│  │  Port: 3000 │    │  - Auth (OAuth Google/Microsoft)     │   │
│  └─────────────┘    │  - Task Queue                        │   │
│                     │  - Document Management               │   │
│                     │  Port: 8080 → 8000                   │   │
│                     └──────────────┬───────────────────────┘   │
│                                    │                            │
│              ┌─────────────────────┼──────────────────┐        │
│              │                     │                  │        │
│              ▼                     ▼                  ▼        │
│  ┌───────────────────┐  ┌──────────────────┐  ┌──────────┐    │
│  │      Langflow     │  │   OpenSearch     │  │ Docling  │    │
│  │  (Workflow UI)    │  │  (Vector DB +    │  │ (AI Doc  │    │
│  │  Port: 7860       │  │   Search Engine) │  │  Parser) │    │
│  │                   │  │  Port: 9200      │  │ Port:5001│    │
│  └──────────┬────────┘  └──────────────────┘  └──────────┘    │
│             │                                       ▲           │
│             └───────────────────────────────────────┘          │
│                    Langflow เรียก Docling                        │
│                                                                 │
│  ┌──────────────────────────────────┐                           │
│  │    OpenSearch Dashboards         │                           │
│  │    (Monitoring UI) Port: 5601    │                           │
│  └──────────────────────────────────┘                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## Service แต่ละตัวทำอะไร?

| Service | บทบาท | Port |
|---------|--------|------|
| **Frontend** | Web UI สำหรับ upload เอกสาร, ค้นหา, chat | 3000 |
| **Backend** | FastAPI — รับ request, จัดการ flow ทั้งหมด | 8080 |
| **Langflow** | Visual workflow — pipeline ingestion + RAG agent | 7860 |
| **OpenSearch** | Vector Database — เก็บ embeddings + ค้นหา | 9200 |
| **Docling** | AI Document Parser — อ่าน PDF/DOCX/ภาพ | 5001 |
| **OS Dashboards** | UI ดู OpenSearch index และ data | 5601 |

---

## Data Flow หลัก 2 กระบวนการ

### 1. Ingestion Flow (อัปโหลดเอกสาร)

```
User Upload File
      │
      ▼
[Backend] รับไฟล์ สร้าง Task ID
      │
      ▼
[Langflow] Upload ไฟล์ → เรียก Ingestion Flow
      │
      ├──► [Docling] แยกเนื้อหา: Text, Tables, Images
      │         └── Return: Structured Markdown/JSON
      │
      ├──► [Text Splitter] แบ่ง text เป็น Chunks
      │         └── chunk_size: ~1000 tokens
      │             chunk_overlap: ~200 tokens
      │
      ├──► [Embedding Model] แปลง Chunk → Vector
      │         └── e.g. text-embedding-3-small → 1536 dimensions
      │
      └──► [OpenSearch] บันทึก Vector + Metadata
                └── fields: text, embeddings, filename, page, owner...
```

### 2. Chat/Query Flow (ถามคำถาม)

```
User Message: "ราคาสินค้า X คือเท่าไร?"
      │
      ▼
[Backend] รับ message → ส่งไป Langflow
      │
      ▼
[Langflow] เรียก RAG Agent Flow
      │
      ├──► [Embedding] แปลง user query → Vector
      │
      ├──► [OpenSearch] ค้นหา Chunks ที่ใกล้เคียงที่สุด (KNN)
      │         └── Return: Top-K chunks พร้อม similarity score
      │
      ├──► [LLM] ส่ง query + chunks → Generate Response
      │         └── Providers: OpenAI, Claude, WatsonX, Ollama
      │
      └──► [Response] ส่งกลับพร้อม Citations (อ้างอิงเอกสาร)
```

---

## ฟีเจอร์หลักของ OpenRAG

### Access Control
- ระบบ JWT Authentication
- Document-level permissions: `owner`, `allowed_users`, `allowed_groups`
- OAuth integration: Google + Microsoft

### Knowledge Filters
- สร้าง Filter เพื่อจำกัด search scope
- เช่น "ค้นหาเฉพาะเอกสาร HR" หรือ "เฉพาะเอกสารของ Department X"

### Multiple LLM Providers
```
Chat LLMs:
- OpenAI (GPT-4, GPT-4o, GPT-4o-mini)
- Anthropic (Claude 3.5 Sonnet, Haiku, Opus)
- IBM WatsonX
- Ollama (Local models)

Embedding Models:
- text-embedding-3-small (1536 dim) — Default
- text-embedding-3-large (3072 dim)
- IBM WatsonX embedding models (384-1024 dim)
```

### Connectors
- Google Drive
- SharePoint
- OneDrive
- URL ingestion (via MCP)

### Public API v1
- REST API สำหรับ integrate กับ Application อื่น
- API Key authentication

---

## โครงสร้างไฟล์โปรเจค

```
openrag/
├── docker-compose.yml          # 5-service orchestration
├── .env.example                # Environment variables template
├── .env                        # Actual config (ต้อง setup)
│
├── src/                        # Backend Python (FastAPI)
│   ├── main.py                 # FastAPI app + routes (988 lines)
│   ├── agent.py                # LLM integration (845 lines)
│   ├── api/                    # 28 API endpoint files
│   ├── services/               # Business logic (15 files)
│   ├── config/
│   │   └── settings.py         # Global config (871 lines)
│   ├── models/
│   ├── connectors/
│   └── utils/
│
├── frontend/                   # Next.js React App
│   ├── app/                    # Pages
│   ├── components/             # UI Components
│   └── hooks/                  # Custom hooks
│
├── flows/                      # Langflow Flow definitions
│   ├── ingestion_flow.json     # Document ingestion pipeline
│   ├── openrag_agent.json      # Chat/RAG agent
│   ├── openrag_url_mcp.json    # URL ingestion
│   └── openrag_nudges.json     # Contextual suggestions
│
├── config/
│   └── config.yaml             # User-level config (providers, models)
│
└── docs/
    └── learning/               # ← ไฟล์ที่คุณกำลังอ่านอยู่!
```

---

## การ Setup เบื้องต้น

```bash
# 1. Clone project
git clone <repo>
cd openrag

# 2. Copy env file
cp .env.example .env

# 3. แก้ไข .env — ต้องกำหนดค่าสำคัญ:
# OPENSEARCH_PASSWORD=YourStrongPassword123!
# SECRET_KEY=your-secret-key-here
# OPENAI_API_KEY=sk-...  (ถ้าใช้ OpenAI)

# 4. Start services
docker compose up -d

# 5. (Optional) Start Docling แยก
# ดู Phase 2 สำหรับ Docling setup
```

---

## สิ่งที่จะเรียนในแต่ละ Phase

| Phase | เนื้อหา |
|-------|---------|
| **Phase 1** (ไฟล์นี้) | ภาพรวม + สถาปัตยกรรม |
| **Phase 2** | Docling — Document Parsing ลึก |
| **Phase 3** | OpenSearch — Vector DB + Search |
| **Phase 4** | Langflow — Visual Workflow |
| **Phase 5** | Integration ทุก Service + ตัวอย่างจริง |

---

*ต่อไป: [Phase 2 — Docling](./phase2-docling.md)*
