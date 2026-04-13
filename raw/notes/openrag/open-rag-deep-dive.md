# Open Source RAG Platforms — Deep Dive ฉบับสมบูรณ์

> เปรียบเทียบทุก Open Source RAG Platform ที่พร้อม Deploy ให้คนภายนอกใช้ได้ทันที
> รวมถึง Framework อย่าง LangChain, LlamaIndex, Haystack

---

## สารบัญ

1. [ภาพรวม — แยกประเภทให้ชัด](#1-ภาพรวม--แยกประเภทให้ชัด)
2. [RAGFlow — Deep Dive](#2-ragflow--deep-dive)
3. [Dify — Deep Dive](#3-dify--deep-dive)
4. [Anything LLM — Deep Dive](#4-anything-llm--deep-dive)
5. [Verba (Weaviate) — Deep Dive](#5-verba-weaviate--deep-dive)
6. [Quivr — Deep Dive](#6-quivr--deep-dive)
7. [Danswer (ปัจจุบันชื่อ Onyx) — Deep Dive](#7-danswer-ปัจจุบันชื่อ-onyx--deep-dive)
8. [PrivateGPT — Deep Dive](#8-privategpt--deep-dive)
9. [Kotaemon — Deep Dive](#9-kotaemon--deep-dive)
10. [Flowise & Langflow — Deep Dive](#10-flowise--langflow--deep-dive)
11. [Framework: LangChain / LlamaIndex / Haystack](#11-framework-langchain--llamaindex--haystack)
12. [เปรียบเทียบทั้งหมด — ตารางใหญ่](#12-เปรียบเทียบทั้งหมด--ตารางใหญ่)
13. [เปรียบเทียบ Architecture](#13-เปรียบเทียบ-architecture)
14. [เปรียบเทียบ Chunking Support](#14-เปรียบเทียบ-chunking-support)
15. [เปรียบเทียบ Retrieval Methods](#15-เปรียบเทียบ-retrieval-methods)
16. [เปรียบเทียบ Deployment](#16-เปรียบเทียบ-deployment)
17. [แนะนำตามสถานการณ์](#17-แนะนำตามสถานการณ์)

---

## 1. ภาพรวม — แยกประเภทให้ชัด

Open Source RAG แบ่งเป็น 3 ประเภท:

### ประเภท A: RAG Application (พร้อมใช้เลย)

มี UI + API + RAG Pipeline ครบจบ ไม่ต้องเขียน code (หรือเขียนน้อยมาก)

```
RAGFlow, Dify, AnythingLLM, Quivr, Danswer/Onyx,
PrivateGPT, Kotaemon, Verba
```

### ประเภท B: Low-Code Builder (ลาก วาง สร้าง Pipeline)

มี UI สำหรับสร้าง RAG pipeline แบบ visual

```
Flowise, Langflow
```

### ประเภท C: Framework (เขียน Code)

Library ที่ต้องเขียน code เอง แต่ยืดหยุ่นสูงสุด

```
LangChain, LlamaIndex, Haystack
```

```
ง่าย / พร้อมใช้ ←────────────────────→ ยืดหยุ่น / ต้องเขียน

RAG Application      Low-Code Builder       Framework
(RAGFlow, Dify)      (Flowise, Langflow)    (LangChain, LlamaIndex)

deploy ได้เลย        ลาก วาง               เขียน code เต็มที่
custom ได้จำกัด      custom ปานกลาง         custom ได้ทุกอย่าง
```

---

## 2. RAGFlow — Deep Dive

> **"Open Source RAG Engine ที่เน้น Deep Document Understanding"**

- GitHub: github.com/infiniflow/ragflow
- Stars: 30k+ (เติบโตเร็วมาก)
- สร้างโดย: InfiniFlow (จีน)
- ภาษา: Python + TypeScript

### จุดเด่นหลัก

RAGFlow ไม่ได้แค่ chunk แบบธรรมดา มันมี **Document Layout Analysis** ที่เข้าใจโครงสร้างเอกสารจริงๆ เช่น ตาราง, รูปภาพ, header, footer, multi-column layout ซึ่ง framework อื่นทำไม่ได้ดีเท่า

### Architecture

```
┌──────────────────────────────────────────────┐
│                  RAGFlow                      │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Document │  │ Chunking │  │ Retrieval│   │
│  │ Parser   │  │ Engine   │  │ Engine   │   │
│  │          │  │          │  │          │   │
│  │ - OCR    │  │ - Naive  │  │ - Vector │   │
│  │ - Layout │  │ - Book   │  │ - BM25   │   │
│  │ - Table  │  │ - Paper  │  │ - Hybrid │   │
│  │ - Image  │  │ - Manual │  │ - Rerank │   │
│  └──────────┘  │ - QA     │  └──────────┘   │
│                │ - Table  │                  │
│  ┌──────────┐  │ - Resume │  ┌──────────┐   │
│  │ LLM      │  │ - Law    │  │ Database │   │
│  │          │  └──────────┘  │          │   │
│  │ - OpenAI │                │ - Elastic│   │
│  │ - Claude │                │ - Infinity│  │
│  │ - Ollama │                │ (vector) │   │
│  │ - Gemini │                └──────────┘   │
│  └──────────┘                               │
│                                              │
│  ┌──────────────────────────────────────┐    │
│  │          Web UI + REST API           │    │
│  └──────────────────────────────────────┘    │
└──────────────────────────────────────────────┘
```

### Deep Document Understanding

นี่คือสิ่งที่ทำให้ RAGFlow ต่างจากตัวอื่น:

```
PDF ที่มีตาราง + รูป + multi-column:

ตัวอื่น (LangChain, LlamaIndex):
  → อ่าน text เรียงๆ → ตารางกลายเป็น text มั่วๆ
  → "ชื่อ สมชาย อายุ 30 แผนก IT เงินเดือน 50000"
  → chunk ตัดกลางตาราง → ข้อมูลผิด

RAGFlow:
  → Layout Analysis ตรวจจับตาราง
  → แปลงตารางเป็น structured data
  → | ชื่อ    | อายุ | แผนก | เงินเดือน |
    | สมชาย  | 30   | IT   | 50,000   |
  → chunk ไม่ตัดกลางตาราง
```

### Chunking Methods ใน RAGFlow

RAGFlow มี template-based chunking ที่ออกแบบมาสำหรับเอกสารแต่ละประเภท:

```
Template        เหมาะกับ                  วิธีตัด
─────────────────────────────────────────────────────────────
Naive           เอกสารทั่วไป              fixed size + overlap
Book            หนังสือ, manual           ตาม chapter/section/heading
Paper           งานวิจัย, academic paper   ตาม abstract/intro/method/result
Manual          คู่มือ                    ตาม section + step
QA              FAQ document              ตาม question-answer pairs
Table           ตาราง                     ตามแถว/กลุ่มแถว
Resume          resume/CV                 ตาม section (education/exp)
Law             เอกสารกฎหมาย              ตามมาตรา/section
Presentation    PowerPoint                ตาม slide
Picture         รูปภาพ                    OCR + description
One            เอกสารสั้น                  ไม่ตัด เก็บทั้งอัน
```

### การติดตั้งและใช้งาน

```bash
# ติดตั้ง
git clone https://github.com/infiniflow/ragflow.git
cd ragflow

# config (เลือก LLM, embedding model)
# แก้ไฟล์ .env

# รัน
docker compose up -d

# เปิด http://localhost:80
# สร้าง account → สร้าง Knowledge Base → อัพโหลดเอกสาร → สร้าง Chat
```

### API ที่ได้

```bash
# สร้าง dataset
curl -X POST http://localhost/api/v1/datasets \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "HR Documents", "chunk_method": "naive"}'

# อัพโหลดเอกสาร
curl -X POST http://localhost/api/v1/datasets/{dataset_id}/documents \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -F "file=@hr_policy.pdf"

# ถามคำถาม
curl -X POST http://localhost/api/v1/chats/{chat_id}/completions \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "ลาป่วยได้กี่วัน?"}]}'
```

### ข้อดี / ข้อเสีย

```
ข้อดี:
  ✅ Document parsing ดีที่สุด (ตาราง, รูป, layout)
  ✅ Template chunking สำหรับเอกสารหลายประเภท
  ✅ Hybrid search (vector + BM25) ในตัว
  ✅ ดู chunk ได้ก่อน query (transparency)
  ✅ Citation/Reference ชี้ไปที่ต้นฉบับได้
  ✅ Multi-tenant รองรับหลาย user
  ✅ API พร้อม

ข้อเสีย:
  ❌ ใช้ RAM/CPU เยอะ (document parsing หนัก)
  ❌ Docker image ใหญ่มาก (~10GB+)
  ❌ Documentation ยังไม่สมบูรณ์ (ภาษาจีนเยอะ)
  ❌ Community เล็กกว่า LangChain/LlamaIndex
  ❌ Custom pipeline ทำได้จำกัด
  ❌ Agent/Tool calling ยังไม่แข็งแรง
```

---

## 3. Dify — Deep Dive

> **"Open Source LLM Application Platform — ครบจบในตัว"**

- GitHub: github.com/langgenius/dify
- Stars: 55k+ (ใหญ่ที่สุด)
- สร้างโดย: LangGenius (จีน)
- ภาษา: Python + TypeScript (Next.js)

### Architecture

```
┌──────────────────────────────────────────────┐
│                    Dify                       │
│                                              │
│  ┌──────────────────────────────────────┐    │
│  │           Application Layer          │    │
│  │  - Chatbot  - Agent  - Workflow      │    │
│  │  - Text Generation  - Completion     │    │
│  └──────────────────────────────────────┘    │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Knowledge│  │ Workflow │  │ Model    │   │
│  │ Base     │  │ Engine   │  │ Provider │   │
│  │          │  │          │  │          │   │
│  │ - Upload │  │ - IF/ELSE│  │ - OpenAI │   │
│  │ - Chunk  │  │ - Loop   │  │ - Claude │   │
│  │ - Embed  │  │ - HTTP   │  │ - Gemini │   │
│  │ - Index  │  │ - Code   │  │ - Ollama │   │
│  └──────────┘  │ - LLM    │  │ - HuggingF│  │
│                │ - Tool   │  └──────────┘   │
│  ┌──────────┐  └──────────┘                  │
│  │ Vector DB│                ┌──────────┐    │
│  │ - Qdrant │                │ External │    │
│  │ - Weaviate│               │ Tools    │    │
│  │ - pgvector│               │ - API    │    │
│  │ - Milvus │                │ - Search │    │
│  └──────────┘                └──────────┘    │
│                                              │
│  ┌──────────────────────────────────────┐    │
│  │    Web UI + REST API + Analytics     │    │
│  └──────────────────────────────────────┘    │
└──────────────────────────────────────────────┘
```

### Knowledge Base (RAG)

```
Chunking Options:
  - Automatic: Dify เลือกให้ตาม document type
  - Custom:
    - Chunk size (100-4000)
    - Chunk overlap
    - Separator (custom ได้)
  - Parent-Child chunking (v0.8+)

Retrieval Options:
  - Vector search
  - Full-text search (BM25)
  - Hybrid search
  - N results (top_k)
  - Score threshold
  - Reranking (Cohere, Jina)

Indexing Options:
  - High Quality (embedding model)
  - Economy (keyword index, ไม่ต้องใช้ embedding)
```

### Workflow Engine

Dify ไม่ใช่แค่ RAG มันมี visual workflow builder:

```
[Start] → [Knowledge Retrieval] → [IF score > 0.7]
                                        ↓ Yes        ↓ No
                                   [LLM Answer]  [Web Search]
                                        ↓              ↓
                                   [Response]     [LLM Answer]
                                                       ↓
                                                  [Response]
```

### การติดตั้ง

```bash
git clone https://github.com/langgenius/dify.git
cd dify/docker
cp .env.example .env
docker compose up -d

# เปิด http://localhost/install
# ตั้ง admin account → เริ่มใช้งาน
```

### ข้อดี / ข้อเสีย

```
ข้อดี:
  ✅ UI สวยที่สุด ใช้ง่ายที่สุด
  ✅ Workflow engine ทรงพลัง (IF/ELSE, Loop, Code)
  ✅ รองรับ LLM provider เยอะมาก (50+)
  ✅ Analytics dashboard ในตัว
  ✅ Multi-tenant + Team collaboration
  ✅ API key management
  ✅ Embed widget สำหรับฝังเว็บ
  ✅ Community ใหญ่ที่สุด
  ✅ มี Cloud version (dify.ai) ถ้าไม่อยาก host เอง

ข้อเสีย:
  ❌ Document parsing ไม่ดีเท่า RAGFlow (ตาราง, layout)
  ❌ Chunking options จำกัดกว่า (ไม่มี semantic chunking)
  ❌ Custom retrieval logic ทำได้จำกัด
  ❌ ช้าถ้าเอกสารเยอะมาก
  ❌ License: Source Available (ไม่ใช่ pure open source)
```

---

## 4. Anything LLM — Deep Dive

> **"All-in-one Desktop & Docker AI application with built-in RAG"**

- GitHub: github.com/Mintplex-Labs/anything-llm
- Stars: 35k+
- สร้างโดย: Mintplex Labs
- ภาษา: JavaScript (Node.js + React)

### จุดเด่น

AnythingLLM เน้น **ง่าย** และ **privacy** มากที่สุด มี desktop app ดาวน์โหลดมารันบนเครื่องได้เลย ไม่ต้อง Docker

### Architecture

```
┌──────────────────────────────────────┐
│           Anything LLM               │
│                                      │
│  ┌────────────┐   ┌──────────────┐   │
│  │ Document   │   │ LLM Provider │   │
│  │ Processor  │   │              │   │
│  │            │   │ - OpenAI     │   │
│  │ - PDF      │   │ - Claude     │   │
│  │ - DOCX     │   │ - Ollama     │   │
│  │ - TXT      │   │ - LM Studio  │   │
│  │ - Web URL  │   │ - LocalAI    │   │
│  │ - YouTube  │   └──────────────┘   │
│  │ - Confluence│                     │
│  │ - GitHub   │   ┌──────────────┐   │
│  └────────────┘   │ Vector DB    │   │
│                   │              │   │
│  ┌────────────┐   │ - LanceDB   │   │
│  │ Workspaces │   │ - Chroma    │   │
│  │ (multi-    │   │ - Pinecone  │   │
│  │  tenant)   │   │ - Qdrant    │   │
│  └────────────┘   │ - Weaviate  │   │
│                   └──────────────┘   │
│                                      │
│  ┌──────────────────────────────┐    │
│  │   Desktop App / Web UI / API │    │
│  └──────────────────────────────┘    │
└──────────────────────────────────────┘
```

### การติดตั้ง

```bash
# วิธีที่ 1: Desktop App (ง่ายสุด)
# ดาวน์โหลดจาก https://anythingllm.com/download
# เปิดแอป → ตั้งค่า LLM → อัพโหลดเอกสาร → ถามคำถาม

# วิธีที่ 2: Docker
docker pull mintplexlabs/anythingllm
docker run -d -p 3001:3001 \
  -v ./storage:/app/server/storage \
  mintplexlabs/anythingllm
```

### ข้อดี / ข้อเสีย

```
ข้อดี:
  ✅ มี Desktop App (ง่ายที่สุดในทุกตัว)
  ✅ รองรับ local LLM (Ollama, LM Studio) ดีมาก
  ✅ Privacy-first (รันบนเครื่องได้ 100%)
  ✅ Workspace แยก context ได้
  ✅ Agent mode ใช้ tools ได้
  ✅ API endpoint พร้อม
  ✅ Multi-user support

ข้อเสีย:
  ❌ Chunking options น้อย (fixed size เป็นหลัก)
  ❌ ไม่มี hybrid search
  ❌ ไม่มี reranking
  ❌ Document parsing ธรรมดา
  ❌ Workflow/Pipeline ไม่มี
  ❌ Scale ยาก (ไม่ได้ออกแบบมาสำหรับ enterprise)
```

---

## 5. Verba (Weaviate) — Deep Dive

> **"The Golden RAGtriever — RAG application by Weaviate"**

- GitHub: github.com/weaviate/Verba
- Stars: 6k+
- สร้างโดย: Weaviate team
- ภาษา: Python + React

### จุดเด่น

สร้างโดยทีม Weaviate (vector database) ดังนั้น integration กับ Weaviate ดีมาก hybrid search ทำงานได้ดีที่สุดเพราะ Weaviate รองรับ native

### ข้อดี / ข้อเสีย

```
ข้อดี:
  ✅ Hybrid search ดีมาก (Weaviate native)
  ✅ UI สวย ใช้ง่าย
  ✅ Suggestion queries
  ✅ Semantic caching
  ✅ รองรับหลาย embedding model

ข้อเสีย:
  ❌ ผูกกับ Weaviate เท่านั้น
  ❌ Chunking options จำกัด
  ❌ Community เล็ก
  ❌ Feature น้อยกว่า Dify/RAGFlow มาก
  ❌ ไม่มี workflow engine
```

---

## 6. Quivr — Deep Dive

> **"Your Second Brain — AI-powered personal knowledge manager"**

- GitHub: github.com/QuivrHQ/quivr
- Stars: 37k+
- สร้างโดย: Quivr team (ฝรั่งเศส)
- ภาษา: Python + TypeScript

### จุดเด่น

Quivr เน้นเป็น **"Second Brain"** คือ personal knowledge management ที่ใช้ AI

### Architecture

```
┌──────────────────────────────────────┐
│              Quivr                    │
│                                      │
│  ┌──────────┐   ┌──────────────┐     │
│  │ Brain    │   │ Supabase     │     │
│  │ (แต่ละ   │   │ (pgvector)   │     │
│  │  หัวข้อ) │   │              │     │
│  │          │   │ - Auth       │     │
│  │ Brain A  │   │ - Storage    │     │
│  │ Brain B  │   │ - Vector DB  │     │
│  │ Brain C  │   │ - Database   │     │
│  └──────────┘   └──────────────┘     │
│                                      │
│  ┌──────────────────────────────┐    │
│  │      Web UI + REST API       │    │
│  └──────────────────────────────┘    │
└──────────────────────────────────────┘
```

### ข้อดี / ข้อเสีย

```
ข้อดี:
  ✅ Brain concept (แยกหัวข้อได้)
  ✅ Supabase backend (pgvector, auth, storage ครบ)
  ✅ Share brain กับคนอื่นได้
  ✅ Multi-model support
  ✅ มี Cloud version

ข้อเสีย:
  ❌ ผูกกับ Supabase
  ❌ Self-host ซับซ้อน (dependency เยอะ)
  ❌ Chunking options จำกัด
  ❌ ไม่มี hybrid search / reranking
  ❌ เปลี่ยน direction บ่อย (API เปลี่ยนบ่อย)
```

---

## 7. Danswer (ปัจจุบันชื่อ Onyx) — Deep Dive

> **"Open Source Enterprise Search + AI Assistant"**

- GitHub: github.com/onyx-dot-com/onyx (เดิม danswer-ai/danswer)
- Stars: 13k+
- สร้างโดย: Onyx team (USA)
- ภาษา: Python + TypeScript

### จุดเด่น

Danswer/Onyx เน้น **Enterprise Search** คือค้นหาข้อมูลจากทุกแหล่งในองค์กร (Slack, Confluence, Google Drive, Notion, GitHub, etc.) แล้วตอบคำถามด้วย AI

### Connectors (40+)

```
Slack, Confluence, Jira, Notion, Google Drive,
GitHub, GitLab, Linear, Zendesk, Hubspot,
Salesforce, SharePoint, Bookstack, Discourse,
Dropbox, File System, Web Scraping, ...
```

### ข้อดี / ข้อเสีย

```
ข้อดี:
  ✅ Connector เยอะที่สุด (40+) ดึงข้อมูลจากทุกที่
  ✅ Hybrid search (vector + keyword) ในตัว
  ✅ Permission-aware (เห็นเฉพาะที่มีสิทธิ์)
  ✅ Real-time sync กับแหล่งข้อมูล
  ✅ Multi-tenant + Teams
  ✅ ออกแบบมาสำหรับ Enterprise จริงจัง

ข้อเสีย:
  ❌ Setup ซับซ้อน (หลาย service)
  ❌ ใช้ resource เยอะ
  ❌ Chunking ปรับแต่งได้น้อย
  ❌ ไม่มี workflow engine
  ❌ เปลี่ยนชื่อ/direction (Danswer → Onyx)
```

---

## 8. PrivateGPT — Deep Dive

> **"Interact with documents using LLMs, 100% private, no data leaks"**

- GitHub: github.com/zylon-ai/private-gpt
- Stars: 55k+ (stars เยอะที่สุด)
- สร้างโดย: Zylon
- ภาษา: Python

### จุดเด่น

เน้น **Privacy 100%** ทุกอย่างรันบนเครื่อง ไม่ส่งข้อมูลออกเลย ใช้ local LLM + local embedding

### ข้อดี / ข้อเสีย

```
ข้อดี:
  ✅ Privacy 100% (ไม่ส่งข้อมูลออก)
  ✅ Local LLM support ดี (llama.cpp, Ollama)
  ✅ API เป็น OpenAI-compatible format
  ✅ ใช้ง่าย (Python package)
  ✅ Stars เยอะ community ใหญ่

ข้อเสีย:
  ❌ Feature น้อย (เน้น simplicity)
  ❌ UI พื้นฐานมาก
  ❌ ไม่มี hybrid search
  ❌ ไม่มี reranking
  ❌ ไม่มี workflow
  ❌ ต้องมี GPU ที่แรงพอสำหรับ local LLM
```

---

## 9. Kotaemon — Deep Dive

> **"Open Source RAG-based tool for chatting with documents"**

- GitHub: github.com/Cinnamon/kotaemon
- Stars: 20k+
- สร้างโดย: Cinnamon AI (เวียดนาม)
- ภาษา: Python

### จุดเด่น

สร้างโดยทีมจาก Southeast Asia เข้าใจเอกสารเอเชียดี รองรับ complex document layout

### ข้อดี / ข้อเสีย

```
ข้อดี:
  ✅ Document parsing ดี (ตาราง, multi-column)
  ✅ Multi-modal RAG (text + image)
  ✅ GraphRAG support (knowledge graph)
  ✅ Citation ชี้ไปหน้าที่อ้างอิงได้
  ✅ รองรับหลาย LLM + Embedding
  ✅ UI ใช้ง่าย (Gradio-based)
  ✅ ติดตั้งง่าย

ข้อเสีย:
  ❌ UI ไม่สวยเท่า Dify (Gradio)
  ❌ Multi-tenant จำกัด
  ❌ API ไม่ครบ
  ❌ Workflow ไม่มี
  ❌ Community เล็ก
```

---

## 10. Flowise & Langflow — Deep Dive

### Flowise

> **"Drag & drop UI to build LLM flows"**

```
ลาก วาง component → ต่อเส้น → ได้ RAG pipeline + API

Components: Document Loaders, Text Splitters, Vector Stores,
            Embeddings, Chat Models, Chains, Agents, Tools

ข้อดี:  เห็น pipeline ชัด, ได้ API ทันที, community ดี
ข้อเสีย: ซับซ้อนขึ้นต้อง code, debug ยาก, performance
```

### Langflow

> **"Visual framework for building multi-agent and RAG applications"**

```
สร้างโดย LangChain team → integrate กับ LangChain ecosystem ดี

ข้อดี:  LangChain component ครบ, export เป็น code ได้
ข้อเสีย: ช้ากว่า Flowise, ยังไม่ stable เท่า
```

---

## 11. Framework: LangChain / LlamaIndex / Haystack

(สรุปสั้นๆ เพราะอธิบายไว้ละเอียดแล้วในไฟล์ก่อนหน้า)

### LangChain

```
จุดเด่น: ecosystem ใหญ่สุด, LangServe deploy ง่าย, TypeScript support
จุดด้อย: abstraction เยอะ, API เปลี่ยนบ่อย
เหมาะกับ: Agent + RAG, ระบบซับซ้อน
```

### LlamaIndex

```
จุดเด่น: RAG ดีที่สุด, chunking ครบที่สุด, เขียนน้อยได้เยอะ
จุดด้อย: ไม่มี server ในตัว
เหมาะกับ: RAG เป็นงานหลัก, document-heavy
```

### Haystack

```
จุดเด่น: pipeline ยืดหยุ่นสุด, production-ready, debug ง่าย
จุดด้อย: community เล็ก, เขียนเยอะกว่า
เหมาะกับ: enterprise production pipeline
```

---

## 12. เปรียบเทียบทั้งหมด — ตารางใหญ่

### พื้นฐาน

| Platform     | ประเภท            | GitHub Stars | ภาษา         | License          |
| ------------ | ----------------- | ------------ | ------------ | ---------------- |
| RAGFlow      | RAG App           | 30k+         | Python+TS    | Apache 2.0       |
| Dify         | LLM Platform      | 55k+         | Python+TS    | Source Available |
| AnythingLLM  | RAG App           | 35k+         | Node.js      | MIT              |
| Quivr        | RAG App           | 37k+         | Python+TS    | Source Available |
| Danswer/Onyx | Enterprise Search | 13k+         | Python+TS    | MIT              |
| PrivateGPT   | RAG App           | 55k+         | Python       | Apache 2.0       |
| Kotaemon     | RAG App           | 20k+         | Python       | Apache 2.0       |
| Verba        | RAG App           | 6k+          | Python+React | BSD              |
| Flowise      | Low-Code          | 33k+         | TypeScript   | Apache 2.0       |
| Langflow     | Low-Code          | 40k+         | Python+TS    | MIT              |
| LangChain    | Framework         | 98k+         | Python+TS    | MIT              |
| LlamaIndex   | Framework         | 38k+         | Python+TS    | MIT              |
| Haystack     | Framework         | 18k+         | Python       | Apache 2.0       |

### Features

| Platform | UI | API | Workflow | Agent | Multi-user | Analytics |
|----------|-----|-----|----------|-------|------------|-----------|
| RAGFlow | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ |
| Dify | ✅ Best | ✅ | ✅ Best | ✅ | ✅ | ✅ |
| AnythingLLM | ✅ | ✅ | ❌ | ✅ | ✅ | ❌ |
| Quivr | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ |
| Danswer/Onyx | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ |
| PrivateGPT | ⚠️ Basic | ✅ | ❌ | ❌ | ❌ | ❌ |
| Kotaemon | ✅ | ⚠️ | ❌ | ❌ | ⚠️ | ❌ |
| Verba | ✅ | ⚠️ | ❌ | ❌ | ❌ | ❌ |
| Flowise | ✅ Visual | ✅ | ✅ Visual | ✅ | ⚠️ | ❌ |
| Langflow | ✅ Visual | ✅ | ✅ Visual | ✅ | ⚠️ | ❌ |
| LangChain | ❌ | ✅ LangServe | ✅ Code | ✅ | ❌ (DIY) | ❌ (LangSmith) |
| LlamaIndex | ❌ | ❌ (DIY) | ✅ Code | ✅ | ❌ (DIY) | ❌ |
| Haystack | ❌ | ✅ Hayhooks | ✅ Code | ✅ | ❌ (DIY) | ❌ |

---

## 13. เปรียบเทียบ Architecture

| Platform | Vector Store | Document Store | Database |
|----------|-------------|---------------|----------|
| RAGFlow | Elasticsearch + InfinityDB | MinIO | MySQL |
| Dify | Qdrant/Weaviate/pgvector/Milvus | S3/local | PostgreSQL |
| AnythingLLM | LanceDB/Chroma/Pinecone/Qdrant | local | SQLite |
| Quivr | pgvector (Supabase) | Supabase Storage | Supabase (Postgres) |
| Danswer/Onyx | Vespa | local | PostgreSQL |
| PrivateGPT | Qdrant/Chroma | local | SQLite |
| Kotaemon | Chroma/LanceDB | local | SQLite |
| Verba | Weaviate only | Weaviate | Weaviate |

---

## 14. เปรียบเทียบ Chunking Support

| Platform | Fixed | Sentence | Recursive | Semantic | Hierarchical | Structure | Custom |
|----------|-------|----------|-----------|----------|-------------|-----------|--------|
| RAGFlow | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ Best (template) | ❌ |
| Dify | ✅ | ✅ | ❌ | ❌ | ✅ (v0.8+) | ❌ | ❌ |
| AnythingLLM | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Quivr | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Danswer/Onyx | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| PrivateGPT | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Kotaemon | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Verba | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| Flowise | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ |
| Langflow | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ❌ |
| **LangChain** | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ |
| **LlamaIndex** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Haystack** | ✅ | ✅ | ❌ | ❌ | ❌ | ⚠️ | ✅ |

**สรุป Chunking: LlamaIndex ครบที่สุด, RAGFlow เด่นเรื่อง template-based**

---

## 15. เปรียบเทียบ Retrieval Methods

| Platform | Vector | BM25/Keyword | Hybrid | Reranking | Multi-Query | Metadata Filter |
|----------|--------|-------------|--------|-----------|-------------|-----------------|
| RAGFlow | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| Dify | ✅ | ✅ | ✅ | ✅ (Cohere/Jina) | ❌ | ✅ |
| AnythingLLM | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Quivr | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Danswer/Onyx | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| PrivateGPT | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Kotaemon | ✅ | ❌ | ✅ | ✅ | ❌ | ❌ |
| Verba | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| Flowise | ✅ | ⚠️ | ⚠️ | ⚠️ | ❌ | ✅ |
| Langflow | ✅ | ⚠️ | ⚠️ | ⚠️ | ❌ | ✅ |
| **LangChain** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **LlamaIndex** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Haystack** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

**สรุป Retrieval: Framework (LangChain/LlamaIndex/Haystack) ครบที่สุด, RAG App ที่ดีสุดคือ RAGFlow + Danswer**

---

## 16. เปรียบเทียบ Deployment

| Platform | Docker | Desktop App | Cloud Hosted | Deploy ง่าย |
|----------|--------|-------------|-------------|------------|
| RAGFlow | ✅ (หนักมาก) | ❌ | ❌ | ⭐⭐ |
| Dify | ✅ | ❌ | ✅ dify.ai | ⭐⭐⭐⭐ |
| AnythingLLM | ✅ | ✅ Best | ✅ | ⭐⭐⭐⭐⭐ |
| Quivr | ✅ (ซับซ้อน) | ❌ | ✅ quivr.com | ⭐⭐ |
| Danswer/Onyx | ✅ (ซับซ้อน) | ❌ | ✅ | ⭐⭐ |
| PrivateGPT | ✅ | ❌ | ❌ | ⭐⭐⭐ |
| Kotaemon | ✅ | ❌ | ❌ | ⭐⭐⭐⭐ |
| Verba | ✅ | ❌ | ❌ | ⭐⭐⭐ |
| Flowise | ✅ | ❌ | ✅ | ⭐⭐⭐⭐ |
| Langflow | ✅ | ❌ | ✅ | ⭐⭐⭐ |
| LangChain | ✅ (DIY) | ❌ | ✅ LangSmith | ⭐⭐⭐ |
| LlamaIndex | ✅ (DIY) | ❌ | ✅ LlamaCloud | ⭐⭐ |
| Haystack | ✅ (DIY) | ❌ | ✅ deepset Cloud | ⭐⭐ |

---

## 17. แนะนำตามสถานการณ์

### "อยากได้ RAG ใช้เลย ไม่อยากเขียน code"

```
เอกสารทั่วไป (PDF, Word, TXT)     → Dify (UI ดีสุด, ครบสุด)
เอกสารซับซ้อน (ตาราง, multi-column) → RAGFlow (document parsing ดีสุด)
ใช้บนเครื่องตัวเอง privacy 100%    → AnythingLLM Desktop
Enterprise หลายแหล่งข้อมูล         → Danswer/Onyx (connector เยอะสุด)
```

### "อยาก custom ได้บ้าง แต่ไม่อยากเขียนเยอะ"

```
ลาก วาง สร้าง pipeline    → Flowise หรือ Langflow
เขียนนิดหน่อย + สวยๆ      → Streamlit + LlamaIndex
```

### "อยาก custom เต็มที่ + deploy ง่าย"

```
RAG อย่างเดียว                    → LlamaIndex + FastAPI
RAG + Agent + Tools               → LangChain + LangServe
RAG + custom logic + deploy ง่าย  → เขียนเอง + LangChain ประกบ
                                     (custom retriever + LangServe)
```

### "Enterprise production"

```
ต้อง custom ทุกอย่าง     → Haystack หรือ เขียนเอง
ต้อง compliance          → AWS Bedrock / Azure AI Search
ต้อง connector เยอะ      → Danswer/Onyx
```

### Decision Flowchart

```
เขียน code ได้ไหม?
├── ไม่ได้ → เอกสารซับซ้อนไหม?
│           ├── ใช่ → RAGFlow
│           └── ไม่  → Dify
│
└── ได้ → ต้องการ UI พร้อมใช้ไหม?
         ├── ใช่ → ต้อง custom มากไหม?
         │        ├── ใช่ → Flowise/Langflow
         │        └── ไม่  → Dify
         │
         └── ไม่ → deploy ง่ายสำคัญไหม?
                  ├── ใช่ → LangChain + LangServe
                  │         (ประกบ custom logic)
                  └── ไม่  → เขียนเอง + หยิบ library
                            (ยืดหยุ่นสูงสุด)
```

---

> **สร้างจากการสนทนากับ Claude — มีนาคม 2026**
