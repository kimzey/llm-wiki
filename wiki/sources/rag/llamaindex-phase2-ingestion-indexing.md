---
title: "LlamaIndex Phase 2 — Data Ingestion & Indexing"
type: source
source_file: raw/notes/liamaindex/llamaindex-phase2-ingestion-indexing.md
tags: [llamaindex, rag, tutorial, ingestion, indexing]
related: [wiki/concepts/llamaindex-framework.md, wiki/concepts/rag-chunking-strategies.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/liamaindex/llamaindex-phase2-ingestion-indexing.md|Original file]]

## สรุป

บทช่วยสอน LlamaIndex Phase 2: เจาะลึก Data Ingestion & Indexing — LlamaHub (100+ loaders) สำหรับโหลดข้อมูลจากแหล่งต่างๆ (PDF, Web, Database, Notion, Google Docs, JSON, CSV, Excel), Node Parsers (SentenceSplitter, SemanticSplitterNodeParser, HierarchicalNodeParser, MarkdownNodeParser), IngestionPipeline พร้อม cache, Index Types (VectorStoreIndex, SummaryIndex, TreeIndex, KeywordTableIndex, KnowledgeGraphIndex), Vector Stores (Chroma, Pinecone, Weaviate, Qdrant), และ Persist & Load Index

## ประเด็นสำคัญ

### Data Loaders (LlamaHub)
- **SimpleDirectoryReader**: โหลดไฟล์จาก folder (รองรับ recursive, required_exts, exclude)
- **PDFReader**: auto-detect จาก SimpleDirectoryReader หรือใช้โดยตรง
- **Web Readers**: SimpleWebPageReader, SitemapReader (crawl ทั้ง website)
- **Database Reader**: PostgreSQL, MySQL, SQLite, MSSQL
- **Notion / Google Docs / Drive**: official integrations
- **JSON/CSV/Excel Readers**: JSONReader, CSVReader, PandasExcelReader
- **Custom Reader**: สร้างเองจาก `BaseReader`

### Node Parsers
- **SentenceSplitter**: แนะนำสำหรับงานทั่วไป (chunk_size, chunk_overlap, separator)
- **SemanticSplitterNodeParser**: ฉลาดที่สุด — แบ่งตาม semantic similarity ไม่ใช่ token count
- **HierarchicalNodeParser**: สร้าง hierarchy ([2048, 512, 128]) สำหรับ parent-child retrieval (AutoMerging)
- **MarkdownNodeParser**: แบ่งตาม Markdown sections
- **SimpleFileNodeParser**: เลือก parser อัตโนมัติตาม file type
- **Metadata Extraction**: TitleExtractor, QuestionsAnsweredExtractor, SummaryExtractor, KeywordExtractor

### IngestionPipeline
- **orchestrate transformations**: parse → split → extract metadata → embed
- **Cache**: IngestionCache สำหรับ incremental processing (skip docs ที่ process แล้ว)
- **Redis Cache**: production-grade cache ด้วย RedisKVStore

### Index Types
- **VectorStoreIndex**: ใช้บ่อยที่สุด — vector similarity search
- **SummaryIndex**: nodes เรียงต่อกัน — เหมาะ summarization
- **TreeIndex**: tree structure of summaries (bottom-up)
- **KeywordTableIndex**: keyword-based retrieval
- **KnowledgeGraphIndex**: สร้าง KG จาก documents (entity relationships)

### Vector Stores
- **Chroma**: แนะนำสำหรับ development (persistent local)
- **Pinecone**: production (cloud, serverless)
- **Weaviate**: open-source, GraphQL API
- **Qdrant**: high-performance, filter support
- **StorageContext**: จัดการ vector_store, docstore, index_store

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- **IngestionPipeline พร้อม Cache**:
  ```python
  pipeline = IngestionPipeline(
      transformations=[SentenceSplitter(), OpenAIEmbedding()],
      cache=IngestionCache(cache=RedisKVStore(redis_url="..."))
  )
  # ครั้งต่อไป: skip docs ที่ cache แล้ว (เร็วกว่ามาก!)
  nodes = pipeline.run(documents=new_documents)
  ```

- **Hierarchical Node Parser**:
  ```python
  parser = HierarchicalNodeParser.from_defaults(chunk_sizes=[2048, 512, 128])
  nodes = parser.get_nodes_from_documents(documents)
  leaf_nodes = get_leaf_nodes(nodes)  # nodes ที่เล็กที่สุด
  ```

- **Persist & Load**:
  ```python
  index.storage_context.persist(persist_dir="./storage")
  # โหลดกลับ
  storage_context = StorageContext.from_defaults(persist_dir="./storage")
  index = load_index_from_storage(storage_context)
  ```

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/llamaindex-framework|LlamaIndex Framework]]
- [[wiki/concepts/rag-chunking-strategies|RAG Chunking Strategies]]
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG — Retrieval-Augmented Generation]]
