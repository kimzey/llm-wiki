---
title: "RAG Glossary & Data Preparation — Complete Guide"
type: source
source_file: raw/notes/rag-knowledge/02-rag-glossary-deepdive.md
tags: [rag, glossary, data-prep, chunking, embedding, vector-db]
related: [wiki/concepts/rag-retrieval-augmented-generation, wiki/concepts/rag-chunking-strategies]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/rag/02-rag-glossary-deepdive.md|Original file]]

## สรุป

คู่มือคำศัพท์ RAG ครบถ้วนพร้อมวิธีเตรียมข้อมูลเข้า Vector Database — ครอบคลุม Core Concepts, Search & Retrieval, Chunking & Indexing, Database & Index, Caching, Anti-abuse, และ Infrastructure

## ประเด็นสำคัญ

### Part 1: Core Concepts

#### RAG (Retrieval-Augmented Generation)
- **หลักการ**: "ค้นหาก่อน แล้วค่อยตอบ"
- **ทำไมไม่ fine-tune**: Fine-tune = แพง, ช้า, update ยาก | RAG = ถูก, เร็ว, update ได้ทันที

#### LLM (Large Language Model)
- **คือ**: โมเดล AI ที่เข้าใจและสร้างภาษาได้ (เหมือน "คนฉลาดมาก" ที่อ่านหนังสองมาเป็นล้านเล่ม)
- **ตัวอย่าง**: GPT-4o, Claude, Gemini, Llama, Mistral
- **ศัพท์สำคัญ**:
  - **Token**: หน่วยเล็กสุดที่ LLM อ่าน (ภาษาไทย 1 คำ ≈ 2-4 tokens)
  - **Prompt**: ข้อความที่ส่งให้ LLM
  - **Context**: ข้อมูลทั้งหมดที่ LLM เห็นใน 1 ครั้ง
  - **Context Window**: ขนาด context สูงสุด (เช่น 128K tokens)

#### Embedding
- **คือ**: การแปลงข้อความเป็น vector (ชุดตัวเลข)
- **ตัวอย่าง**:
  - "ลาพักร้อน" → [0.82, -0.15, 0.43, ..., 0.07] (768 ตัวเลข)
  - "หยุดพักผ่อน" → [0.80, -0.14, 0.41, ..., 0.09] (คล้ายกัน)
  - "ซื้อข้าว" → [0.12, 0.67, -0.33, ..., 0.55] (ต่างกัน)

#### Vector Database
- **คือ**: ฐานข้อมูลที่เก็บ vectors และค้นหา "vector ที่ใกล้กัน" ได้เร็ว
- **ตัวอย่าง**: pgvector, ParadeDB, Pinecone, Qdrant, ChromaDB, Milvus, Weaviate, LanceDB

### Part 2: Search & Retrieval

#### Cosine Similarity
- **คือ**: วัดว่า 2 vectors "ชี้ไปทิศทางเดียวกัน" แค่ไหน
- **ค่า**: 1.0 = เหมือนกันเป๊ะ | 0.0 = ไม่เกี่ยวกันเลย | -1.0 = ตรงข้ามกัน
- **ตัวอย่าง**:
  - "ลาพักร้อน" vs "หยุดพักผ่อน" = 0.95 (คล้ายมาก)
  - "ลาพักร้อน" vs "ซื้อข้าว" = 0.15 (ไม่เกี่ยว)

#### BM25 (Best Match 25)
- **คือ**: อัลกอริทึมจัดอันดับข้อความ (full-text search)
- **หลักการ**:
  1. **Term Frequency (TF)**: คำนั้นปรากฏในเอกสารกี่ครั้ง
  2. **Inverse Document Frequency (IDF)**: คำนั้นหายากแค่ไหน
  3. **Document Length**: เอกสารสั้นที่มีคำตรง → น่าจะเกี่ยวข้องมากกว่า
- **เปรียบเทียบ**:
  - BM25: แม่นชื่อเฉพาะ, คำ technical
  - Vector Search: เข้าใจ synonyms, paraphrase
  - **Hybrid**: ได้ข้อดีทั้งสอง → แม่นที่สุด

#### Hybrid Search
- **คือ**: รวม Vector Search + BM25 เข้าด้วยกัน (RRF - Reciprocal Rank Fusion)
- **ผลลัพธ์**: ดีกว่า vector อย่างเดียว **15-30%** ในเกือบทุก benchmark

#### Re-ranking
- **คือ**: ใช้ AI model ตัวที่ 2 มาจัดอันดับผลลัพธ์ search ใหม่
- **Flow**: Search ได้ 20 ผลลัพธ์ → ส่งให้ Re-ranker → เอา top 5
- **Models**: Cohere Rerank, bge-reranker-v2-m3, Jina Reranker
- **ใช้เมื่อ**: เอกสารเยอะมาก (> 1000 pages), ต้องการความแม่นยำสูงสุด

### Part 3: Chunking & Indexing

#### Chunk / Chunking
- **คือ**: ท่อนข้อความเล็กๆ ที่ถูกตัดมาจากเอกสารใหญ่
- **ทำไมต้อง chunk**:
  1. Embedding model มี limit (8192 tokens)
  2. ข้อความสั้น → embedding แม่นกว่า
  3. ส่งแค่ chunk ที่เกี่ยวข้องให้ LLM → ประหยัด tokens

#### Chunk Overlap
- **คือ**: ส่วนที่ซ้อนกันระหว่าง chunk ต่อเนื่อง
- **ทำไมต้อง overlap**: ถ้าไม่ overlap → ข้อมูลอาจหายตรง "รอยตัด"
- **ค่าแนะนำ**:
  - chunk_size = 800 ตัวอักษร
  - overlap = 150-200 ตัวอักษร (15-25% ของ chunk_size)

#### Content Hash
- **คือ**: ค่า fingerprint ของข้อความ (SHA256)
- **ใช้สำหรับ**: Diff-based Indexing (skip embed ถ้า content ไม่เปลี่ยน)

### Part 4: Database & Index

#### HNSW (Hierarchical Navigable Small World)
- **คือ**: อัลกอริทึมสร้าง index สำหรับ vector search
- **หลักการ**: สร้าง "กราฟหลายชั้น" เพื่อค้นหาเร็วขึ้น
- **Parameters**:
  - **m**: จำนวน connections ต่อ node (default 16) — ยิ่งมาก = แม่นกว่า แต่ใช้ memory มากกว่า
  - **ef_construction**: ความกว้างตอนสร้าง index (default 64) — ยิ่งมาก = index ดีกว่า แต่สร้างนานกว่า
  - **ef_search**: ความกว้างตอนค้นหา (default 40) — ยิ่งมาก = แม่นกว่า แต่ช้ากว่า

#### IVFFlat (Inverted File Flat)
- **คือ**: อัลกอริทึม index แบบแบ่งเป็นกลุ่ม
- **หลักการ**: แบ่งพื้นที่เป็นกลุ่ม (clusters/lists) → ค้นเฉพาะกลุ่มที่ใกล้
- **Parameters**:
  - **lists**: จำนวนกลุ่ม (แนะนำ √จำนวน rows)
  - **probes**: กี่กลุ่มที่จะค้น (ยิ่งมาก = แม่นกว่า แต่ช้ากว่า)

**HNSW vs IVFFlat**:
- HNSW: แม่นกว่า, ไม่ต้อง train, เพิ่ม data ได้เรื่อยๆ ← **แนะนำ**
- IVFFlat: เร็วกว่าเล็กน้อย, ใช้ memory น้อยกว่า, ต้อง rebuild เมื่อ data เปลี่ยนเยอะ

### Part 5: Caching

#### Semantic Cache
- **คือ**: cache ที่ดูความหมาย ไม่ใช่แค่ข้อความตรงตัว
- **Cache ปกติ**: "ลาพักร้อนกี่วัน" → hit | "ลาพักร้อนได้กี่วัน" → miss (ต่างกัน!)
- **Semantic Cache**: Cosine similarity > threshold (0.95) → hit
- **ประหยัด**: 40-60% ของ LLM calls สำหรับคำถามซ้ำๆ

#### TTFT (Time to First Token)
- **คือ**: เวลาตั้งแต่ส่ง request จนได้ token แรกกลับมา
- **ตัวอย่าง**:
  - GPT-4o: ~300-500ms
  - Groq: ~50-150ms (เร็วกว่า 3-5x)
  - Cerebras: ~30-100ms (เร็วที่สุด)

#### Flex Mode
- **คือ**: โหมดที่ API request มีโอกาสล้มเหลวได้ แลกกับราคาถูกลง ~50%
- **ใช้เมื่อ**: Non-realtime, มี retry mechanism

### Part 6: Anti-abuse

#### PoW (Proof of Work)
- **คือ**: ให้ client "ทำงาน" (คำนวณ) ก่อนจึงจะส่ง request ได้
- **ทำไมป้อง abuse**: Bot ต้องคำนวณทุก request → ส่ง spam ได้ช้ามาก

#### Turnstile (Cloudflare)
- **คือ**: CAPTCHA ที่ไม่ต้องให้ user ทำอะไร
- **หลักการ**: วิเคราะห์ behavior อัตโนมัติ (mouse movement, timing)

### Part 7: Infrastructure

#### Co-location
- **คือ**: วาง services ทั้งหมดไว้ที่เดียวกัน/ใกล้กัน
- **ปัญหา (services กระจาย)**: App (US) → DB (EU) → Cache (Asia) → LLM (US) = 550ms 😱
- **Co-location**: App + DB + Cache อยู่ server เดียวกัน = ~0.3ms + LLM (50ms) = **~50ms** 🚀

#### Coolify
- **คือ**: self-hosted PaaS (Platform as a Service)
- **ทำให้**: Deploy ด้วย git push, จัดการ Docker containers

## ข้อมูลน่าสนใจ

### Hybrid Search RRF Formula
```python
score(doc) = Σ 1 / (k + rank_i)

# k=60 (มาตรฐาน)
# BM25: [doc_A(1), doc_C(2), doc_B(3)]
# Vector: [doc_B(1), doc_A(2), doc_D(3)]

doc_A: 1/(60+1) + 1/(60+2) = 0.0325 ← ชนะ
doc_B: 1/(60+3) + 1/(60+1) = 0.0322
doc_C: 1/(60+2) + 0 = 0.0161
```

### Index Selection Guide
```
Data < 10K     → ไม่ต้อง index
Data 10K-100K  → HNSW (แนะนำ)
Data > 100K    → HNSW (m=32, ef=128)
Data > 1M      → partition + HNSW
```

### Embedding Quality (Thai)
```
Cohere embed-multilingual-v3: ดีที่สุดสำหรับภาษาไทย
BGE-M3: open source ดีมาก
OpenAI text-embedding-3-small: ใช้ได้ดีสำหรับ use case ทั่วไป
```

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG — Retrieval-Augmented Generation]]
- [[wiki/concepts/rag-chunking-strategies|RAG Chunking Strategies]]
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search — BM25 + Vector]]
- [[wiki/concepts/semantic-caching|Semantic Caching]]
- [[wiki/concepts/vector-database|Vector Database]]
