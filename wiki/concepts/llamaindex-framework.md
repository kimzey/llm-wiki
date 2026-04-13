---
title: "LlamaIndex Framework"
type: concept
tags: [llamaindex, rag, llm, ingestion-pipeline, python, typescript, llamaparse]
sources: [llamaindex-deep-dive.md, llamaindex-full-guide.md, langchain-llamaindex-deep-dive.md, rag-decision-guide.md]
related: [wiki/concepts/langchain-framework.md, wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/rag-chunking-strategies.md, wiki/concepts/hybrid-search-bm25-vector.md, wiki/concepts/semantic-caching.md]
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

LlamaIndex คือ Data Framework สำหรับ LLM Applications โดยเฉพาะ RAG — "Specialized Search Engine Builder" ที่เชี่ยวชาญเรื่อง Index, Query, Retrieval ออกแบบมาเพื่อเชื่อม LLM เข้ากับข้อมูลของคุณให้มีประสิทธิภาพ มี IngestionPipeline + Cache built-in ที่ดีที่สุดในตลาด

## อธิบาย

LlamaIndex แก้ปัญหาที่เอกสารมีหลายรูปแบบ, ข้อมูลเยอะมากทำยังไงให้ค้นหาได้แม่นยำ, เอกสารอัพเดตบ่อยทำยังไงไม่ให้ต้อง re-index ทั้งหมด, ต้องการ filter ตาม permission ทำยังไง — ด้วยการมองเป็น 2 ขั้นตอน: **INGESTION** (Documents → Nodes → Embeddings → Index) และ **QUERYING** (Question → Retrieve Nodes → Synthesize → Answer)

### LlamaIndex Ecosystem

| Component | คือ | ใช้เมื่อ |
|-----------|-----|---------|
| **llama-index-core** | หัวใจ: nodes, indexes, engines | ทุก use case |
| **LlamaHub** | 300+ community connectors | ดึงข้อมูลจาก Notion, Confluence, Google Drive, Slack |
| **LlamaParse** | Parse เอกสารซับซ้อน (PDF ที่มีตาราง, รูป) | PDF ยาก, ดีกว่า PyPDF2/pdfplumber มาก, รองรับภาษาไทย |
| **LlamaTrace / Arize Phoenix** | Observability — retrieval quality, latency | Production monitoring |
| **LlamaAgents / Workflows** | Multi-step AI workflows, multi-agent | Router Agent เลือก index ที่ถูกต้อง |

## ประเด็นสำคัญ

### Core Building Blocks

**Document:**
```python
doc = Document(
    text="พนักงาน Sellsuki มีสิทธิ์ลาพักร้อน 10 วันต่อปี",
    metadata={"source": "hr_policy.md", "department": "hr", "access_level": "all"},
    id_="unique-doc-id",  # สำคัญ! ใช้ update/delete
    excluded_llm_metadata_keys=["internal_id"],  # metadata ไม่ส่งให้ LLM
)
```

**Node (TextNode):** chunk ที่แตกจาก Document
- มี relationships: `PREVIOUS`, `NEXT`, `PARENT`, `CHILD`, `SOURCE`
- สำคัญสำหรับ hierarchical retrieval

**NodeParsers (Chunking Strategies):**
```python
from llama_index.core.node_parser import (
    SentenceSplitter,              # แบ่งตาม sentence boundary
    SemanticSplitterNodeParser,    # แบ่งตาม semantic similarity
    HierarchicalNodeParser,        # สร้าง parent-child hierarchy
    MarkdownNodeParser,            # เข้าใจ Markdown structure
    SentenceWindowNodeParser,      # เก็บ context รอบข้าง
)

# Hierarchical — แนะนำสำหรับ Sellsuki
parser = HierarchicalNodeParser.from_defaults(chunk_sizes=[2048, 512, 128])
# 2048 = parent (context ใหญ่ ส่งให้ LLM)
# 512  = middle
# 128  = leaf (precise match)
```

**IngestionPipeline** — หัวใจของ LlamaIndex:
```python
pipeline = IngestionPipeline(
    transformations=[
        SentenceSplitter(chunk_size=512, chunk_overlap=64),
        TitleExtractor(nodes=5),
        QuestionsAnsweredExtractor(questions=3, llm=llm),  # LLM generate Q&A อัตโนมัติ
        KeywordExtractor(keywords=5),
        embed_model,
    ],
    vector_store=vector_store,
    cache=cache,  # ← RedisKVStore บน Dragonfly
)
```

**IngestionCache + RedisKVStore:**
```python
# Dragonfly (Redis-compatible) สำหรับ Ingestion Cache
cache = IngestionCache(
    cache=RedisKVStore.from_host_and_port("dragonfly", 6379),
    collection="ingestion_cache"
)
# ← ถ้า chunk content เหมือนเดิม → ข้าม embedding → ประหยัด cost
```

**VectorStoreIndex:**
```python
index = VectorStoreIndex.from_vector_store(vector_store)
query_engine = index.as_query_engine(
    similarity_top_k=10,
    node_postprocessors=[reranker],  # rerank เหลือ 3
    response_mode="tree_summarize"
)
```

**QueryEngine with Metadata Filter:**
```python
from llama_index.core.vector_stores import MetadataFilter, MetadataFilters

filters = MetadataFilters(filters=[
    MetadataFilter(key="access_level", value="all"),
    MetadataFilter(key="department", value="hr"),
])

response = await query_engine.aquery("ลาพักร้อนได้กี่วัน?", filters=filters)
```

**AutoMergingRetriever:**
```python
base_retriever = index.as_retriever(similarity_top_k=6)
retriever = AutoMergingRetriever(base_retriever, storage_context, verbose=True)
# ถ้าดึง leaf nodes มาหลายอัน → auto merge เป็น parent
# ได้ context ที่ครบกว่า โดยไม่ต้อง send tokens เยอะ
```

### LlamaParse — รองรับภาษาไทย

```python
from llama_parse import LlamaParse

parser = LlamaParse(
    result_type="markdown",   # หรือ "text"
    language="th",
    api_key="llx-..."
)
docs = parser.load_data("hr_policy.pdf")
# Preserve ตาราง + headers ได้ดีมาก
```

### Incremental Update (ไม่ Embed ซ้ำ)

```python
async def update_document(doc_id: str, new_content: str, metadata: dict):
    # 1. ลบ nodes เก่าที่มี doc_id นี้
    vector_store.delete(ref_doc_id=doc_id)

    # 2. Ingest ใหม่ — pipeline cache จะ embed เฉพาะ chunk ที่เปลี่ยน
    new_doc = Document(text=new_content, metadata=metadata, id_=doc_id)
    await pipeline.arun(documents=[new_doc])
```

### TypeScript vs Python

| Feature | Python | TypeScript |
|---------|--------|------------|
| IngestionPipeline + Cache | ✅ built-in | ❌ ต้องเขียนเอง |
| SemanticSplitter | ✅ | ❌ |
| HierarchicalNodeParser | ✅ | ❌ |
| QuestionsAnsweredExtractor | ✅ | ❌ |
| PGVectorStore | ✅ | ✅ |
| Community / examples | ✅✅✅ | ✅✅ |

**แนะนำ:** Python FastAPI (RAG Core) + TypeScript Next.js (UI/API Layer)

## ข้อดี-ข้อเสีย

**ข้อดี:**
- ✅ ออกแบบมาเพื่อ RAG โดยตรง — API สะอาด ตรงไปตรงมา
- ✅ IngestionPipeline + Cache = handle incremental update ได้สวย
- ✅ Metadata extraction อัตโนมัติดีมาก (`QuestionsAnsweredExtractor`)
- ✅ Query modes หลากหลาย (HyDE, SubQuestionQueryEngine, RouterQueryEngine)
- ✅ ParadeDB / PGVector integration ดีมาก
- ✅ ง่ายกว่า LangChain สำหรับ RAG use case
- ✅ Debug ง่ายกว่า LangChain เพราะ abstraction ไม่ลึกเกิน
- ✅ Chunking experiments — มี NodeParsers ครบ
- ✅ LlamaParse: PDF parser ดีที่สุด รองรับภาษาไทย

**ข้อเสีย:**
- ❌ Agent/Tool ecosystem เล็กกว่า LangChain
- ❌ Community เล็กกว่า LangChain (แต่กำลังโต)
- ❌ TypeScript version มี feature น้อยกว่า Python ชัดเจน

## ตัวอย่าง / กรณีศึกษา

**Sellsuki Knowledge Bot ด้วย LlamaIndex:**
```python
# IngestionPipeline พร้อม cache
pipeline = IngestionPipeline(
    transformations=[
        SentenceSplitter(chunk_size=512, chunk_overlap=64),
        TitleExtractor(nodes=5),
        QuestionsAnsweredExtractor(questions=3),
        OpenAIEmbedding(model="text-embedding-3-small"),
    ],
    vector_store=vector_store,
    cache=cache,  # ← Dragonfly
)

# QueryEngine พร้อม Reranker
query_engine = index.as_query_engine(
    similarity_top_k=10,
    node_postprocessors=[SentenceTransformerRerank(
        model="cross-encoder/ms-marco-MiniLM-L-6-v2",
        top_n=3
    )],
    response_mode="tree_summarize"
)

# Query พร้อม Metadata Filter
response = await query_engine.aquery(
    "ลาพักร้อนได้กี่วัน?",
    filters=filters
)

print(response.response)           # คำตอบ
for node in response.source_nodes:
    print(node.metadata["source"])  # แหล่งที่มา
    print(node.score)               # relevance score
```

## เปรียบเทียบกับ LangChain

| เกณฑ์ | LangChain | LlamaIndex |
|-------|-----------|------------|
| **Focus** | General-purpose LLM apps | RAG/Data-focused |
| **RAG Features** | พอใช้ | ครบและดีกว่า |
| **Ingestion Pipeline** | Manual | ✅ Built-in |
| **Incremental Update** | Manual | ✅ Built-in Cache |
| **Metadata Handling** | ✅ | ✅✅ (ดีกว่า) |
| **Chunking Strategies** | หลายแบบ | หลากหลายกว่า |
| **Agent/Workflow** | ✅✅ (LangGraph) | ✅ (LlamaIndex Workflows) |
| **Observability** | ✅ LangSmith | ✅ LlamaTrace |
| **Debug ง่าย** | ❌ | ✅ |
| **Time to Production (RAG)** | 3-6 สัปดาห์ | 2-4 สัปดาห์ |

## ควรใช้เมื่อไหร่

- ✅ RAG คือ core use case
- ✅ ต้องการ incremental update ที่ดี
- ✅ Metadata handling ซับซ้อน (permission filter, multi-level access)
- ✅ เอกสารซับซ้อน (PDF ที่มีตาราง, layout ยาก)
- ✅ ต้องการ LLM-generated metadata (Q&A extraction)
- ✅ ต้องการ hierarchical chunking
- ✅ Debug ง่ายสำคัญกว่า feature ecosystem

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/langchain-framework|LangChain Framework]] — คู่แข่งหลัก เน้น general-purpose
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — LlamaIndex เหมาะกับ RAG ที่สุด
- [[wiki/concepts/rag-chunking-strategies|RAG Chunking Strategies]] — LlamaIndex มี NodeParsers หลากหลาย
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search]] — LlamaIndex PGVectorStore รองรับ ParadeDB
- [[wiki/concepts/semantic-caching|Semantic Caching]] — LlamaIndex RedisKVStore ใช้กับ Dragonfly ได้

## แหล่งที่มา

- [[wiki/sources/llamaindex-deep-dive|LlamaIndex & RAG Deep Dive]]
- [[wiki/sources/llamaindex-full-guide|LlamaIndex Full Guide — Sellsuki RAG System]]
- [[wiki/sources/langchain-llamaindex-deep-dive|LangChain & LlamaIndex — Deep Dive Complete Guide]]
- [[wiki/sources/rag-decision-guide|RAG Framework Decision Guide — Sellsuki]]
