---
title: "Haystack Deep Dive — Phase 3: Embedding, Retrievers & Query Pipeline"
type: source
source_file: raw/notes/Hetstack/haystack-phase3-embedding-retrieval.md
url: ""
published: 2026-04-01
tags: [haystack, embedding, retrieval, hybrid-search, bm25, semantic-search, ranker]
related: [wiki/sources/haystack-phase2-indexing.md, wiki/sources/haystack-phase4-advanced.md, wiki/concepts/haystack-framework.md, wiki/concepts/hybrid-search-bm25-vector.md, wiki/concepts/rag-retrieval-augmented-generation.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/Hetstack/haystack-phase3-embedding-retrieval.md|Original file]]
> *ดูเพิ่ม: [[../../raw/notes/Hetstack/haystack-phase3-query-pipeline.md|Query Pipeline alt file]]*

## สรุป

อธิบาย Retrieval ทุกประเภทใน Haystack — BM25 (keyword), Embedding (semantic), Hybrid (BM25+Vector), Cross-encoder Ranker, Metadata Filtering, และ PromptBuilder + Generators สำหรับสร้าง complete RAG Query Pipeline

## ประเด็นสำคัญ

- **BM25Retriever**: ค้นหาด้วย keyword matching ไม่ต้องการ embedding, เร็ว
- **EmbeddingRetriever**: ค้นหาด้วย cosine similarity, ต้องการ vector, เข้าใจความหมาย
- **Hybrid Search = DocumentJoiner(join_mode="reciprocal_rank_fusion")**: RRF คือ mode ที่แนะนำที่สุด
- **Cross-encoder Ranker vs Bi-encoder**: Bi-encoder (embedder) เร็วแต่อาจไม่แม่น → ใช้ retrieve; Cross-encoder (ranker) ช้ากว่าแต่แม่นมาก → ใช้หลัง retrieve
- **PromptBuilder ใช้ Jinja2**: `{% for doc in documents %}{{ doc.content }}{% endfor %}` + conditional logic
- **Generator types**: OpenAIGenerator (text), OpenAIChatGenerator (multi-turn + tools), HuggingFaceLocalGenerator, OllamaGenerator

## ข้อมูล / หลักฐาน ที่น่าสนใจ

**Metadata Filter syntax:**
```python
{
    "operator": "AND",
    "conditions": [
        {"field": "meta.year", "operator": ">=", "value": 2024},
        {"field": "meta.category", "operator": "in", "value": ["finance", "hr"]}
    ]
}
```

**Query Pipeline flow:**
```
Query → TextEmbedder → EmbeddingRetriever → Ranker → PromptBuilder → Generator → Answer
```
หรือ Hybrid:
```
Query ─┬→ BM25Retriever ──┬→ DocumentJoiner(RRF) → PromptBuilder → Generator
       └→ TextEmbedder → EmbeddingRetriever ─┘
```

**Ranker model**: `cross-encoder/ms-marco-MiniLM-L-6-v2` — เหมาะสำหรับ production

**DocumentJoiner join_mode:**
| mode | คำอธิบาย |
|---|---|
| `concatenate` | รวมทั้งหมด ไม่ปรับ score |
| `merge` | เฉลี่ย score |
| `reciprocal_rank_fusion` | **แนะนำ** — ใช้ลำดับ ranking แทน score |

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่มีข้อขัดแย้ง — เพิ่ม implementation details ของ Hybrid Search ที่มีอยู่แล้วใน wiki

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/haystack-framework|Haystack Framework]]
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search — BM25 + Vector Search]]
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG — Retrieval-Augmented Generation]]
