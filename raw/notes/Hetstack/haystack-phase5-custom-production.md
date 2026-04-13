# 🌾 Haystack Deep Dive — Phase 5: Custom Components & Production

> **เป้าหมาย Phase 5:** สร้าง Custom Component เอง, Agents, Evaluation, และ Deploy สู่ Production

---

## 1. Custom Component — สร้าง Component ของตัวเอง

การสร้าง Custom Component ทำได้ง่ายมาก แค่ใช้ `@component` decorator

### 1.1 โครงสร้างพื้นฐาน

```python
from haystack import component, default_from_dict, default_to_dict
from haystack.core.component.types import Variadic

@component
class MyCustomComponent:
    """
    Custom Component ตัวอย่าง
    
    Input: text (str)
    Output: processed_text (str), word_count (int)
    """

    def __init__(self, uppercase: bool = False):
        self.uppercase = uppercase

    @component.output_types(
        processed_text=str,
        word_count=int
    )
    def run(self, text: str) -> dict:
        """
        เมธอดนี้จะถูกเรียกตอน pipeline.run()
        ต้อง return dict ที่ key ตรงกับ output_types
        """
        processed = text.upper() if self.uppercase else text
        word_count = len(text.split())
        return {
            "processed_text": processed,
            "word_count": word_count
        }
```

### 1.2 ใช้ใน Pipeline

```python
pipeline = Pipeline()
pipeline.add_component("my_comp", MyCustomComponent(uppercase=True))
pipeline.add_component("generator", OpenAIGenerator())

pipeline.connect("my_comp.processed_text", "generator.prompt")

result = pipeline.run({"my_comp": {"text": "hello world"}})
```

### 1.3 Custom Component พร้อม Optional Input

```python
from typing import Optional

@component
class QueryEnhancer:
    """เพิ่ม context ให้ query"""

    def __init__(self, prefix: str = ""):
        self.prefix = prefix

    @component.output_types(enhanced_query=str)
    def run(
        self,
        query: str,
        context: Optional[str] = None   # Optional input
    ) -> dict:
        enhanced = f"{self.prefix} {query}"
        if context:
            enhanced = f"[Context: {context}] {enhanced}"
        return {"enhanced_query": enhanced.strip()}
```

### 1.4 Custom Component ที่รองรับ List input

```python
from typing import List

@component
class DocumentScorer:
    """ให้คะแนน document ตาม keyword"""

    def __init__(self, keywords: List[str]):
        self.keywords = [kw.lower() for kw in keywords]

    @component.output_types(
        documents=List[Document],
        scores=List[float]
    )
    def run(self, documents: List[Document]) -> dict:
        scored_docs = []
        scores = []
        for doc in documents:
            content_lower = doc.content.lower()
            score = sum(1 for kw in self.keywords if kw in content_lower)
            doc.score = score / len(self.keywords)
            scored_docs.append(doc)
            scores.append(doc.score)

        # เรียงตาม score
        paired = sorted(zip(scored_docs, scores), key=lambda x: x[1], reverse=True)
        sorted_docs, sorted_scores = zip(*paired) if paired else ([], [])

        return {
            "documents": list(sorted_docs),
            "scores": list(sorted_scores)
        }
```

### 1.5 Warm-up สำหรับ Component ที่โหลด Model

```python
@component
class CustomEmbedder:
    def __init__(self, model_name: str):
        self.model_name = model_name
        self.model = None  # โหลดตอน warm_up

    def warm_up(self):
        """เรียกอัตโนมัติตอน pipeline.warm_up()"""
        from sentence_transformers import SentenceTransformer
        self.model = SentenceTransformer(self.model_name)
        print(f"✅ โหลด {self.model_name} สำเร็จ")

    @component.output_types(embedding=List[float])
    def run(self, text: str) -> dict:
        if self.model is None:
            raise RuntimeError("ต้องเรียก warm_up() ก่อน")
        embedding = self.model.encode(text).tolist()
        return {"embedding": embedding}
```

### 1.6 Serialization สำหรับ Custom Component

```python
@component
class ConfigurableComponent:
    def __init__(self, config: dict):
        self.config = config

    def to_dict(self) -> dict:
        """สำหรับบันทึกเป็น YAML"""
        return default_to_dict(self, config=self.config)

    @classmethod
    def from_dict(cls, data: dict) -> "ConfigurableComponent":
        """สำหรับโหลดจาก YAML"""
        return default_from_dict(cls, data)

    @component.output_types(result=str)
    def run(self, text: str) -> dict:
        return {"result": text}
```

---

## 2. Agents — AI ที่ตัดสินใจเองและใช้ Tools

### 2.1 Tool Definition

```python
from haystack.tools import Tool

def get_weather(city: str) -> str:
    """ดึงข้อมูลสภาพอากาศของเมือง"""
    # จริงๆ ต่อ API
    return f"อุณหภูมิใน {city}: 28°C, ท้องฟ้าแจ่มใส"

def search_documents(query: str, top_k: int = 3) -> str:
    """ค้นหาเอกสารในระบบ"""
    # จริงๆ ต่อ Retriever
    return f"ผลการค้นหา '{query}': พบ {top_k} เอกสาร"

# สร้าง Tool จาก function
weather_tool = Tool(
    name="get_weather",
    description="ดึงข้อมูลสภาพอากาศปัจจุบันของเมืองที่ระบุ",
    function=get_weather,
    parameters={
        "type": "object",
        "properties": {
            "city": {
                "type": "string",
                "description": "ชื่อเมืองที่ต้องการทราบอุณหภูมิ"
            }
        },
        "required": ["city"]
    }
)
```

### 2.2 Agent Pipeline

```python
from haystack.components.agents import ToolInvoker
from haystack.components.generators.chat import OpenAIChatGenerator
from haystack.dataclasses import ChatMessage

tools = [weather_tool, search_documents_tool]

generator = OpenAIChatGenerator(
    model="gpt-4o",
    tools=tools,
    generation_kwargs={"temperature": 0.1}
)

tool_invoker = ToolInvoker(tools=tools)

# Agent Loop Pipeline
agent_pipeline = Pipeline()
agent_pipeline.add_component("llm", generator)
agent_pipeline.add_component("tool_invoker", tool_invoker)

# Loop: LLM → Tools → LLM → Tools → ... → Final Answer
agent_pipeline.connect("llm.replies", "tool_invoker.messages")
# ต้องมี loop mechanism ด้วย AgentComponent

# ใช้ Agent Component ที่สร้าง loop อัตโนมัติ
from haystack.components.agents import Agent

agent = Agent(
    chat_generator=OpenAIChatGenerator(model="gpt-4o"),
    tools=tools,
    system_prompt="คุณเป็น Assistant ที่ใช้ tools ในการหาข้อมูล",
    max_agent_steps=5    # จำนวนรอบสูงสุด
)

result = agent.run(
    messages=[ChatMessage.from_user("อุณหภูมิที่กรุงเทพวันนี้เท่าไหร่?")]
)
print(result["messages"][-1].content)
```

---

## 3. Evaluation — วัดความแม่นยำของ RAG Pipeline

### 3.1 Metrics ที่สำคัญ

```
┌────────────────────────────────────────────────┐
│              RAG Evaluation Metrics            │
├────────────────┬───────────────────────────────┤
│ Retrieval      │ Recall@K, Precision@K, MRR    │
├────────────────┼───────────────────────────────┤
│ Generation     │ Faithfulness, Answer Relevance│
├────────────────┼───────────────────────────────┤
│ End-to-end     │ Exact Match, F1, BLEU          │
└────────────────┴───────────────────────────────┘
```

### 3.2 Built-in Evaluators

```python
from haystack.components.evaluators import (
    FaithfulnessEvaluator,       # คำตอบ faithful กับ context ไหม?
    AnswerExactMatchEvaluator,   # ตรงกับ ground truth ไหม?
    DocumentMRREvaluator,        # Mean Reciprocal Rank
    DocumentRecallEvaluator,     # Recall@K
    SASEvaluator                 # Semantic Answer Similarity
)

# Faithfulness (ต้องใช้ LLM judge)
faithfulness_evaluator = FaithfulnessEvaluator(
    api="openai",
    api_key="sk-..."
)

result = faithfulness_evaluator.run(
    questions=["Haystack คืออะไร?"],
    contexts=[["Haystack เป็น framework..."]],  # retrieved docs
    responses=["Haystack เป็น framework สำหรับสร้าง NLP pipeline"]
)
print(result["individual_scores"])  # [0.95]
print(result["score"])              # 0.95
```

### 3.3 Evaluation Pipeline

```python
from haystack.evaluation import EvaluationRunResult

# สร้าง test dataset
test_questions = [
    "Haystack คืออะไร?",
    "RAG ย่อมาจากอะไร?",
    "DocumentStore มีกี่ประเภท?"
]

ground_truths = [
    "Haystack เป็น open-source framework สำหรับ NLP",
    "Retrieval-Augmented Generation",
    "มีหลายประเภท เช่น InMemory, Qdrant, Elasticsearch"
]

# รัน RAG pipeline กับทุก question
responses = []
for q in test_questions:
    result = rag_pipeline.run({
        "embedder": {"text": q},
        "prompt_builder": {"question": q}
    })
    responses.append(result["llm"]["replies"][0])

# Evaluate
exact_match = AnswerExactMatchEvaluator()
result = exact_match.run(
    questions=test_questions,
    ground_truth_answers=ground_truths,
    predicted_answers=responses
)
print(f"Exact Match Score: {result['score']:.2%}")
```

### 3.4 RAGAS Integration

```python
# pip install ragas
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_recall

from datasets import Dataset

data = {
    "question": test_questions,
    "answer": responses,
    "contexts": [[doc.content for doc in docs] for docs in all_retrieved_docs],
    "ground_truth": ground_truths
}

dataset = Dataset.from_dict(data)
result = evaluate(dataset, metrics=[faithfulness, answer_relevancy, context_recall])
print(result)
```

---

## 4. Caching — เพิ่มประสิทธิภาพด้วย Cache

```python
from haystack.components.caching import SerializedDocumentCacheChecker, DocumentWriter
from haystack.document_stores.in_memory import InMemoryDocumentStore

cache_store = InMemoryDocumentStore()

# ตรวจสอบก่อนว่า document นี้ embed แล้วหรือยัง
cache_checker = SerializedDocumentCacheChecker(document_store=cache_store)

indexing = Pipeline()
indexing.add_component("cache_checker", cache_checker)
indexing.add_component("embedder", SentenceTransformersDocumentEmbedder(model="BAAI/bge-m3"))
indexing.add_component("cache_writer", DocumentWriter(document_store=cache_store))
indexing.add_component("doc_writer", DocumentWriter(document_store=document_store))

# Route: ถ้า cache hit → ข้าม embedder
indexing.connect("cache_checker.misses",  "embedder.documents")    # ยังไม่ได้ embed
indexing.connect("cache_checker.hits",    "doc_writer.documents")  # embed แล้ว
indexing.connect("embedder.documents",    "cache_writer.documents")
indexing.connect("embedder.documents",    "doc_writer.documents")
```

---

## 5. Logging & Monitoring

```python
import logging
import haystack

# เปิด Debug logging
logging.basicConfig(level=logging.INFO)
haystack.logging.configure_logging(use_json=True)  # JSON format สำหรับ production

# Tracing ด้วย OpenTelemetry
from haystack.tracing import OpenTelemetryTracer
import opentelemetry

tracer = OpenTelemetryTracer(
    tracer=opentelemetry.trace.get_tracer("my_app")
)
haystack.tracing.enable_tracing(tracer)

# หลังจากนี้ ทุก pipeline.run() จะมี trace
result = rag_pipeline.run({"embedder": {"text": "query"}})
```

---

## 6. Production Patterns

### 6.1 Async Pipeline

```python
import asyncio

async def main():
    # Pipeline รองรับ async
    result = await rag_pipeline.run_async({
        "embedder": {"text": "query"},
        "prompt_builder": {"question": "query"}
    })
    return result

asyncio.run(main())
```

### 6.2 FastAPI Integration

```python
from fastapi import FastAPI
from pydantic import BaseModel
from haystack import Pipeline
import uvicorn

app = FastAPI()

# โหลด pipeline ครั้งเดียวตอน startup
rag_pipeline: Pipeline = None

@app.on_event("startup")
async def startup():
    global rag_pipeline
    # โหลด pipeline จาก YAML
    with open("pipeline.yaml") as f:
        rag_pipeline = Pipeline.load(f)
    rag_pipeline.warm_up()

class QueryRequest(BaseModel):
    question: str
    filters: dict = {}

class QueryResponse(BaseModel):
    answer: str
    sources: list[str]

@app.post("/query", response_model=QueryResponse)
async def query(request: QueryRequest):
    result = rag_pipeline.run({
        "embedder":       {"text": request.question},
        "retriever":      {"filters": request.filters},
        "prompt_builder": {"question": request.question}
    })

    answer = result["llm"]["replies"][0]
    docs = result.get("retriever", {}).get("documents", [])
    sources = [doc.meta.get("file_path", "") for doc in docs]

    return QueryResponse(answer=answer, sources=sources)

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### 6.3 Environment Configuration

```python
# config/settings.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    openai_api_key: str
    qdrant_url: str = "http://localhost:6333"
    embedding_model: str = "BAAI/bge-m3"
    llm_model: str = "gpt-4o-mini"
    top_k: int = 5
    chunk_size: int = 200
    chunk_overlap: int = 20

    class Config:
        env_file = ".env"

settings = Settings()
```

```bash
# .env
OPENAI_API_KEY=sk-...
QDRANT_URL=http://qdrant:6333
EMBEDDING_MODEL=BAAI/bge-m3
LLM_MODEL=gpt-4o-mini
```

### 6.4 Docker Compose Setup

```yaml
# docker-compose.yml
version: "3.8"
services:
  qdrant:
    image: qdrant/qdrant
    ports:
      - "6333:6333"
    volumes:
      - qdrant_data:/qdrant/storage

  app:
    build: .
    ports:
      - "8000:8000"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - QDRANT_URL=http://qdrant:6333
    depends_on:
      - qdrant

volumes:
  qdrant_data:
```

---

## 7. Advanced Pattern: Self-Querying

```python
from haystack.components.routers import ConditionalRouter
from haystack.components.builders import PromptBuilder
from haystack.components.generators import OpenAIGenerator

# ให้ LLM แยก query และ filter ออกมาเอง
QUERY_ANALYSIS_TEMPLATE = """
แยก query ต่อไปนี้เป็น JSON:
- "search_query": คำค้นหาหลัก
- "filters": metadata filter (year, category, author)

Query: {{ query }}

ตอบเฉพาะ JSON เท่านั้น:
"""

# Pipeline
analysis_pipeline = Pipeline()
analysis_pipeline.add_component("prompt", PromptBuilder(template=QUERY_ANALYSIS_TEMPLATE))
analysis_pipeline.add_component("llm", OpenAIGenerator(
    model="gpt-4o-mini",
    generation_kwargs={"response_format": {"type": "json_object"}}
))
analysis_pipeline.connect("prompt.prompt", "llm.prompt")

import json
result = analysis_pipeline.run({"prompt": {"query": "รายงานการเงินปี 2024 ของ Alice"}})
parsed = json.loads(result["llm"]["replies"][0])

# ใช้ parsed filter ใน RAG pipeline
search_result = rag_pipeline.run({
    "embedder": {"text": parsed["search_query"]},
    "retriever": {"filters": parsed["filters"]},
    "prompt_builder": {"question": parsed["search_query"]}
})
```

---

## 8. Checklist สำหรับ Production RAG

```
Retrieval Quality
  ✅ เลือก chunk size ที่เหมาะสม (200-300 words)
  ✅ ใช้ overlap เพื่อไม่ตัดบริบท
  ✅ ใช้ Hybrid Search (BM25 + Semantic)
  ✅ Re-ranking ด้วย Cross-encoder
  ✅ เพิ่ม metadata ที่มีประโยชน์

Generation Quality  
  ✅ Prompt ชัดเจน บอก role และ format ที่ต้องการ
  ✅ ระบุให้ตอบจาก context เท่านั้น
  ✅ มี fallback เมื่อไม่พบข้อมูล
  ✅ ขอให้ cite แหล่งที่มา

Performance
  ✅ Cache embedding ที่ compute แล้ว
  ✅ Batch processing สำหรับ indexing
  ✅ Async สำหรับ concurrent requests
  ✅ DocumentStore ที่ scale ได้ (Qdrant, ES)

Monitoring
  ✅ Log ทุก query และ response
  ✅ Track latency แต่ละ component
  ✅ Monitor token usage
  ✅ Evaluate เป็นระยะด้วย test dataset
```

---

## 9. สรุปภาพรวมทั้ง 5 Phase

```
Phase 1: รู้จัก Haystack
  └─ Architecture, Installation, Hello World

Phase 2: Document Processing  
  └─ Converters, Cleaner, Splitter, Indexing Pipeline

Phase 3: Embedding & Retrieval
  └─ Embedders, BM25/Vector Retriever, Hybrid Search, Ranker

Phase 4: RAG Pipeline ขั้นสูง
  └─ Generators, PromptBuilder, Streaming, Conversational RAG

Phase 5: Custom & Production
  └─ Custom Components, Agents, Evaluation, FastAPI, Docker
```

---

## 10. Resources

| แหล่งข้อมูล | URL |
|---|---|
| Official Docs | https://docs.haystack.deepset.ai |
| GitHub | https://github.com/deepset-ai/haystack |
| Cookbook | https://haystack.deepset.ai/cookbook |
| Discord Community | https://discord.gg/haystack |
| HuggingFace Hub | https://huggingface.co/deepset |

---

> 🎉 **จบ Haystack Deep Dive ครบทั้ง 5 Phase** — ตอนนี้คุณพร้อมสร้าง Production-ready RAG System ด้วย Haystack แล้ว!
