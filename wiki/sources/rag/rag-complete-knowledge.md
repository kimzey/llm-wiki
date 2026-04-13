---
title: "Complete RAG & Agent Knowledge Base"
type: source
source_file: raw/notes/rag-knowledge/01-rag-agent-complete-knowledge.md
tags: [rag, llm, agent, vector-db, embedding, knowledge-base]
related: [wiki/concepts/rag-retrieval-augmented-generation, wiki/concepts/llm-large-language-model, wiki/concepts/vector-database]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/rag/01-rag-agent-complete-knowledge.md|Original file]]

## สรุป

คู่มือสอน RAG & Agent ที่ครอบคลุมทุกแง่มุม — ตั้งแต่ LLM พื้นฐาน, Prompt Engineering, Embedding, Vector Database, RAG ยุคแรก, Advanced RAG, Agent, Agentic Patterns, Multi-Agent, จนถึง Evaluation และ Production Concerns

## ประเด็นสำคัญ

### 1. LLM — สมองของระบบ
- **LLM ทำงานยังไง**: เครื่องทำนาย "คำถัดไป" จาก patterns ที่เรียนรู้มา
- **สิ่งสำคัญ**:
  - Token: หน่วยเล็กสุดที่ LLM อ่าน (ภาษาไทย 1 คำ ≈ 2-4 tokens)
  - Context Window: จำนวน tokens สูงสุดที่เห็นใน 1 ครั้ง
  - Temperature: ความสุ่มของคำตอบ (0 = ตอบเหมือนเดิม, 1.0 = สุ่มมาก)
  - System Prompt: คำสั่งที่บอก LLM ว่า "คุณคือใคร ทำอะไร"

### 2. Prompt Engineering
- **Role Assignment**: บอก LLM ว่าเป็นใคร
- **Few-shot Examples**: ให้ตัวอย่างว่าอยากได้คำตอบแบบไหน
- **Chain of Thought (CoT)**: ให้ LLM คิดทีละขั้นตอน
- **Output Format Control**: กำหนดรูปแบบ output (เช่น JSON)

### 3. Embedding — แปลงข้อความเป็นตัวเลข
- **การทำงาน**: แปลงข้อความเป็น vector (ชุดตัวเลข 768-1536 มิติ)
- **Distance Metrics**:
  - **Cosine Similarity/Distance**: วัด "มุม" ระหว่าง vectors (ใช้บ่อยที่สุด)
  - **Euclidean Distance**: วัด "ระยะทางตรง"
  - **Dot Product**: เร็วที่สุด (ถ้า vectors normalized)
- **Embedding Models ภาษาไทย**:
  - Cohere embed-multilingual-v3: ดีที่สุดสำหรับภาษาไทย
  - BGE-M3: open source ดีมาก
  - OpenAI text-embedding-3-small: ใช้ได้ดีสำหรับ use case ทั่วไป

### 4. Vector Database — คลังความรู้
- **ทำไมต้อง Vector Database**: ค้นหา "ความหมายคล้ายกัน" ไม่ใช่แค่คำตรงตัว
- **Vector Index ทำงานยังไง**:
  - **Flat**: Brute force (data < 10K rows)
  - **IVFFlat**: แบ่งเป็นกลุ่ม (lists = √n)
  - **HNSW**: กราฟหลายชั้น (แนะนำ — แม่นยำ + เร็ว)
- **Metadata Filtering**: Filter ด้วย category, department, doc_type ช่วยให้แม่นยำขึ้น

### 5. RAG — ค้นหาแล้วตอบ
- **ทำงานยังไง**:
  1. Embed Query
  2. Search ใน Vector DB → ได้ chunks ที่เกี่ยวข้อง
  3. Build Context
  4. Generate ด้วย LLM
- **RAG Prompt ที่ดี**: ตอบเฉพาะจาก context, ห้ามเดา, อ้างอิง source

### 6. Advanced RAG
- **Query Rewriting**: เขียน query ใหม่ให้ดีขึ้น
- **Hybrid Search**: รวม BM25 + Vector Search (ดีกว่า vector อย่างเดียว 15-30%)
- **Re-ranking**: ใช้ AI model ตัวที่ 2 จัดอันดับใหม่
- **Context Compression**: ย่อ context ให้กระชับ

### 7. Agent — AI ที่ตัดสินใจเอง
- **คืออะไร**: ระบบ AI ที่เลือกว่าจะทำอะไร (tool selection)
- **ทำไมต้อง Agent**: RAG มี flow ตายตัว — Agent มี decision making
- **Agent Loop**:
  1. LLM ตัดสินใจ: "ต้อง search หรือไม่?"
  2. Search → ได้ผลลัพธ์
  3. LLM ประเมิน: "พอมั้ย?"
  4. ถ้าไม่พอ → ทำซ้ำ (loop)

### 8. Agentic Patterns
- **ReAct**: Reasoning + Acting (คิด → ทำ → observe → คิดใหม่)
- **Plan-and-Execute**: วางแผนก่อน →  execute ทีละขั้น
- **Self-Reflection**: LLM ตรวจสอบผลลัพธ์ของตัวเอง

### 9. Multi-Agent
- หลาย Agent ทำงานร่วมกัน
- แต่ละ Agent มีความเชี่ยวชาญเฉพาะทาง
- มี Coordinator Agent จัดการ workflow

### 10. Evaluation — วัดผลระบบ
- **Retrieval Metrics**:
  - Context Relevance: chunks ที่ดึงมาเกี่ยวข้องมั้ย
  - MRR (Mean Reciprocal Rank): อันดับแรกมีความสำคัญมาก
- **Generation Metrics**:
  - Faithfulness: ตอบตาม context มั้ย
  - Relevance: ตอบตรงประเด็นมั้ย
- **End-to-end Metrics**:
  - Exact Match: ตอบตรงตัว
  - SAS (Semantic Answer Similarity): ความหมายใกล้เคียง

### 11. Production Concerns
- **Cost Optimization**: Semantic cache, batch embedding, incremental update
- **Latency**: Co-location (app + DB อยู่ใกล้กัน), vector index
- **Scalability**: Horizontal scale, partition, cache
- **Reliability**: Rate limiting, retry, fallback

## ข้อมูลน่าสนใจ

### Temperature Settings
```
temperature = 0:   ตอบเหมือนเดิมทุกครั้ง (เหมาะ: RAG, fact-based Q&A)
temperature = 0.7: สุ่มปานกลาง (เหมาะ: เขียนบทความ)
temperature = 1.0: สุ่มมาก (เหมาะ: brainstorming)
```

### Vector Index Selection
```
Data < 10K     → ไม่ต้อง index (flat ก็เร็วพอ)
Data 10K-100K  → HNSW (แนะนำ) หรือ IVFFlat
Data > 100K    → HNSW (m=32, ef=128)
Data > 1M      → partition + HNSW
```

### Hybrid Search Benefit
- ผลวิจัย 2024-2025: Hybrid search ดีกว่า vector อย่างเดียว **15-30%** ในเกือบทุก benchmark

### RAG System Prompt Template
```python
RAG_SYSTEM_PROMPT = """คุณเป็น AI Assistant ของบริษัท

## กฎการตอบ:
1. ตอบ **เฉพาะ** จากข้อมูลใน [Context] เท่านั้น
2. ถ้าไม่มีข้อมูลเพียงพอ → บอกตรงๆ ว่า "ไม่พบข้อมูล"
3. ห้ามเดา ห้ามสร้างข้อมูลเอง
4. อ้างอิงแหล่งที่มาเสมอ
"""
```

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG — Retrieval-Augmented Generation]]
- [[wiki/concepts/llm-large-language-model|LLM — Large Language Model]]
- [[wiki/concepts/vector-database|Vector Database]]
- [[wiki/concepts/embedding|Embedding]]
- [[wiki/concepts/ai-agent|AI Agent]]
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search]]
- [[wiki/concepts/semantic-caching|Semantic Caching]]
