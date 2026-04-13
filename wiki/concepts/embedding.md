---
title: "Embedding — แปลงข้อความเป็น Vector"
type: concept
tags: [embedding, vector, nlp, semantic-search, rag, text-embedding]
sources: [wiki/sources/rag-complete-knowledge, wiki/sources/rag-glossary-data-prep, wiki/sources/chunking-deepdive-complete, wiki/sources/rag-knowledge-overview]
related: [wiki/concepts/vector-database, wiki/concepts/llm-large-language-model, wiki/concepts/rag-retrieval-augmented-generation, wiki/concepts/rag-chunking-strategies, wiki/concepts/hybrid-search-bm25-vector]
created: 2026-04-14
updated: 2026-04-14
---

## สรุปสั้น

Embedding คือกระบวนการแปลงข้อความ (หรือสื่ออื่น) ให้เป็น **vector ตัวเลขหลายมิติ** ที่แสดง "ความหมาย" — ข้อความที่มีความหมายใกล้เคียงกันจะได้ vectors ที่อยู่ใกล้กันในพื้นที่ vector นั้น เป็นพื้นฐานของ semantic search และ RAG

## อธิบาย

### Embedding คืออะไร

```
"แมวกินปลา"  →  [0.12, -0.45, 0.78, 0.23, ...]  (1536 มิติ)
"The cat ate fish" → [0.11, -0.46, 0.77, 0.24, ...]  (1536 มิติ)
```

แม้สองประโยคจะคนละภาษา แต่ vectors จะ "ใกล้กัน" เพราะความหมายเหมือนกัน — นี่คือพลังของ embedding

### ทำไม Embedding ถึงสำคัญกับ RAG

```
ขั้นตอน RAG:
1. Ingest: Document → chunks → embedding → เก็บใน Vector DB
2. Query:  คำถาม → embedding → ค้นหา chunks ที่ใกล้ที่สุด → ส่งให้ LLM ตอบ
```

ถ้า embedding ไม่ดี → chunks ที่ดึงมาจะไม่ตรงกับคำถาม → LLM ตอบผิด

### Embedding Models ยอดนิยม

| Model | มิติ | Context | ราคา | เหมาะกับ |
|-------|------|---------|------|---------|
| `text-embedding-3-small` | 1536 | 8191 tokens | $0.02/1M | **แนะนำ** — ราคา/คุณภาพดีที่สุด |
| `text-embedding-3-large` | 3072 | 8191 tokens | $0.13/1M | ต้องการ accuracy สูงสุด |
| `text-embedding-ada-002` | 1536 | 8191 tokens | $0.10/1M | Legacy (เก่า) |
| `BGE-M3` | 1024 | 8192 tokens | ฟรี (local) | Multilingual รวมถึงภาษาไทย |
| `E5-large` | 1024 | 512 tokens | ฟรี (local) | Open source คุณภาพสูง |
| `nomic-embed-text` | 768 | 2048 tokens | ฟรี (local) | Context ยาว, ฟรี |

## ประเด็นสำคัญ

### Cosine Similarity — วัดความใกล้ของ Vectors

```python
import numpy as np

def cosine_similarity(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

# ตัวอย่าง
v1 = embed("แมวกินปลา")
v2 = embed("The cat ate fish")  
v3 = embed("วิธีทำโปรแกรม Python")

print(cosine_similarity(v1, v2))  # ~0.95 (ใกล้มาก)
print(cosine_similarity(v1, v3))  # ~0.20 (ไกลมาก)
```

threshold ที่ใช้บ่อย: `≥ 0.7` สำหรับ relevant, `≥ 0.9` สำหรับ semantic cache

### Chunking กับ Embedding

ขนาด chunk ส่งผลต่อคุณภาพ embedding:
- **chunk เล็กเกินไป** → context น้อย → embedding ไม่จับ "ธีม" ของเนื้อหา
- **chunk ใหญ่เกินไป** → หลายความหมายรวมกัน → vector เฉลี่ยออกมา noisy
- **แนะนำ**: 256–512 tokens ต่อ chunk พร้อม 20–50 tokens overlap

### Batch Embedding — ประหยัดค่าใช้จ่าย

```python
# แย่ — เรียก API ทีละ chunk
for chunk in chunks:
    embedding = openai.embeddings.create(input=chunk, model="text-embedding-3-small")

# ดี — batch ส่งทีเดียว
embeddings = openai.embeddings.create(
    input=chunks,  # list ได้สูงสุด 2048 items
    model="text-embedding-3-small"
)
```

### Multilingual Embedding

สำหรับเนื้อหาภาษาไทย:
- `text-embedding-3-small/large` รองรับภาษาไทยได้ดี (ผ่าน multilingual training)
- `BGE-M3` (Meta) รองรับ 100+ ภาษา รวมถึงไทย — ฟรี แต่ต้องรัน local

## ตัวอย่าง / กรณีศึกษา

**Arona RAG System:** ใช้ OpenAI `text-embedding-3-small` embed เอกสาร Elysia ทั้งหมด เก็บใน ParadeDB — เมื่อ user ถามเรื่อง API, embedding ของคำถามจะ match กับ chunk เอกสาร API ที่ใกล้กัน

**Haystack Thai RAG:** แนะนำ `intfloat/multilingual-e5-large` สำหรับภาษาไทย เนื่องจาก OpenAI embeddings อาจ noisy กับ Thai text ยาวๆ

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/vector-database|Vector Database]] — vectors จาก embedding จะถูกเก็บใน Vector DB
- [[wiki/concepts/llm-large-language-model|LLM]] — LLM ใช้ embeddings ภายในเพื่อเข้าใจภาษา แต่ "embedding model" เป็นคนละตัว
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — embedding เป็นขั้นตอนสำคัญทั้ง Ingest และ Retrieval phase
- [[wiki/concepts/rag-chunking-strategies|RAG Chunking]] — ต้อง chunk ก่อน แล้วค่อย embed แต่ละ chunk
- [[wiki/concepts/semantic-caching|Semantic Caching]] — ใช้ embedding similarity ≥ 90% แทน exact match สำหรับ cache

## แหล่งที่มา

- [[wiki/sources/rag-complete-knowledge|Complete RAG & Agent Knowledge Base]]
- [[wiki/sources/rag-glossary-data-prep|RAG Glossary & Data Preparation]]
- [[wiki/sources/chunking-deepdive-complete|Chunking Deep Dive]]
- [[wiki/sources/rag-knowledge-overview|RAG Agent Knowledge — Overview]]
