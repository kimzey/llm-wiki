---
title: "LangChain & LlamaIndex — Deep Dive Complete Guide (OpenRAG)"
type: source
source_file: raw/notes/openrag/langchain-llamaindex-deep-dive.md
url: ""
published: 2026-01-01
tags: [langchain, llamaindex, rag, lcel, ingestion-pipeline, retriever, reranking, vector-store, agent]
related: [wiki/concepts/langchain-framework.md, wiki/concepts/llamaindex-framework.md, wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/lcel-langchain-expression-language.md]
created: 2026-04-14
updated: 2026-04-14
---

**Full source**: [[../../raw/notes/openrag/langchain-llamaindex-deep-dive.md|Original file]]

## สรุป

คู่มือเปรียบเทียบ LangChain และ LlamaIndex แบบ Deep Dive ตั้งแต่ศูนย์จนถึง production — ครอบคลุม Building Blocks ของทั้งสอง framework, ความแตกต่างในแนวคิด (LangChain = Swiss Army Knife, LlamaIndex = Specialized Search Engine Builder), LlamaIndex IngestionPipeline พร้อม cache, Retrievers, Reranking, QueryEngine, และการเลือกใช้ framework ที่เหมาะสม

## ประเด็นสำคัญ

- **Mental Model**: LangChain เน้น "AI ทำอะไรได้?", LlamaIndex เน้น "ข้อมูลที่ AI รู้มาจากไหน?" — ต่างกันในจุดมุ่งหมาย
- **LangChain 10 Building Blocks**: LLM, PromptTemplate, OutputParser, LCEL Chain, Memory, Document Loaders, Text Splitters, Embeddings+VectorStore, RetrievalChain, Tools+Agent
- **LlamaIndex Mental Model**: Ingestion (Documents → Nodes → Embeddings → Index) + Querying (Question → Retrieve → Synthesize → Answer)
- **LlamaIndex Document**: มี `id_`, `excluded_llm_metadata_keys`, `excluded_embed_metadata_keys` สำหรับควบคุม metadata อย่างละเอียด
- **NodeParsers**: `SentenceSplitter`, `SemanticSplitterNodeParser` (วัด embedding similarity), `HierarchicalNodeParser` (parent-child), `MarkdownNodeParser`
- **IngestionPipeline**: รวม chunk + extract + embed ใน pipeline เดียว + **cache อัตโนมัติ** — chunk เดิมไม่ embed ซ้ำ
- **Retrievers + Post-processors**: `VectorIndexRetriever` + `MetadataFilters` + `SentenceTransformerRerank` (cross-encoder)
- **QueryEngine Types**: `RetrieverQueryEngine`, `SubQuestionQueryEngine`, `RouterQueryEngine`, `TransformQueryEngine`

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- `HierarchicalNodeParser` สร้าง nodes หลายขนาด (2048/512/128 tokens) — ค้นหาด้วย leaf nodes เล็ก แต่ส่ง parent ใหญ่ให้ LLM เพื่อ context ที่ดีขึ้น
- `QuestionsAnsweredExtractor` สร้าง Q&A อัตโนมัติจากแต่ละ chunk — ช่วย retrieval เมื่อคำถาม user ตรงกับ Q ที่สร้างไว้
- `IngestionCache` ใช้ Redis (Dragonfly) เก็บ hash ของ chunk — ถ้า content ไม่เปลี่ยน ข้ามขั้น embed ไปเลย
- `index.delete_ref_doc("doc-id", delete_from_docstore=True)` — LlamaIndex รองรับ document update/delete ด้วย ID ที่ตั้งไว้

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่มีข้อขัดแย้ง — เนื้อหา deep dive ที่เพิ่ม detail ให้กับ LangChain และ LlamaIndex knowledge เดิม

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/langchain-framework|LangChain Framework]] — ส่วนที่ 2 ของ document
- [[wiki/concepts/llamaindex-framework|LlamaIndex Framework]] — ส่วนที่ 3 ของ document
- [[wiki/concepts/lcel-langchain-expression-language|LCEL]] — Block 4 ใน LangChain section
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — Retrieval + QueryEngine ใน LlamaIndex section
- [[wiki/concepts/rag-chunking-strategies|RAG Chunking Strategies]] — NodeParsers ใน LlamaIndex section
