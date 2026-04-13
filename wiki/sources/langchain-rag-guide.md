---
title: "LangChain, RAG & Sellsuki Knowledge Base — Deep Dive"
type: source
source_file: raw/notes/tool-rag/langchain-rag-guide.md
url: ""
published: 2026-01-01
tags: [langchain, langgraph, langsmith, langserve, rag, sellsuki, pipeline]
related: [wiki/concepts/langchain-framework.md, wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/rag-chunking-strategies.md, wiki/concepts/semantic-caching.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/tool-rag/langchain-rag-guide.md|Original file]]

## สรุป

Deep Dive ครบของ LangChain Ecosystem ทั้งหมด (LangChain, LangGraph, LangSmith, LangServe, Hub), Core Concepts (Document, LCEL, Agent vs Chain), RAG Pipeline โดยละเอียด, Chunking Strategies, Metadata, Embedding, Semantic Cache และ Full Architecture สำหรับ Sellsuki RAG System

## ประเด็นสำคัญ

### LangChain Ecosystem ทั้ง 5 ตัว

| Framework | หน้าที่ | Output |
|-----------|--------|--------|
| **LangChain** | Core building blocks | Chains, Agents, Retrievers |
| **LangGraph** | Complex workflows (stateful DAG) | Graph execution with loops/conditions |
| **LangSmith** | Observability — Debug, Trace, Evaluate | Traces, Latency, Token usage dashboard |
| **LangServe** | Deploy chains เป็น REST API อัตโนมัติ | FastAPI endpoints `/invoke`, `/stream`, `/batch` |
| **LangChain Hub** | Prompt repository | Reusable tested prompts |

**LangGraph** เหมาะสำหรับ RAG ที่มี fallback logic:
```python
workflow.add_conditional_edges(
    "grade_docs",
    decide_next_step,
    {"relevant": "generate", "not_relevant": "web_search"}
)
```

**LangSmith** — enable ด้วยแค่ 2 env vars:
```python
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "ls_..."
```

### RAG Pipeline (Phase 1: Indexing + Phase 2: Querying)

```
Phase 1: INDEXING
Documents → Loader → Text Splitter + Metadata → Embedding → Vector Store (ParadeDB)
                                                            + Hash Store (Dragonfly) สำหรับ dedup

Phase 2: QUERYING
User Question → Semantic Cache Check (Dragonfly)
              ↓ cache miss
              Embed Question → Vector Search (ParadeDB) + Metadata Filter
              → Top-K Chunks → Reranking → Prompt Construction
              → LLM → Answer + Sources → Save to Cache
```

### Chunking Strategies

| Strategy | คำอธิบาย | เหมาะกับ |
|---------|----------|---------|
| Fixed-Size | `RecursiveCharacterTextSplitter(chunk_size=500, overlap=50)` | เอกสารทั่วไป, เริ่มต้น |
| Semantic Chunking | `SemanticChunker(OpenAIEmbeddings())` — ตัดเมื่อความหมายเปลี่ยน | HR Policy, Technical Docs |
| Markdown Header | `MarkdownHeaderTextSplitter` — แบ่งตาม `#`, `##`, `###` | Markdown docs, เก็บ hierarchy |
| Token-based | `TokenTextSplitter` — แบ่งตาม token จริง | LLM context window critical |
| Parent-Document | ค้นหา child chunks เล็ก, ส่ง parent ใหญ่ให้ LLM | เพิ่ม context ลด precision loss |

### Metadata สำหรับ RAG

```python
doc = Document(
    page_content="...",
    metadata={
        "source": "hr_policy.pdf",
        "page": 3,
        "department": "HR",
        "last_updated": "2024-01-15",
        "access_level": "all_staff"
    }
)
```

### Sellsuki RAG Full Architecture

```
LINE/Slack/Web → API Gateway (FastAPI)
              → LangChain/LlamaIndex Query Engine
              → Semantic Cache (Dragonfly) → HIT: return
              → Vector Search (ParadeDB: pgvector + BM25 Hybrid)
              → Reranker → LLM (GPT-4o) → Answer + Sources
```

**Semantic Cache ด้วย LangChain:**
```python
langchain.llm_cache = RedisSemanticCache(
    redis_url="redis://dragonfly:6379",
    embedding=OpenAIEmbeddings(),
    score_threshold=0.95
)
```

### LangServe — Deploy เป็น API ง่ายมาก

```python
from langserve import add_routes
app = FastAPI()
add_routes(app, sellsuki_rag_chain, path="/sellsuki-rag")
# ได้: POST /sellsuki-rag/invoke, /stream + GET /playground
```

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- `PydanticOutputParser` ทำให้ได้ structured output type-safe จาก LLM
- `RunnableWithMessageHistory` ห่อ chain เพื่อเก็บ chat history ใน Redis โดยแยก session ด้วย `session_id`
- LangChain Hub prompt `rlm/rag-prompt` เป็น RAG prompt ที่ผ่านการ test มาแล้ว
- `MarkdownHeaderTextSplitter` เก็บ header เป็น metadata อัตโนมัติ: `{"h1": "HR Policy", "h2": "วันลา"}`

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/langchain-framework|LangChain Framework]]
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG — Retrieval-Augmented Generation]]
- [[wiki/concepts/rag-chunking-strategies|RAG Chunking Strategies]]
- [[wiki/concepts/semantic-caching|Semantic Caching]]
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search — BM25 + Vector]]
