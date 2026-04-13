---
title: "Haystack Deep Dive — Phase 5: Custom Components, Production & Integrations"
type: source
source_file: raw/notes/Hetstack/haystack-phase5-custom-production.md
url: ""
published: 2026-04-01
tags: [haystack, production, fastapi, docker, evaluation, ragas, integrations, ollama, bedrock]
related: [wiki/sources/haystack-phase4-advanced.md, wiki/sources/haystack-phase6-observability.md, wiki/concepts/haystack-framework.md, wiki/concepts/rag-evaluation.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/Hetstack/haystack-phase5-custom-production.md|Original file]]
> *ดูเพิ่ม: [[../../raw/notes/Hetstack/haystack-phase5-integrations.md|Integrations alt file]]*

## สรุป

Phase 5 ครอบคลุม production deployment patterns — FastAPI integration, Docker Compose stack (Qdrant+App), RAG Evaluation metrics (Faithfulness, Recall, SAS, RAGAS), Caching pattern, Async pipeline, Integrations กับ Ollama/AWS Bedrock/Google Gemini/pgvector, และ Component selection guide

## ประเด็นสำคัญ

- **FastAPI Integration**: โหลด pipeline ครั้งเดียวตอน startup event, `pipeline.warm_up()`, serve ผ่าน `POST /query`
- **Evaluation metrics สำคัญ**: Faithfulness (คำตอบ faithful กับ context ไหม), Context Relevance, Answer Relevancy, DocumentMRR, RAGAS
- **RAGAS**: open-source evaluation framework ที่รวม faithfulness + answer_relevancy + context_recall
- **Caching**: `SerializedDocumentCacheChecker` ป้องกัน re-embed document ซ้ำ — ประหยัด cost มาก
- **Ollama**: local LLM ฟรี — `llama3.2`, `qwen2.5` (ดีสำหรับภาษาไทย), `mistral`
- **pgvector**: ใช้ PostgreSQL เป็น Vector Store ได้ — ดีถ้ามี Postgres อยู่แล้ว
- **Unstructured.io**: แปลงไฟล์ทุกประเภทรวม OCR (PPTX, XLS, images)

## ข้อมูล / หลักฐาน ที่น่าสนใจ

**Production Checklist:**
```
Retrieval:  chunk size เหมาะสม, overlap, Hybrid Search, re-ranking, metadata
Generation: prompt ชัดเจน, fallback เมื่อไม่พบ, cite source
Perf:       cache embedding, batch processing, async, scalable DocumentStore
Monitor:    log query/response, track latency, token usage, periodic eval
```

**Self-querying pattern** (LLM แยก query + filter):
```python
# ให้ LLM แยก "รายงาน Q3 2024 ของ Alice" → search_query + filters
QUERY_ANALYSIS_TEMPLATE = "แยก query เป็น JSON: search_query + filters"
# response: {"search_query": "รายงาน", "filters": {"year": 2024, "author": "Alice"}}
```

**LLM selection guide:**
```
ฟรี/Local: Ollama (llama, qwen2.5 สำหรับไทย)
ถูก:       GPT-4o-mini / Claude Haiku
ดี:        OpenAI GPT-4o / Claude Sonnet
Enterprise: AWS Bedrock / Google Vertex AI
```

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่มีข้อขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/haystack-framework.md|Haystack Framework]]
- [[wiki/concepts/rag-evaluation.md|RAG Evaluation]]
