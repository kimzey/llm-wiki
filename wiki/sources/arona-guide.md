---
title: "Arona - คู่มือการใช้งานระบบ"
type: source
source_file: raw/notes/arona/ARONA_GUIDE.md
tags: [arona, rag, elysia, documentation-search]
related: []
created: 2026-04-13
updated: 2026-04-13
---

## สรุป

คู่มือการใช้งานระบบ Arona อย่างครอบคลุม - ระบบค้นหาเอกสารด้วย AI โดยใช้เทคนิค RAG (Retrieval-Augmented Generation) สำหรับเอกสารของ Elysia Framework ออกแบบมาเพื่อเป็นทางเลือกที่ราคาย่อมเมื่อเทียบกับ SaaS providers อย่าง Kapa, Mendable (ที่ราคา $1,000/เดือน)

## ประเด็นสำคัญ

### ภาพรวมระบบ
- **Arona** เป็นระบบ RAG แบบ self-hosted สำหรับค้นหาเอกสาร
- ใช้ Elysia Framework (Bun/TypeScript)
- ต้นทุนเฉลี่ย ~$21-27/เดือน (สำหรับ 750 requests/day)
- รองรับค้นหาด้วยภาษาธรรมชาติ
- อ้างอิงแหล่งข้อมูลอัตโนมัติ
- มี Semantic Caching เพื่อประสิทธิภาพ
- ป้องกันการใช้งานโดยบอท

### สถาปัตยกรรม
```
Client → Security Layer (PoW + Turnstile + Rate Limit)
       → Cache Layer (LRU + Semantic + Redis)
       → AI Engine (GPT-OSS-120B + Tools)
       → Search Layer (BM25 + Vector + Hybrid)
       → Data Layer (ParadeDB + DragonflyDB + OpenRouter)
```

### Components หลัก
- **Backend**: Elysia Framework
- **Database**: ParadeDB (PostgreSQL + BM25 + Vector Search)
- **Cache**: DragonflyDB (Redis-compatible)
- **AI Models**: GPT-OSS-120B (main), GPT-OSS-20B (query normalization)
- **Embeddings**: OpenAI text-embedding-3-small (1536 dimensions)

### Features สำคัญ

#### 1. Hybrid Search
- BM25 Full-Text Search (ParadeDB)
- Vector Semantic Search (pgvector)
- Hybrid Scoring (0.775 × BM25 + 0.275 × weight)
- Parent Document Retrieval

#### 2. Multi-Layer Caching
- In-Memory LRU (30s - 9hr TTL)
- Semantic Cache (Redis Vector Search, 90% similarity threshold)
- Redis Cache (3hr - 10hr TTL)
- Embedding Cache (750 items, ~4.4MB)

#### 3. Security Layers
- Proof of Work (19 bits SHA256)
- Cloudflare Turnstile
- Rate Limiting (10 req / 35 sec)
- Checksum Verification

#### 4. AI Tool System
- `search`: ค้นหาเอกสารด้วยคำสำคัญ
- `readPage`: อ่านเอกสารทั้งหน้า
- `tableOfContents`: ดูเนื้อหาทั้งหมด
- `readHistory`: อ่านประวัติการสนทนา

#### 5. Document Indexing
- Header-based chunking (แบ่งตาม ## headers)
- Triple embeddings per chunk (content, title, filename)
- AI-generated summaries (SKILLS.md style)
- Weight system (essential=1.0, blog=0.8, patterns=0.5, etc.)
- Incremental updates (อัปเดตเฉพาะที่เปลี่ยน)

### API Endpoints หลัก

#### POST /model/ask
คำถามเกี่ยวกับ Elysia Framework

**Request:**
```typescript
{
  message: string,        // คำถาม (max 4096 chars)
  seed?: number,          // Random seed
  reference?: string,     // Specific page
  think?: boolean,        // Deeper reasoning
  history?: Array<{       // Conversation history
    role: 'user' | 'assistant',
    content: string,
    checksum: string
  }>
}
```

**Response:** Streaming text + sources

#### GET /pow/request
ขอ Proof of Work challenge

**Response:**
```typescript
{
  nonce: string,      // Random challenge
  bits: 19,           // Difficulty
  expires: number     // Expiry timestamp
}
```

### Database Schema
```sql
CREATE TABLE documents (
    link VARCHAR(255) PRIMARY KEY,
    file VARCHAR(255) NOT NULL,
    title VARCHAR(255) NOT NULL,
    content TEXT NOT NULL,
    summary TEXT NOT NULL,
    weight FLOAT NOT NULL DEFAULT 0.5,
    embedding VECTOR(1536) NOT NULL,
    title_embedding VECTOR(1536) NOT NULL,
    file_name_embedding VECTOR(1536) NOT NULL,
    sequence smallint NOT NULL DEFAULT 0
);

-- BM25 Index
CREATE INDEX idx_documents_content_bm25
ON documents USING bm25 (link, title, summary, file_name);
```

### Stack ที่ใช้
- **Backend**: Elysia (Bun)
- **Database**: ParadeDB (PostgreSQL + BM25 + Vector)
- **Cache**: DragonflyDB (Redis)
- **AI**: OpenRouter (GPT-OSS models)
- **Monitoring**: OpenTelemetry + Axiom
- **Security**: Cloudflare Turnstile
- **Hosting**: Hetzner (Europe)

### ข้อดี
- ต้นทุนต่ำ (~$21/เดือน vs $250+ SaaS)
- Control ได้ทั้งหมด
- Performance ดี (ไม่มี framework overhead)
- เข้าใจระบบลึกๆ แก้ไขง่าย
- Scale ง่าย (stateless design)

### ข้อเสีย
- ใช้เวลาพัฒนานาน (3-4 สัปดาห์)
- ต้องดูแลเองทั้งหมด
- ต้องเข้าใจหลายด้าน (DB, AI, DevOps)
- ต้องมี observability ดี

## เทคนิคที่น่าสนใจ

### Semantic Caching
ใช้ Vector Similarity Search หาคำถามที่คล้ายกัน
- "ลาพักร้อนกี่วัน" ≈ "ลาพักผ่อนประจำปีกี่วัน" (90%+)
- ช่วยลด AI call ได้มาก

### Checksum Verification
ป้องกันการปลอมแปลงประวัติการสนทนา
```typescript
checksum = sha256(content + secret)
```

### Flex Mode
ยอมรับ errors และ retry ด้วย exponential backoff
- `maxRetries: 3` ใน streamText
- retry() function ในทุก external API call

## Concepts ที่เกี่ยวข้อง
- [[wiki/concepts/rag]] - Retrieval-Augmented Generation
- [[wiki/concepts/hybrid-search]] - BM25 + Vector Search
- [[wiki/concepts/semantic-caching]] - Semantic Similarity Cache
- [[wiki/concepts/stateless-rag]] - Stateless Architecture
