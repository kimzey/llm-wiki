---
title: "LlamaIndex Phase 3 — Querying, Retrieval & RAG Pipeline"
type: source
source_file: raw/notes/liamaindex/llamaindex-phase3-querying-rag.md
tags: [llamaindex, rag, tutorial, querying, retrieval, advanced-rag]
related: [wiki/concepts/llamaindex-framework.md, wiki/concepts/rag-evaluation.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/liamaindex/llamaindex-phase3-querying-rag.md|Original file]]

## สรุป

บทช่วยสอน LlamaIndex Phase 3: เจาะลึก Querying, Retrieval & RAG Pipeline — Query Engine vs Chat Engine vs Retriever, Retrievers (Vector, BM25, Hybrid, AutoMerging, Recursive), Node Postprocessors (Similarity, Keyword, CohereRerank, LLMRerank), Response Synthesizers (refine, compact, tree_summarize), Query Engines (RetrieverQueryEngine, SubQuestion, Router, MultiStep), Chat Engines (Simple, Context, CondensePlusContext, OpenAI), Advanced RAG (HyDE, Sentence Window, RAG Fusion), Evaluation (Faithfulness, Relevancy, Correctness), และ Observability (Arize Phoenix, Callbacks)

## ประเด็นสำคัญ

### Retrievers
- **VectorIndexRetriever**: default — semantic similarity search
- **BM25Retriever**: keyword-based search
- **QueryFusionRetriever**: Hybrid (Vector + BM25) — ดีที่สุดสำหรับงานทั่วไป, modes: reciprocal_rerank, relative_score, dist_based_score
- **AutoMergingRetriever**: parent-child retrieval — ดึง leaf nodes แต่ return parent ถ้า children ครบ threshold
- **RecursiveRetriever**: follow references ไปยัง related nodes

### Node Postprocessors
- **SimilarityPostprocessor**: filter ด้วย similarity_cutoff
- **KeywordNodePostprocessor**: required_keywords / exclude_keywords
- **CohereRerank**: ดีที่สุด — re-rank ด้วย Cohere API (top_n)
- **LLMRerank**: re-rank ด้วย LLM
- **Pipeline**: ใช้หลาย postprocessors ต่อกัน (filter → keyword → rerank)

### Response Modes
| Mode | คำอธิบาย | เมื่อไรใช้ |
|------|----------|----------|
| `refine` | LLM call ทีละ node แล้ว refine | เนื้อหาซับซ้อน |
| `compact` | รวม nodes แล้ว call ครั้งเดียว (default) | งานทั่วไป |
| `tree_summarize` | สรุปแบบ tree (bottom-up) | summarization |
| `simple_summarize` | ตัด nodes ให้ fit context แล้ว summarize | เร็ว |
| `accumulate` | รวม responses จากทุก node | เปรียบเทียบ |

### Query Engines
- **RetrieverQueryEngine**: manual assembly — retriever + synthesizer + postprocessors
- **SubQuestionQueryEngine**: แยกคำถามซับซ้อนออกเป็น sub-questions
- **RouterQueryEngine**: เลือก query engine อัตโนมัติ (LLMSingleSelector)
- **MultiStepQueryEngine**: ถามซ้ำๆ จนได้คำตอบ (iterative)

### Chat Engines
- **SimpleChatEngine**: ไม่มี context จาก index (เฉพาะ chat history)
- **ContextChatEngine**: มี index context + memory
- **CondensePlusContextChatEngine**: แนะนำ — condense คำถาม (รวม history) แล้วค้นหา context
- **Custom Memory**: ChatMemoryBuffer (token_limit, chat_store)

### Advanced RAG
- **HyDE**: Hypothetical Document Embeddings — generate สมมติคำตอบก่อน แล้วค้นหาจาก embedding
- **Query Rewriting**: DecomposeQueryTransform
- **Sentence Window Retrieval**: ดึง node ที่ match แต่ return window ของ sentences รอบๆ
- **RAG Fusion**: QueryFusionRetriever — generate หลาย query variations แล้วรวม results

### Evaluation
- **Faithfulness**: คำตอบ hallucinate หรือเปล่า? (0-1 score)
- **Relevancy**: context ที่ retrieve มา relevant กับ query ไหม?
- **Correctness**: คำตอบถูกต้องไหม? (ต้องมี reference answer, score 1-5)
- **BatchEvalRunner**: run multiple evaluators parallel

### Observability
- **Arize Phoenix**: web UI (localhost:6006) สำหรับ tracing
- **LlamaDebugHandler**: log ทุก operation (get_llm_inputs_outputs)
- **Verbose Mode**: แสดง intermediate steps

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- **Production-Ready RAG Pipeline**:
  ```python
  # Hybrid Retrieval (Vector + BM25) + AutoMerging + CohereRerank
  hybrid_retriever = QueryFusionRetriever(
      retrievers=[vector_retriever, bm25_retriever],
      mode="reciprocal_rerank",
      use_async=True
  )
  auto_merging_retriever = AutoMergingRetriever(hybrid_retriever, storage_context)
  query_engine = RetrieverQueryEngine(
      retriever=auto_merging_retriever,
      node_postprocessors=[CohereRerank(top_n=5)]
  )
  ```

- **RAG Fusion (Multiple Query Generation)**:
  ```python
  retriever = QueryFusionRetriever(
      retrievers=[index.as_retriever()],
      num_queries=4,  # generate 4 query variations
      mode="reciprocal_rerank"
  )
  # LlamaIndex จะ generate:
  # Original, "อธิบาย LlamaIndex", "LlamaIndex ทำงานยังไง", "การใช้งาน LlamaIndex"
  ```

- **Sentence Window Retrieval**:
  ```python
  parser = SentenceWindowNodeParser.from_defaults(window_size=3)
  query_engine = index.as_query_engine(
      node_postprocessors=[
          MetadataReplacementPostProcessor(target_metadata_key="window")
      ]
  )
  ```

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/llamaindex-framework.md|LlamaIndex Framework]]
- [[wiki/concepts/rag-evaluation.md|RAG Evaluation]]
- [[wiki/concepts/hybrid-search-bm25-vector.md|Hybrid Search — BM25 + Vector Search]]
- [[wiki/concepts/agentic-rag.md|Agentic RAG]]
