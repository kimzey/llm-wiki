# LlamaIndex Deep Dive — Phase 4: Advanced Features, Agents & Production

> **ซีรีส์นี้แบ่งเป็น 4 Phase:**
> - Phase 1 → Introduction & Core Concepts
> - Phase 2 → Data Ingestion & Indexing
> - Phase 3 → Querying, Retrieval & RAG Pipeline
> - **Phase 4** → Advanced Features, Agents & Production *(ไฟล์นี้)*

---

## 1. LlamaIndex Agents

Agent คือ LLM ที่สามารถ **ตัดสินใจเองว่าจะใช้ tool ไหน** เพื่อตอบคำถาม

### 1.1 ReActAgent (พื้นฐาน)

```python
from llama_index.core.agent import ReActAgent
from llama_index.core.tools import FunctionTool, QueryEngineTool
from llama_index.llms.openai import OpenAI

# สร้าง Tools
def multiply(a: float, b: float) -> float:
    """คูณตัวเลข a และ b"""
    return a * b

def add(a: float, b: float) -> float:
    """บวกตัวเลข a และ b"""
    return a + b

def search_web(query: str) -> str:
    """ค้นหาข้อมูลจากอินเทอร์เน็ต"""
    # implement web search
    return f"ผลการค้นหา: {query}"

# สร้าง FunctionTools
multiply_tool = FunctionTool.from_defaults(fn=multiply)
add_tool = FunctionTool.from_defaults(fn=add)
search_tool = FunctionTool.from_defaults(fn=search_web)

# สร้าง Agent
llm = OpenAI(model="gpt-4o")
agent = ReActAgent.from_tools(
    tools=[multiply_tool, add_tool, search_tool],
    llm=llm,
    verbose=True,      # แสดง reasoning steps
    max_iterations=10  # กันวน loop ไม่หยุด
)

# ถามคำถาม
response = agent.chat("20 คูณ 53 แล้วบวก 100 เท่ากับเท่าไร?")
print(response)
# Agent จะ: ใช้ multiply(20, 53) → 1060 → add(1060, 100) → 1160
```

### 1.2 QueryEngineTool (ให้ Agent ค้นหาใน documents)

```python
from llama_index.core.tools import QueryEngineTool

# สร้าง query engine จาก index
query_engine = index.as_query_engine()

# wrap เป็น tool
doc_tool = QueryEngineTool.from_defaults(
    query_engine=query_engine,
    name="company_knowledge",
    description=(
        "ใช้สำหรับค้นหาข้อมูลเกี่ยวกับบริษัท "
        "รวมถึงนโยบาย, ผลิตภัณฑ์, และประวัติบริษัท"
    )
)

agent = ReActAgent.from_tools(
    tools=[doc_tool, search_tool],
    llm=llm,
    verbose=True
)

response = agent.chat("ราคา product A คือเท่าไร และตอนนี้ราคาตลาดเป็นยังไง?")
# Agent จะ: ค้นใน company_knowledge → ค้นจาก search_web → รวมคำตอบ
```

### 1.3 OpenAIAgent (Function Calling)

```python
from llama_index.agent.openai import OpenAIAgent

agent = OpenAIAgent.from_tools(
    tools=[multiply_tool, add_tool, doc_tool],
    llm=OpenAI(model="gpt-4o"),
    verbose=True,
    system_prompt="คุณเป็น assistant ที่ตอบคำถามเป็นภาษาไทย"
)

# Chat history preserved
response = agent.chat("สวัสดี ฉันต้องการคำนวณ 25 × 48")
response = agent.chat("แล้วบวกกับ 500 อีก")  # จำบริบทได้

# Reset
agent.reset()
```

### 1.4 Custom Tools ขั้นสูง

```python
from llama_index.core.tools import BaseTool, ToolMetadata
from llama_index.core.tools.types import ToolOutput
from pydantic import BaseModel
import requests

class WeatherInput(BaseModel):
    city: str
    unit: str = "celsius"

class WeatherTool(BaseTool):
    metadata = ToolMetadata(
        name="get_weather",
        description="ดูสภาพอากาศของเมืองที่ระบุ",
        fn_schema=WeatherInput
    )
    
    def __call__(self, city: str, unit: str = "celsius") -> ToolOutput:
        # เรียก weather API
        # api_response = requests.get(f"https://api.weather.com/{city}")
        result = f"อุณหภูมิใน {city} คือ 28°{unit[0].upper()}"
        return ToolOutput(
            content=result,
            tool_name=self.metadata.name,
            raw_input={"city": city, "unit": unit},
            raw_output=result
        )

weather_tool = WeatherTool()
agent = ReActAgent.from_tools(tools=[weather_tool], llm=llm, verbose=True)
response = agent.chat("อากาศที่กรุงเทพเป็นยังไง?")
```

---

## 2. Multi-Index Agents

### 2.1 หลาย documents, หลาย indexes

```python
from llama_index.core import VectorStoreIndex, SummaryIndex
from llama_index.core.tools import QueryEngineTool
from llama_index.agent.openai import OpenAIAgent

# สร้าง indexes สำหรับแต่ละ document
docs_2022 = SimpleDirectoryReader("./reports/2022").load_data()
docs_2023 = SimpleDirectoryReader("./reports/2023").load_data()

index_2022 = VectorStoreIndex.from_documents(docs_2022)
index_2023 = VectorStoreIndex.from_documents(docs_2023)

# tools
tools = [
    QueryEngineTool.from_defaults(
        query_engine=index_2022.as_query_engine(),
        name="report_2022",
        description="รายงานประจำปี 2022 ของบริษัท"
    ),
    QueryEngineTool.from_defaults(
        query_engine=index_2023.as_query_engine(),
        name="report_2023",
        description="รายงานประจำปี 2023 ของบริษัท"
    ),
]

agent = OpenAIAgent.from_tools(tools=tools, verbose=True)
response = agent.chat("เปรียบเทียบรายได้ปี 2022 และ 2023")
```

---

## 3. Workflows (LlamaIndex 0.10.43+)

Workflow คือ event-driven pipeline สำหรับ complex applications

```python
from llama_index.core.workflow import (
    Workflow,
    Event,
    StartEvent,
    StopEvent,
    step
)

# Define Events
class QueryEvent(Event):
    query: str

class RetrieveEvent(Event):
    nodes: list
    query: str

class AnswerEvent(Event):
    answer: str

# Define Workflow
class RAGWorkflow(Workflow):
    
    @step
    async def receive_query(self, event: StartEvent) -> QueryEvent:
        print(f"Received query: {event.query}")
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
        response = await llm.acomplete(
            f"Context: {context}\n\nQ: {event.query}\nA:"
        )
        return StopEvent(result=str(response))

# ใช้งาน
workflow = RAGWorkflow(timeout=60, verbose=True)
result = await workflow.run(query="LlamaIndex คืออะไร?")
print(result)
```

---

## 4. Multi-Modal (รูปภาพ + ข้อความ)

### 4.1 Multi-Modal Index

```python
pip install llama-index-multi-modal-llms-openai
```

```python
from llama_index.core.indices.multi_modal.base import MultiModalVectorStoreIndex
from llama_index.core import SimpleDirectoryReader

# โหลด documents (รวมรูปภาพ)
documents = SimpleDirectoryReader(
    "./data",
    required_exts=[".pdf", ".png", ".jpg", ".txt"]
).load_data()

# สร้าง multi-modal index
index = MultiModalVectorStoreIndex.from_documents(documents)

# Query engine พร้อม image understanding
query_engine = index.as_query_engine(multi_modal_llm=OpenAIMultiModal(
    model="gpt-4o",
    max_new_tokens=1000
))

response = query_engine.query("อธิบายรูปภาพในเอกสาร")
```

### 4.2 ส่ง Image โดยตรง

```python
from llama_index.multi_modal_llms.openai import OpenAIMultiModal
from llama_index.core.schema import ImageDocument

# โหลดรูปภาพ
image_doc = ImageDocument(image_path="./chart.png")
text_doc = Document(text="บริบทเพิ่มเติม...")

llm = OpenAIMultiModal(model="gpt-4o")
response = llm.complete(
    prompt="วิเคราะห์ graph ในรูปภาพนี้",
    image_documents=[image_doc]
)
print(response)
```

---

## 5. Structured Data & SQL

### 5.1 NL to SQL

```python
from llama_index.core import SQLDatabase
from llama_index.core.query_engine import NLSQLTableQueryEngine
from sqlalchemy import create_engine

# เชื่อมต่อ database
engine = create_engine("sqlite:///./mydb.sqlite")
sql_database = SQLDatabase(
    engine,
    include_tables=["orders", "customers", "products"]
)

# สร้าง query engine
query_engine = NLSQLTableQueryEngine(sql_database=sql_database)

# ถามด้วยภาษาธรรมชาติ
response = query_engine.query(
    "แสดงยอดขายรวมของแต่ละ category ในเดือนมกราคม 2024"
)
print(response)
# LlamaIndex จะ: generate SQL → execute → explain results
print(response.metadata["sql_query"])  # ดู SQL ที่ generate
```

### 5.2 SQL + Vector Search (Hybrid)

```python
from llama_index.core.query_engine import SQLJoinQueryEngine

sql_tool = QueryEngineTool.from_defaults(
    query_engine=NLSQLTableQueryEngine(sql_database),
    name="sql_engine",
    description="ค้นหาข้อมูลเชิงตัวเลข statistics จากฐานข้อมูล"
)

vector_tool = QueryEngineTool.from_defaults(
    query_engine=vector_index.as_query_engine(),
    name="vector_engine",
    description="ค้นหาข้อมูลจากเอกสาร policies และ descriptions"
)

hybrid_engine = SQLJoinQueryEngine(
    sql_query_tool=sql_tool,
    other_query_tool=vector_tool,
    sql_join_synthesis_prompt=None  # ใช้ default
)

response = hybrid_engine.query(
    "Product A มียอดขายเท่าไร และตามนโยบายบริษัทมีข้อกำหนดอะไรบ้าง?"
)
```

---

## 6. Fine-tuning Embeddings

Fine-tune embedding model ให้เหมาะกับ domain ของเรา

```python
from llama_index.finetuning import EmbeddingAdapterFinetuneEngine
from llama_index.core.evaluation import EmbeddingQAFinetuneDataset

# 1. สร้าง training data (QA pairs)
from llama_index.finetuning import generate_qa_embedding_pairs

train_dataset = generate_qa_embedding_pairs(
    nodes=nodes[:100],   # ใช้ nodes เป็น training data
    llm=OpenAI(model="gpt-4o-mini")
)
train_dataset.save_json("train_pairs.json")

# 2. Fine-tune
from llama_index.embeddings.adapter import LinearAdapterEmbeddingModel
from llama_index.finetuning import EmbeddingAdapterFinetuneEngine

base_embed_model = OpenAIEmbedding()
finetune_engine = EmbeddingAdapterFinetuneEngine(
    train_dataset,
    base_embed_model,
    model_output_path="my_embedding_adapter",
    epochs=4,
    verbose=True
)
finetune_engine.finetune()

# 3. ใช้ fine-tuned model
embed_model = finetune_engine.get_finetuned_model()
Settings.embed_model = embed_model
```

---

## 7. Caching Strategies

### 7.1 LLM Response Cache

```python
from llama_index.core.storage.cache import SimpleCache
from llama_index.core import Settings

# In-memory cache
cache = SimpleCache()

# ใช้ cache กับ LLM
llm = OpenAI(model="gpt-4o")
# LlamaIndex จะ cache responses อัตโนมัติถ้า input เหมือนกัน
```

### 7.2 Ingestion Cache

```python
from llama_index.core.ingestion import IngestionCache, IngestionPipeline
from llama_index.core.storage.kvstore import SimpleKVStore

# In-memory cache
cache = IngestionCache(cache=SimpleKVStore(), collection="my_cache")

# Redis cache (production)
from llama_index.storage.kvstore.redis import RedisKVStore
redis_cache = IngestionCache(
    cache=RedisKVStore(redis_url="redis://localhost:6379"),
    collection="my_cache"
)

pipeline = IngestionPipeline(
    transformations=[SentenceSplitter(), OpenAIEmbedding()],
    cache=redis_cache
)
```

### 7.3 Semantic Cache (Cache LLM responses by similarity)

```python
pip install llama-index-storage-chat-store-redis
```

```python
from llama_index.storage.chat_store.redis import RedisChatStore
from llama_index.core.memory import ChatMemoryBuffer

chat_store = RedisChatStore(redis_url="redis://localhost:6379", ttl=300)

memory = ChatMemoryBuffer.from_defaults(
    token_limit=3000,
    chat_store=chat_store,
    chat_store_key="user-123"
)

chat_engine = index.as_chat_engine(
    chat_mode="context",
    memory=memory
)
```

---

## 8. Async & Performance

```python
import asyncio
from llama_index.core import VectorStoreIndex

# Async indexing
async def build_index(documents):
    return await VectorStoreIndex.afrom_documents(documents)

# Async query
async def query_async(query_engine, question):
    return await query_engine.aquery(question)

# Concurrent queries
async def batch_query(query_engine, questions):
    tasks = [query_engine.aquery(q) for q in questions]
    responses = await asyncio.gather(*tasks)
    return responses

# ใช้งาน
async def main():
    docs = SimpleDirectoryReader("./data").load_data()
    index = await build_index(docs)
    qe = index.as_query_engine()
    
    questions = ["คำถาม 1", "คำถาม 2", "คำถาม 3"]
    responses = await batch_query(qe, questions)
    
    for q, r in zip(questions, responses):
        print(f"Q: {q}\nA: {r}\n")

asyncio.run(main())
```

---

## 9. FastAPI Integration

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from contextlib import asynccontextmanager
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader, Settings
from llama_index.llms.openai import OpenAI
from llama_index.embeddings.openai import OpenAIEmbedding
import os

# Global query engine
query_engine = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global query_engine
    
    # Startup: build index
    Settings.llm = OpenAI(model="gpt-4o-mini")
    Settings.embed_model = OpenAIEmbedding()
    
    docs = SimpleDirectoryReader("./data").load_data()
    index = VectorStoreIndex.from_documents(docs)
    query_engine = index.as_query_engine(similarity_top_k=5)
    
    print("Index built successfully!")
    yield
    
    # Shutdown
    print("Shutting down...")

app = FastAPI(lifespan=lifespan)

class QueryRequest(BaseModel):
    question: str
    top_k: int = 5

class QueryResponse(BaseModel):
    answer: str
    sources: list[dict]

@app.post("/query", response_model=QueryResponse)
async def query(request: QueryRequest):
    if not query_engine:
        raise HTTPException(status_code=503, detail="Index not ready")
    
    response = await query_engine.aquery(request.question)
    
    sources = [
        {
            "text": node.text[:200],
            "score": node.score,
            "metadata": node.metadata
        }
        for node in response.source_nodes
    ]
    
    return QueryResponse(
        answer=str(response),
        sources=sources
    )

@app.post("/chat")
async def chat(message: str, session_id: str = "default"):
    # implement chat engine per session
    pass

# รัน: uvicorn main:app --reload
```

---

## 10. Streaming via FastAPI (Server-Sent Events)

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

@app.post("/stream")
async def stream_query(request: QueryRequest):
    query_engine = index.as_query_engine(streaming=True)
    streaming_response = query_engine.query(request.question)
    
    async def event_generator():
        for token in streaming_response.response_gen:
            yield f"data: {token}\n\n"
        yield "data: [DONE]\n\n"
    
    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream"
    )
```

---

## 11. Best Practices & Checklist

### Chunking

```
✅ ใช้ chunk_size=512-1024 สำหรับงานทั่วไป
✅ เพิ่ม chunk_overlap=50-100 เพื่อไม่ให้ข้อมูลขาดหาย
✅ ใช้ SemanticSplitter ถ้าเนื้อหามีความหลากหลายสูง
✅ ใช้ HierarchicalNodeParser ถ้าต้องการ AutoMerging
```

### Retrieval

```
✅ เริ่มด้วย similarity_top_k=5-10
✅ ใช้ Hybrid (Vector + BM25) สำหรับงาน production
✅ เพิ่ม Reranker (Cohere/LLM) เพื่อความแม่นยำ
✅ ทดสอบ SentenceWindow หากต้องการ context รอบๆ
```

### LLM & Embedding

```
✅ ใช้ temperature=0 สำหรับ factual Q&A
✅ ใช้ gpt-4o-mini เพื่อประหยัดค่าใช้จ่าย (ปรับขึ้นถ้าต้องการ)
✅ ใช้ text-embedding-3-small แทน ada-002 (ดีกว่า, ถูกกว่า)
✅ Cache embeddings ไว้เพื่อไม่ต้อง re-embed
```

### Production

```
✅ ใช้ persistent vector store (Chroma/Pinecone/Weaviate)
✅ Implement incremental indexing (ไม่ต้อง re-index ทั้งหมด)
✅ ใช้ async endpoints
✅ Monitor ด้วย Arize Phoenix หรือ LangFuse
✅ Evaluate ด้วย Faithfulness + Relevancy metrics
✅ Implement rate limiting
```

---

## 12. Troubleshooting

### ปัญหา: คำตอบไม่ถูกต้อง / hallucinate

```python
# แก้: เพิ่ม similarity_top_k และเพิ่ม strictness ใน prompt
query_engine = index.as_query_engine(
    similarity_top_k=10,
    node_postprocessors=[SimilarityPostprocessor(similarity_cutoff=0.7)]
)

# ตรวจสอบ retrieved nodes
nodes = retriever.retrieve("query")
for n in nodes:
    print(f"Score: {n.score} | {n.text[:100]}")
```

### ปัญหา: ช้าเกินไป

```python
# แก้ 1: ใช้ async
response = await query_engine.aquery("query")

# แก้ 2: ลด top_k
query_engine = index.as_query_engine(similarity_top_k=3)

# แก้ 3: ใช้ gpt-4o-mini แทน gpt-4o

# แก้ 4: Cache embeddings ผ่าน IngestionPipeline
```

### ปัญหา: Context ไม่พอ / token limit exceeded

```python
# แก้: เล็ก chunk size ลง
Settings.chunk_size = 512

# หรือ: ใช้ compact response mode
query_engine = index.as_query_engine(response_mode="compact")
```

### ปัญหา: ไม่พบข้อมูลที่ควรจะ retrieve ได้

```python
# Debug: ดู raw retrieved nodes
nodes = retriever.retrieve("query")
print(f"Found {len(nodes)} nodes")
if len(nodes) == 0:
    print("ไม่พบ nodes - ลอง:")
    print("1. เพิ่ม top_k")
    print("2. ลด similarity_cutoff")
    print("3. ตรวจสอบว่า embed model เหมือนกับตอน index")
```

---

## สรุปภาพรวมทั้ง Series

```
Phase 1: พื้นฐาน
  Document → Node → Settings → VectorStoreIndex → QueryEngine

Phase 2: Data
  Loaders → NodeParsers → IngestionPipeline → VectorStores → Persist

Phase 3: RAG Pipeline
  Retrievers → Postprocessors → Synthesizers → ChatEngine → Evaluation

Phase 4: Advanced
  Agents → Workflows → Multi-modal → SQL → Fine-tuning → Production
```

### Stack แนะนำสำหรับ Production

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

---

## Resources

- **Official Docs**: https://docs.llamaindex.ai
- **LlamaHub**: https://llamahub.ai (loaders, tools, packs)
- **GitHub**: https://github.com/run-llama/llama_index
- **Discord**: https://discord.gg/llama-index
- **Cookbook**: https://github.com/run-llama/llama_index/tree/main/docs/docs/examples

---

*Phase 4/4 | LlamaIndex Deep Dive Series — จบแล้ว! 🎉*
