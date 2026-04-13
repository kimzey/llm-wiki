---
title: "LangChain & LlamaIndex — Deep Dive Complete Guide"
type: source
source_file: raw/notes/tool-rag/langchain-llamaindex-deep-dive.md
url: ""
published: 2026-01-01
tags: [langchain, llamaindex, rag, lcel, agent, ingestion-pipeline, python]
related: [wiki/concepts/langchain-framework.md, wiki/concepts/llamaindex-framework.md, wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/rag-chunking-strategies.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/tool-rag/langchain-llamaindex-deep-dive.md|Original file]]

## สรุป

คู่มือ Deep Dive อธิบาย LangChain และ LlamaIndex ตั้งแต่ศูนย์จนถึง production ครอบคลุม Building Blocks ทุกตัว พร้อม code examples ภาษาไทยละเอียด

## ประเด็นสำคัญ

### ภาพรวม

```
LangChain   = "Swiss Army Knife"  — เน้น Chain, Agent, Tool, Memory
LlamaIndex  = "Specialized Search Engine Builder"  — เน้น Index, Query, Retrieval, RAG
```

### LangChain Building Blocks

| Block | คือ | ตัวอย่าง |
|-------|-----|---------|
| LLM / Chat Model | สมอง | `ChatOpenAI(model="gpt-4o")` |
| Prompt Template | แม่พิมพ์ | `ChatPromptTemplate.from_messages([...])` |
| Output Parser | แปลง output | `StrOutputParser`, `PydanticOutputParser` |
| Chain (LCEL) | เชื่อม component ด้วย `\|` | `prompt \| llm \| StrOutputParser()` |
| Memory | จำ conversation | `RedisChatMessageHistory` ใน Dragonfly |
| Document Loaders | โหลดไฟล์ | PDF, Notion, Git, Web, Slack |
| Text Splitter | แบ่ง chunk | `RecursiveCharacterTextSplitter`, `MarkdownHeaderTextSplitter` |
| Embeddings + VectorStore | embed + เก็บ | `OpenAIEmbeddings` + `PGVector` (ParadeDB) |
| Retrieval Chain | RAG ครบวงจร | `RetrievalQA`, LCEL RAG chain |
| Tools & Agent | AI ตัดสินใจเอง | `@tool` decorator + `AgentExecutor` |

**LCEL (LangChain Expression Language)** — syntax `|` เหมือน Unix pipe:
```python
chain = prompt | llm | StrOutputParser()
result = chain.invoke({"question": "ลาพักร้อนกี่วัน?"})
```

**RunnableParallel** — ทำหลายอย่างพร้อมกัน:
```python
parallel_chain = RunnableParallel(
    answer=(prompt | llm | StrOutputParser()),
    question=RunnablePassthrough(),
)
```

### LlamaIndex Building Blocks

| Block | คือ | ตัวอย่าง |
|-------|-----|---------|
| Document | เอกสารทั้งชิ้น (ก่อน chunk) | `Document(text=..., metadata=..., id_=...)` |
| Node (TextNode) | chunk ที่แตกจาก Document | มี relationships: PREVIOUS, NEXT, PARENT, CHILD, SOURCE |
| NodeParser | chunking strategies | `SentenceSplitter`, `SemanticSplitter`, `HierarchicalNodeParser` |
| IngestionPipeline | pipeline: chunk → extract metadata → embed | Built-in cache → skip ที่ไม่เปลี่ยน |
| VectorStoreIndex | index บน vector store | `VectorStoreIndex.from_vector_store(vector_store)` |
| QueryEngine | ตอบคำถาม | `.as_query_engine(similarity_top_k=10, node_postprocessors=[reranker])` |
| Retriever | ค้นหา nodes | `VectorIndexRetriever`, `AutoMergingRetriever` |
| ResponseSynthesizer | สร้างคำตอบจาก nodes | `tree_summarize`, `compact`, `refine` |

**IngestionPipeline** — หัวใจของ LlamaIndex:
```python
pipeline = IngestionPipeline(
    transformations=[
        SentenceSplitter(chunk_size=512, chunk_overlap=64),
        TitleExtractor(nodes=5),
        QuestionsAnsweredExtractor(questions=3),  # LLM generate Q&A อัตโนมัติ
        OpenAIEmbedding(model="text-embedding-3-small"),
    ],
    vector_store=vector_store,
    cache=cache,  # ← RedisKVStore บน Dragonfly
)
```

**Metadata ที่ควรแนบทุก Document:**
```python
metadata={
    "source": "hr_policy.md",
    "department": "hr",
    "doc_type": "policy",
    "access_level": "all",
    "last_updated": "2025-01-01",
    "language": "th",
    "parent_doc_id": "hr-policy-2025",
}
```

**LangChain Agent ทำงานแบบ dynamic:**
```
คำถาม → AI คิด → เลือก Tool → ดูผล → คิดอีก → คำตอบ
```
เทียบกับ Chain: กำหนด steps ไว้ล่วงหน้า (deterministic)

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- `excluded_llm_metadata_keys` ใน LlamaIndex: กำหนดว่า metadata ไหนไม่ต้องส่งให้ LLM (ประหยัด tokens)
- `excluded_embed_metadata_keys`: กำหนด metadata ที่ไม่ต้องนำไป embed (เช่น version number)
- Node มี `text_template` ที่กำหนดว่า LLM จะเห็นข้อมูลในรูปแบบไหน
- LlamaIndex `QuestionsAnsweredExtractor` ใช้ LLM สร้าง Q&A จากแต่ละ chunk → ช่วย retrieval quality สูงขึ้น

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/langchain-framework|LangChain Framework]]
- [[wiki/concepts/llamaindex-framework|LlamaIndex Framework]]
- [[wiki/concepts/rag-chunking-strategies|RAG Chunking Strategies]]
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG — Retrieval-Augmented Generation]]
