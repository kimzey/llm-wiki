---
title: "RAG Deep Dive — Retrieval-Augmented Generation ทุกเทคนิค (LangChain)"
type: source
source_file: raw/notes/LangChain/rag-deep-dive.md
url: ""
published: 2026-01-01
tags: [rag, langchain, langsmith, langgraph, advanced-rag, multi-query, reranking, contextual-compression, parent-document-retriever, conversational-rag]
related: [wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/rag-chunking-strategies.md, wiki/concepts/langchain-framework.md, wiki/concepts/agentic-rag.md, wiki/concepts/rag-evaluation.md]
created: 2026-04-14
updated: 2026-04-14
---

**Full source**: [[../../raw/notes/LangChain/rag-deep-dive.md|Original file]]

## สรุป

คู่มือ RAG เชิงลึกสำหรับ LangChain ครอบคลุมทุก step ของ RAG pipeline พร้อม code ตัวอย่างสมบูรณ์ ตั้งแต่ Document Loaders, Chunking, Embedding, Vector Stores, Retrieval methods ขั้นสูง (MMR, Multi-Query, Contextual Compression, Parent Document, Self-Query, Reranking), Conversational RAG ที่จำ history, RAG Agent ด้วย LangGraph และ Evaluation ด้วย LangSmith

## ประเด็นสำคัญ

### Document Loaders (LangChain)

```python
# PDF, Text, CSV, Word, Website, JSON, Directory
from langchain_community.document_loaders import (
    PyPDFLoader, TextLoader, CSVLoader, Docx2txtLoader,
    WebBaseLoader, JSONLoader, DirectoryLoader
)
```

- metadata สำคัญ: `source`, `department`, `category`, `version`, `last_updated`
- ควรเพิ่ม metadata เองสำหรับใช้ filter ตอน retrieval

### Chunking Strategies + Guidance

| เอกสาร | Splitter | chunk_size | overlap |
|--------|---------|------------|---------|
| เอกสารทั่วไป | RecursiveCharacter | 500 | 100 |
| FAQ (Q&A pairs) | อย่า split | — | — |
| Markdown / Wiki | MarkdownHeader + Recursive | 500 | 100 |
| Source Code | Language-specific | 500 | 50 |
| กฎหมาย / สัญญา | RecursiveCharacter | 300 | 100 |
| บทความยาว | SemanticChunker | auto | auto |

**หลักสำคัญ**: FAQ ไม่ควร split เลย (1 Q&A = 1 chunk)

### Embedding Models (LangChain)

```python
from langchain_google_genai import GoogleGenerativeAIEmbeddings
from langchain_openai import OpenAIEmbeddings
from langchain_huggingface import HuggingFaceEmbeddings  # ฟรี local
```

| Model | Dimensions | ราคา | เหมาะกับ |
|-------|-----------|------|---------|
| text-embedding-004 (Google) | 768 | ฟรี 1500 req/min | ทั่วไป — **แนะนำ** |
| text-embedding-3-small (OpenAI) | 1536 | $0.02/1M | ประหยัด |
| all-MiniLM-L6-v2 (HuggingFace) | 384 | ฟรี local | offline/ทดสอบ |

### Vector Stores (LangChain)

| Store | ประเภท | เหมาะกับ |
|-------|--------|---------|
| **Chroma** | Embedded/Server | Development, Small-Medium — **แนะนำเริ่มต้น** |
| **FAISS** | In-memory | Large dataset, Speed |
| **pgvector** | PostgreSQL ext | มี Postgres อยู่แล้ว |
| Pinecone | Cloud managed | Production Managed |
| Qdrant | Self-hosted | Production Performance |

### Retrieval Methods ขั้นสูง

**3 Search Types:**
```python
search_type="similarity"                          # ค้นหาแบบพื้นฐาน
search_type="mmr"                                 # แนะนำ — หลากหลาย
search_type="similarity_score_threshold"          # filter score
```

**Advanced Retrievers:**

1. **Multi-Query Retriever** — LLM สร้างคำถาม 3 มุมมอง → ค้นหา 3 ครั้ง → รวม
2. **Contextual Compression** — ตัดส่วนที่ไม่เกี่ยวข้องออกจาก chunk ก่อนส่ง LLM
3. **Parent Document Retriever** — ค้นหาด้วย child เล็ก → ส่ง parent ใหญ่ให้ LLM
4. **Self-Query Retriever** — LLM แปลงคำถามเป็น structured query + metadata filter อัตโนมัติ
5. **Cross-Encoder Reranking** — ดึงมา 10 chunks → cross-encoder rerank → เลือก top 3
6. **Ensemble Retriever (Hybrid)** — BM25 + Semantic ด้วย `EnsembleRetriever(weights=[0.4, 0.6])`

### Conversational RAG Pattern

```python
# Pattern: Condense question → ค้นหาด้วย condensed query → Generate ด้วย history
if chat_history:
    condensed_query = condense_prompt.invoke({
        "chat_history": history, "question": question
    })
    search_query = model.invoke(condensed_query).content
else:
    search_query = question

docs = retriever.invoke(search_query)
```

### RAG Agent ด้วย LangGraph

```python
# Pattern: StateGraph + ToolNode + tools_condition
tools = [search_hr_docs, search_it_docs, calculate]
model_with_tools = model.bind_tools(tools)

builder = StateGraph(RAGAgentState)
builder.add_node("agent", agent_node)
builder.add_node("tools", ToolNode(tools=tools))
builder.add_edge(START, "agent")
builder.add_conditional_edges("agent", tools_condition)
builder.add_edge("tools", "agent")
```

Agent สามารถเลือก tool เองตามประเภทคำถาม (HR, IT, Calculate)

### Evaluation ด้วย LangSmith

3 Custom Evaluators:
1. **answer_correctness**: expected answer ปรากฏอยู่ใน answer หรือไม่
2. **source_correctness**: อ้างอิง source ถูกต้องไหม
3. **no_hallucination**: ใช้ LLM judge ตรวจ SAFE vs HALLUCINATION

### Best Practices

**Chunking:**
- ทดลอง chunk_size หลายค่า (300, 500, 800) แล้ววัดผล
- Overlap = 10-20% ของ chunk_size
- ไม่ตัดเล็กเกิน (ขาดบริบท) หรือใหญ่เกิน (retrieval ไม่แม่น)

**Retrieval:**
- ใช้ MMR แทน similarity (ได้ผลลัพธ์หลากหลายขึ้น)
- ใช้ metadata filter เพื่อ scope การค้นหา
- k=3-5 เหมาะสม (ไม่น้อยเกินหรือมากเกิน)

**Generation:**
- บอก LLM ให้ตอบจาก context เท่านั้น ห้ามแต่งเพิ่ม
- ให้ LLM ปฏิเสธถ้าไม่มีข้อมูล
- ให้ LLM อ้างอิง source เสมอ
- `temperature=0` สำหรับ factual Q&A

**Production:**
- ใช้ persistent vector store
- Incremental indexing (อัปเดตเฉพาะที่เปลี่ยน)
- Monitor ด้วย LangSmith
- Cache embeddings ที่ใช้บ่อย
- ตั้ง rate limit สำหรับ embedding API

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- Self-Query Retriever เป็นเทคนิคที่ LLM แปลงคำถาม "นโยบายลาของ HR" เป็น `query="นโยบายลา", filter={"department": "HR"}` อัตโนมัติ — ลดภาระการเขียน filter เอง
- `RunnableParallel` ใน LangChain สามารถ run การดึง sources และสร้างคำตอบ แบบ parallel ได้
- `MMR (Maximal Marginal Relevance)` ช่วยให้ได้ chunks หลากหลาย ไม่ซ้ำเนื้อหากัน — ดีกว่า similarity search สำหรับคำถามที่ต้องการ coverage กว้าง

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบความขัดแย้งกับเนื้อหาที่มีอยู่แล้ว — เนื้อหานี้เพิ่มรายละเอียด advanced retrieval techniques ที่ยังไม่ได้ครอบคลุมใน wiki

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]]
- [[wiki/concepts/langchain-framework|LangChain Framework]]
- [[wiki/concepts/rag-chunking-strategies|RAG Chunking Strategies]]
- [[wiki/concepts/agentic-rag|Agentic RAG]]
- [[wiki/concepts/rag-evaluation|RAG Evaluation]]
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search BM25 + Vector]]
