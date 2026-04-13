---
title: "LlamaIndex Phase 4 — Advanced Features, Agents & Production"
type: source
source_file: raw/notes/liamaindex/llamaindex-phase4-agents-production.md
tags: [llamaindex, rag, tutorial, agents, production, advanced]
related: [wiki/concepts/llamaindex-framework.md, wiki/concepts/ai-agent.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/liamaindex/llamaindex-phase4-agents-production.md|Original file]]

## สรุป

บทช่วยสอน LlamaIndex Phase 4: Advanced Features, Agents & Production — LlamaIndex Agents (ReActAgent, OpenAIAgent), Tools (FunctionTool, QueryEngineTool, Custom Tools), Multi-Index Agents, Workflows (event-driven pipeline), Multi-Modal (รูปภาพ + ข้อความ), Structured Data & SQL (NL to SQL, Hybrid), Fine-tuning Embeddings (EmbeddingAdapterFinetuneEngine), Caching Strategies (LLM Response Cache, Ingestion Cache, Semantic Cache), Async & Performance, FastAPI Integration, Streaming via SSE, Best Practices, Troubleshooting, และ Production Stack Recommendations

## ประเด็นสำคัญ

### Agents
- **ReActAgent**: พื้นฐาน — LLM ตัดสินใจเองว่าจะใช้ tool ไหน (Reasoning + Acting)
- **OpenAIAgent**: Function Calling — ใช้ OpenAI's native function calling
- **QueryEngineTool**: ให้ Agent ค้นหาใน documents
- **Multi-Index Agents**: หลาย documents, หลาย indexes (เช่น รายงานปี 2022 vs 2023)
- **Custom Tools**: สร้างจาก BaseTool + ToolMetadata + Pydantic schema

### Workflows (LlamaIndex 0.10.43+)
- **Event-driven pipeline**: StartEvent → QueryEvent → RetrieveEvent → AnswerEvent → StopEvent
- **Async steps**: `@step` decorator + `async def`
- **Custom workflows**: สร้าง RAG workflow, Agent workflow แบบ custom

### Multi-Modal
- **MultiModalVectorStoreIndex**: รองรับทั้งข้อความและรูปภาพ (PDF, PNG, JPG)
- **OpenAIMultiModal**: gpt-4o สำหรับ image understanding
- **ImageDocument**: ส่งรูปภาพโดยตรง

### Structured Data & SQL
- **NLSQLTableQueryEngine**: Natural Language to SQL — generate SQL → execute → explain
- **SQLJoinQueryEngine**: Hybrid SQL + Vector search
- **Use case**: Product pricing (SQL) + Company policies (Vector)

### Fine-tuning Embeddings
- **EmbeddingAdapterFinetuneEngine**: fine-tune embedding model ให้เหมาะกับ domain
- **generate_qa_embedding_pairs**: สร้าง training data จาก nodes
- **LinearAdapterEmbeddingModel**: adapter layer บน base model

### Caching Strategies
- **LLM Response Cache**: SimpleCache — cache  responses อัตโนมัติ
- **Ingestion Cache**: IngestionCache + RedisKVStore — skip docs ที่ process แล้ว
- **Semantic Cache**: RedisChatStore + ChatMemoryBuffer — cache ตาม similarity

### Production
- **FastAPI Integration**: lifespan (startup/shutdown), async endpoints, streaming
- **Streaming via SSE**: Server-Sent Events สำหรับ real-time streaming
- **Async Performance**: `afrom_documents`, `aquery`, `asyncio.gather`
- **Monitoring**: Arize Phoenix, LangFuse
- **Best Practices**: chunking (512-1024), retrieval (top_k=5-10, Hybrid), LLM (temperature=0), production stack

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- **ReActAgent Example**:
  ```python
  agent = ReActAgent.from_tools(
      tools=[multiply_tool, add_tool, doc_tool],
      llm=llm,
      verbose=True,
      max_iterations=10
  )
  response = agent.chat("20 คูณ 53 แล้วบวก 100 เท่ากับเท่าไร?")
  # Agent จะ: multiply(20, 53) → add(1060, 100) → 1160
  ```

- **Workflow Example**:
  ```python
  class RAGWorkflow(Workflow):
      @step
      async def receive_query(self, event: StartEvent) -> QueryEvent:
          return QueryEvent(query=event.query)

      @step
      async def retrieve(self, event: QueryEvent) -> RetrieveEvent:
          retriever = index.as_retriever(similarity_top_k=5)
          nodes = await retriever.aretrieve(event.query)
          return RetrieveEvent(nodes=nodes, query=event.query)

      @step
      async def synthesize(self, event: RetrieveEvent) -> StopEvent:
          context = "\n".join([n.text for n in event.nodes])
          llm = OpenAI(model="gpt-4o")
          response = await llm.acomplete(f"Context: {context}\n\nQ: {event.query}\nA:")
          return StopEvent(result=str(response))

  workflow = RAGWorkflow(timeout=60, verbose=True)
  result = await workflow.run(query="LlamaIndex คืออะไร?")
  ```

- **FastAPI Integration**:
  ```python
  @asynccontextmanager
  async def lifespan(app: FastAPI):
      global query_engine
      Settings.llm = OpenAI(model="gpt-4o-mini")
      docs = SimpleDirectoryReader("./data").load_data()
      index = VectorStoreIndex.from_documents(docs)
      query_engine = index.as_query_engine(similarity_top_k=5)
      yield

  @app.post("/query")
  async def query(request: QueryRequest):
      response = await query_engine.aquery(request.question)
      sources = [{"text": n.text[:200], "score": n.score} for n in response.source_nodes]
      return QueryResponse(answer=str(response), sources=sources)
  ```

- **Production Stack Recommendations**:
  ```python
  # LLM
  Settings.llm = OpenAI(model="gpt-4o-mini")  # หรือ Anthropic Claude

  # Embedding
  Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")

  # Node Parser
  HierarchicalNodeParser(chunk_sizes=[2048, 512, 128])

  # Vector Store
  ChromaDB (development) | Pinecone/Weaviate (production)

  # Retrieval
  QueryFusionRetriever (Hybrid Vector + BM25)
  + AutoMergingRetriever
  + CohereRerank

  # Serving
  FastAPI + async + streaming

  # Monitoring
  Arize Phoenix / LangFuse
  ```

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/llamaindex-framework.md|LlamaIndex Framework]]
- [[wiki/concepts/ai-agent.md|AI Agent]]
- [[wiki/concepts/agentic-rag.md|Agentic RAG]]
- [[wiki/concepts/rag-evaluation.md|RAG Evaluation]]
