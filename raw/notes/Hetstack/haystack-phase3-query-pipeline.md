# 🌾 Haystack Deep Dive — Phase 3: Retrievers & Query Pipeline

> **เป้าหมาย:** เข้าใจวิธีค้นหาเอกสาร และสร้าง Query Pipeline เพื่อตอบคำถาม

---

## 1. ภาพรวม Query Pipeline

```
[คำถามของ User]
       ↓
[Text Embedder]  ← ถ้าใช้ semantic search
       ↓
[Retriever]  → ค้นหาเอกสารที่เกี่ยวข้อง
       ↓
[Ranker] (optional)  → เรียงลำดับใหม่
       ↓
[Prompt Builder]  → สร้าง prompt
       ↓
[LLM Generator]  → สร้างคำตอบ
       ↓
[คำตอบ]
```

---

## 2. Retrievers ประเภทต่างๆ

### 2.1 InMemoryBM25Retriever (Keyword Search)

ค้นหาด้วย BM25 algorithm (keyword matching) — **ไม่ต้องการ embedding**

```python
from haystack.components.retrievers.in_memory import InMemoryBM25Retriever
from haystack.document_stores.in_memory import InMemoryDocumentStore

store = InMemoryDocumentStore()
# (สมมติว่า index ไว้แล้ว)

retriever = InMemoryBM25Retriever(
    document_store=store,
    top_k=5,            # คืน top 5 เอกสาร
    filters=None,       # กรองด้วย metadata (optional)
    scale_score=True,   # normalize score 0-1
)

result = retriever.run(query="Python คืออะไร")
documents = result["documents"]

for doc in documents:
    print(f"Score: {doc.score:.4f}")
    print(f"Content: {doc.content[:100]}")
    print("---")
```

### BM25 กับ Filter

```python
# ค้นหาเฉพาะเอกสารภาษาไทย
result = retriever.run(
    query="Python คืออะไร",
    filters={
        "operator": "AND",
        "conditions": [
            {"field": "meta.language", "operator": "==", "value": "th"},
            {"field": "meta.department", "operator": "in", "value": ["engineering", "it"]},
        ]
    }
)
```

### Filter Operators ที่รองรับ

| Operator | ความหมาย | ตัวอย่าง |
|----------|---------|---------|
| `==` | เท่ากับ | `"value": "thai"` |
| `!=` | ไม่เท่ากับ | `"value": "english"` |
| `>` | มากกว่า | `"value": 5` |
| `>=` | มากกว่าหรือเท่ากับ | `"value": 2024` |
| `<` | น้อยกว่า | `"value": 100` |
| `<=` | น้อยกว่าหรือเท่ากับ | `"value": 50` |
| `in` | อยู่ใน list | `"value": ["a", "b"]` |
| `not in` | ไม่อยู่ใน list | `"value": ["x", "y"]` |

---

### 2.2 InMemoryEmbeddingRetriever (Semantic Search)

ค้นหาด้วย vector similarity — **ต้องการ embedding**

```python
from haystack.components.retrievers.in_memory import InMemoryEmbeddingRetriever
from haystack.components.embedders import SentenceTransformersTextEmbedder

text_embedder = SentenceTransformersTextEmbedder(
    model="intfloat/multilingual-e5-large"
)
text_embedder.warm_up()

retriever = InMemoryEmbeddingRetriever(
    document_store=store,
    top_k=5,
)

# ต้อง embed query ก่อน แล้วค้น
embed_result = text_embedder.run(text="Python คืออะไร")
retriever_result = retriever.run(query_embedding=embed_result["embedding"])
```

---

### 2.3 Elasticsearch Retrievers

```python
from haystack_integrations.components.retrievers.elasticsearch import (
    ElasticsearchBM25Retriever,
    ElasticsearchEmbeddingRetriever,
)
from haystack_integrations.document_stores.elasticsearch import ElasticsearchDocumentStore

store = ElasticsearchDocumentStore(hosts="http://localhost:9200")

# BM25
bm25_retriever = ElasticsearchBM25Retriever(
    document_store=store,
    top_k=10,
    fuzziness="AUTO",  # รองรับ typo
)

# Embedding (vector)
embedding_retriever = ElasticsearchEmbeddingRetriever(
    document_store=store,
    top_k=10,
)
```

---

### 2.4 Qdrant Retriever

```python
from qdrant_haystack.retrievers import QdrantEmbeddingRetriever

retriever = QdrantEmbeddingRetriever(
    document_store=store,
    top_k=5,
    score_threshold=0.7,  # คืนเฉพาะเอกสารที่ score >= 0.7
)
```

---

## 3. Hybrid Retrieval — รวม BM25 + Semantic

ใช้ทั้งสองวิธีพร้อมกัน แล้วรวมผล = ดีกว่าใช้อย่างเดียว

```python
from haystack import Pipeline
from haystack.components.retrievers.in_memory import (
    InMemoryBM25Retriever,
    InMemoryEmbeddingRetriever,
)
from haystack.components.embedders import SentenceTransformersTextEmbedder
from haystack.components.joiners import DocumentJoiner
from haystack.document_stores.in_memory import InMemoryDocumentStore

store = InMemoryDocumentStore()

pipeline = Pipeline()
pipeline.add_component("text_embedder", SentenceTransformersTextEmbedder(
    model="intfloat/multilingual-e5-large"
))
pipeline.add_component("bm25_retriever", InMemoryBM25Retriever(
    document_store=store, top_k=10
))
pipeline.add_component("embedding_retriever", InMemoryEmbeddingRetriever(
    document_store=store, top_k=10
))
pipeline.add_component("joiner", DocumentJoiner(
    join_mode="reciprocal_rank_fusion",  # RRF — วิธีรวมที่ดีที่สุด
    top_k=5,
))

# เชื่อม
pipeline.connect("text_embedder.embedding", "embedding_retriever.query_embedding")
pipeline.connect("bm25_retriever.documents", "joiner.documents")
pipeline.connect("embedding_retriever.documents", "joiner.documents")

# รัน
result = pipeline.run({
    "text_embedder": {"text": "Python ใช้ทำอะไรได้บ้าง"},
    "bm25_retriever": {"query": "Python ใช้ทำอะไรได้บ้าง"},
})
docs = result["joiner"]["documents"]
```

### DocumentJoiner Modes

| join_mode | ความหมาย |
|-----------|---------|
| `concatenate` | ต่อรายการ (อาจซ้ำ) |
| `merge` | รวม score เฉลี่ย |
| `reciprocal_rank_fusion` | RRF — ดีที่สุดสำหรับ hybrid |

---

## 4. Ranker — เรียงลำดับผลลัพธ์ใหม่

หลังจาก Retrieve แล้ว ใช้ Ranker เรียงใหม่ให้แม่นกว่า

```python
from haystack.components.rankers import TransformersSimilarityRanker

ranker = TransformersSimilarityRanker(
    model="cross-encoder/ms-marco-MiniLM-L-6-v2",
    top_k=3,
)
ranker.warm_up()

result = ranker.run(
    query="Python คืออะไร",
    documents=retrieved_documents,
)
reranked_docs = result["documents"]
```

> **Cross-encoder vs Bi-encoder:**
> - **Bi-encoder** (Embedder): เร็ว แต่อาจไม่แม่น — ใช้ตอน Retrieve
> - **Cross-encoder** (Ranker): ช้ากว่า แต่แม่นมาก — ใช้หลัง Retrieve

---

## 5. Generators — สร้างคำตอบด้วย LLM

### 5.1 OpenAIGenerator

```python
from haystack.components.generators import OpenAIGenerator
import os

os.environ["OPENAI_API_KEY"] = "sk-..."

generator = OpenAIGenerator(
    model="gpt-4o-mini",
    generation_kwargs={
        "temperature": 0.0,    # 0 = deterministic, 1 = creative
        "max_tokens": 500,
        "top_p": 1.0,
    }
)

result = generator.run(prompt="อธิบาย Python ให้ฟัง")
print(result["replies"][0])  # คำตอบ string
print(result["meta"])        # token usage, model info
```

### 5.2 OpenAIChatGenerator (รองรับ conversation history)

```python
from haystack.components.generators.chat import OpenAIChatGenerator
from haystack.dataclasses import ChatMessage

generator = OpenAIChatGenerator(model="gpt-4o-mini")

messages = [
    ChatMessage.from_system("คุณเป็น AI Assistant ที่เชี่ยวชาญ Python"),
    ChatMessage.from_user("Python ดีกว่า Java ยังไง?"),
]

result = generator.run(messages=messages)
reply = result["replies"][0]
print(reply.content)  # คำตอบ
```

### 5.3 HuggingFaceLocalGenerator (ฟรี/Local)

```python
from haystack.components.generators import HuggingFaceLocalGenerator

generator = HuggingFaceLocalGenerator(
    model="google/flan-t5-large",
    task="text2text-generation",
    generation_kwargs={
        "max_new_tokens": 300,
        "temperature": 0.1,
    }
)
generator.warm_up()

result = generator.run(prompt="What is Python?")
```

### 5.4 HuggingFaceAPIGenerator (Inference API)

```python
from haystack.components.generators import HuggingFaceAPIGenerator
import os

os.environ["HF_API_TOKEN"] = "hf_..."

generator = HuggingFaceAPIGenerator(
    api_type="serverless_inference_api",
    api_params={"model": "mistralai/Mistral-7B-Instruct-v0.2"},
)
```

---

## 6. PromptBuilder — สร้าง Prompt

```python
from haystack.components.builders import PromptBuilder

# Template ใช้ Jinja2 syntax
template = """คุณเป็น AI Assistant ที่ตอบคำถามจากเอกสาร

เอกสารที่เกี่ยวข้อง:
{% for doc in documents %}
- {{ doc.content }}
{% endfor %}

คำถาม: {{ question }}

โปรดตอบคำถามโดยอิงจากเอกสารเท่านั้น ถ้าไม่มีข้อมูลให้บอกว่าไม่ทราบ:"""

prompt_builder = PromptBuilder(template=template)

result = prompt_builder.run(
    documents=retrieved_docs,
    question="Python ถูกสร้างโดยใคร?"
)
prompt = result["prompt"]
print(prompt)
```

### Jinja2 Syntax ที่ใช้บ่อย

```jinja2
{# แสดง variable #}
{{ variable }}

{# Loop #}
{% for item in items %}
  {{ item.property }}
{% endfor %}

{# Condition #}
{% if variable %}
  {{ variable }}
{% else %}
  ไม่มีข้อมูล
{% endif %}

{# Filter #}
{{ text | upper }}
{{ number | round(2) }}
{{ list | length }}

{# Loop with index #}
{% for doc in documents %}
  [{{ loop.index }}] {{ doc.content }}
{% endfor %}
```

---

## 7. RAG Pipeline แบบสมบูรณ์ (BM25)

```python
import os
from haystack import Pipeline
from haystack.components.retrievers.in_memory import InMemoryBM25Retriever
from haystack.components.builders import PromptBuilder
from haystack.components.generators import OpenAIGenerator
from haystack.document_stores.in_memory import InMemoryDocumentStore
from haystack import Document

os.environ["OPENAI_API_KEY"] = "sk-..."

# เตรียมข้อมูล
store = InMemoryDocumentStore()
store.write_documents([
    Document(content="Haystack คือ framework สำหรับสร้าง NLP pipeline พัฒนาโดย deepset"),
    Document(content="Haystack รองรับ RAG, Question Answering, และ Conversational AI"),
    Document(content="Haystack 2.x ใช้ pip install haystack-ai"),
    Document(content="Haystack รองรับ OpenAI, HuggingFace, Cohere และ LLM อื่นๆ"),
])

# Template
template = """ตอบคำถามจากบริบทต่อไปนี้ ถ้าไม่ทราบให้บอกว่าไม่ทราบ

บริบท:
{% for doc in documents %}
{{ loop.index }}. {{ doc.content }}
{% endfor %}

คำถาม: {{ question }}

คำตอบ:"""

# สร้าง RAG Pipeline
rag = Pipeline()
rag.add_component("retriever", InMemoryBM25Retriever(document_store=store, top_k=3))
rag.add_component("prompt_builder", PromptBuilder(template=template))
rag.add_component("llm", OpenAIGenerator(model="gpt-4o-mini"))

rag.connect("retriever.documents", "prompt_builder.documents")
rag.connect("prompt_builder.prompt", "llm.prompt")

# ถามคำถาม
def ask(question: str) -> str:
    result = rag.run({
        "retriever": {"query": question},
        "prompt_builder": {"question": question},
    })
    return result["llm"]["replies"][0]

print(ask("Haystack คืออะไร?"))
print(ask("ติดตั้ง Haystack ยังไง?"))
```

---

## 8. RAG Pipeline แบบสมบูรณ์ (Semantic Search)

```python
import os
from haystack import Pipeline
from haystack.components.embedders import (
    SentenceTransformersTextEmbedder,
    SentenceTransformersDocumentEmbedder
)
from haystack.components.retrievers.in_memory import InMemoryEmbeddingRetriever
from haystack.components.builders import PromptBuilder
from haystack.components.generators import OpenAIGenerator
from haystack.components.writers import DocumentWriter
from haystack.document_stores.in_memory import InMemoryDocumentStore
from haystack import Document

os.environ["OPENAI_API_KEY"] = "sk-..."

MODEL = "intfloat/multilingual-e5-large"

# --- Indexing ---
store = InMemoryDocumentStore()

docs = [
    Document(content="Haystack คือ framework สำหรับสร้าง NLP pipeline"),
    Document(content="Haystack รองรับ Retrieval-Augmented Generation หรือ RAG"),
    Document(content="ใช้ pip install haystack-ai เพื่อติดตั้ง Haystack"),
]

doc_embedder = SentenceTransformersDocumentEmbedder(model=MODEL)
doc_embedder.warm_up()
embedded = doc_embedder.run(documents=docs)["documents"]

writer = DocumentWriter(document_store=store)
writer.run(documents=embedded)

# --- Query Pipeline ---
rag = Pipeline()
rag.add_component("text_embedder", SentenceTransformersTextEmbedder(model=MODEL))
rag.add_component("retriever", InMemoryEmbeddingRetriever(document_store=store, top_k=3))
rag.add_component("prompt_builder", PromptBuilder(template="""
{% for doc in documents %}{{ doc.content }}
{% endfor %}
คำถาม: {{ question }}
ตอบ:"""))
rag.add_component("llm", OpenAIGenerator(model="gpt-4o-mini"))

rag.connect("text_embedder.embedding", "retriever.query_embedding")
rag.connect("retriever.documents", "prompt_builder.documents")
rag.connect("prompt_builder.prompt", "llm.prompt")

# รัน
result = rag.run({
    "text_embedder": {"text": "Haystack ติดตั้งยังไง"},
    "prompt_builder": {"question": "Haystack ติดตั้งยังไง"},
})
print(result["llm"]["replies"][0])
```

---

## 9. Metadata Filtering ใน Query

```python
# กรองเฉพาะเอกสารจาก department engineering
result = pipeline.run({
    "retriever": {
        "query": "deployment process",
        "filters": {
            "operator": "AND",
            "conditions": [
                {"field": "meta.department", "operator": "==", "value": "engineering"},
                {"field": "meta.year", "operator": ">=", "value": 2023},
            ]
        },
        "top_k": 5,
    },
    "prompt_builder": {"question": "What is the deployment process?"}
})
```

---

## 10. Debug Pipeline

```python
# ดู Pipeline diagram
pipeline.show()

# ดู schema
pipeline.get_component("retriever")

# Dry run — ไม่รันจริง แต่ validate
pipeline.run(
    {"retriever": {"query": "test"}},
    include_outputs_from={"retriever", "prompt_builder"}  # ดู intermediate output
)
```

---

## 11. สรุป Phase 3

```
✅ BM25Retriever — keyword search ไม่ต้อง embedding
✅ EmbeddingRetriever — semantic search ต้องมี embedding
✅ Hybrid = BM25 + Embedding ผ่าน DocumentJoiner (RRF)
✅ Ranker (Cross-encoder) เรียงผลใหม่ให้แม่นขึ้น
✅ PromptBuilder ใช้ Jinja2 template สร้าง prompt
✅ Generator รองรับ OpenAI, HuggingFace, Local
✅ RAG Pipeline = Retriever → PromptBuilder → Generator
✅ Metadata filtering กรองเอกสารก่อนค้นหา
```

---

**➡️ ต่อไป: Phase 4 — Advanced Features: Agents, Routers, Custom Components**
