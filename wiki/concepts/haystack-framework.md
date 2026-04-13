---
title: "Haystack Framework"
type: concept
tags: [haystack, rag, pipeline, nlp, framework, python, deepset]
sources: [haystack-phase1-intro.md, haystack-phase1-introduction.md, haystack-phase2-document-processing.md, haystack-phase2-indexing.md, haystack-phase3-embedding-retrieval.md, haystack-phase3-query-pipeline.md, haystack-phase4-advanced.md, haystack-phase4-rag-advanced.md, haystack-phase5-custom-production.md, haystack-phase5-integrations.md, haystack-phase6-api-auth-observability.md]
related: [wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/hybrid-search-bm25-vector.md, wiki/concepts/rag-evaluation.md, wiki/concepts/opentelemetry.md]
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

**Haystack** คือ Open-Source Python Framework (by deepset) สำหรับสร้าง Production-ready AI Search & NLP Pipelines — โดดเด่นด้วยสถาปัตยกรรม **Component-Pipeline-DocumentStore** ที่ type-safe, validate schema, และ serialize เป็น YAML ได้ รองรับ RAG, QA, Semantic Search, Agent Systems, และ Conversational AI

## อธิบาย

### Core Architecture: 3 สิ่งหลัก

**1. Component** — หน่วยเล็กสุด รับ input → ประมวลผล → ส่ง output
```python
@component
class MyComponent:
    @component.output_types(result=str)
    def run(self, input: str) -> dict:
        return {"result": input.upper()}
```

**2. Pipeline** — DAG ของ Components ที่เชื่อมกัน
```python
pipeline = Pipeline()
pipeline.add_component("retriever", InMemoryBM25Retriever(store))
pipeline.add_component("llm", OpenAIGenerator(model="gpt-4o-mini"))
pipeline.connect("retriever.documents", "llm.prompt")
```

**3. DocumentStore** — ฐานข้อมูลเก็บ Document + Vector
| Store | เหมาะกับ |
|---|---|
| InMemoryDocumentStore | Dev/Testing |
| QdrantDocumentStore | Production vector search |
| ElasticsearchDocumentStore | Full-text + vector (enterprise) |
| PgvectorDocumentStore | PostgreSQL users |

### 2 Pipeline หลัก

**Indexing Pipeline** (เตรียมข้อมูล):
```
Files → Converter → Cleaner → Splitter → [Embedder] → Writer → DocumentStore
```

**Query Pipeline** (ตอบคำถาม):
```
Query → [TextEmbedder] → Retriever → [Ranker] → PromptBuilder → Generator → Answer
```

### Document Object
```python
doc = Document(
    content="text",
    meta={"source": "report.pdf", "year": 2024},
    embedding=[0.1, 0.2, ...],  # หลัง embed
    score=0.95                   # หลัง retrieval
)
```

### Retrieval Strategies

| Strategy | Component | เมื่อใช้ |
|---|---|---|
| Keyword | InMemoryBM25Retriever | ต้องการคำตรง, เร็ว |
| Semantic | InMemoryEmbeddingRetriever | เข้าใจความหมาย |
| Hybrid | BM25 + Embedding + DocumentJoiner(RRF) | **แนะนำ** สำหรับ production |

## ประเด็นสำคัญ

- **Type-safe Pipeline**: input/output types ถูก validate — หา bug ได้ง่ายกว่า LangChain
- **Component เป็น building block**: Custom Component ด้วย `@component` decorator, สร้างได้ง่าย
- **Serialization**: `pipeline.dump(f)` / `Pipeline.load(f)` เป็น YAML — deploy ง่าย
- **Async support**: `pipeline.run_async()` สำหรับ high-throughput
- **warm_up()**: โหลด ML models ครั้งเดียว ป้องกัน cold start ใน production
- **Haystack 2.x เท่านั้น** (`pip install haystack-ai`) — 1.x เป็น maintenance mode แล้ว

## ตัวอย่าง / กรณีศึกษา

**Complete RAG Pipeline (Hybrid Search):**
```python
pipeline = Pipeline()
pipeline.add_component("text_embedder", SentenceTransformersTextEmbedder(model="BAAI/bge-m3"))
pipeline.add_component("bm25_retriever", InMemoryBM25Retriever(store, top_k=10))
pipeline.add_component("embedding_retriever", InMemoryEmbeddingRetriever(store, top_k=10))
pipeline.add_component("joiner", DocumentJoiner(join_mode="reciprocal_rank_fusion", top_k=5))
pipeline.add_component("ranker", TransformersSimilarityRanker(top_k=3))
pipeline.add_component("prompt_builder", PromptBuilder(template=RAG_TEMPLATE))
pipeline.add_component("llm", OpenAIGenerator(model="gpt-4o-mini"))
```

**Chunk sizing guidelines:**
```
เอกสารทั่วไป:   split_length=200-300 words, overlap=20-30
FAQ/Q&A:        split_length=100-150 words
Legal docs:     split_length=300-500 words
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — Haystack คือ framework หลักสำหรับ implement RAG
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search]] — Haystack implement ด้วย DocumentJoiner(RRF)
- [[wiki/concepts/rag-evaluation|RAG Evaluation]] — built-in evaluators + RAGAS integration
- [[wiki/concepts/opentelemetry|OpenTelemetry]] — Haystack รองรับ OTel tracing built-in

## แหล่งที่มา

- [[wiki/sources/haystack-phase1-overview|Haystack Phase 1 — Overview & Core Concepts]]
- [[wiki/sources/haystack-phase2-indexing|Haystack Phase 2 — Document Processing & Indexing]]
- [[wiki/sources/haystack-phase3-retrieval|Haystack Phase 3 — Embedding & Retrieval]]
- [[wiki/sources/haystack-phase4-advanced|Haystack Phase 4 — Advanced Features]]
- [[wiki/sources/haystack-phase5-production|Haystack Phase 5 — Production & Integrations]]
- [[wiki/sources/haystack-phase6-observability|Haystack Phase 6 — API, Auth & Observability]]
