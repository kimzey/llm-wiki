# 🌾 Haystack Deep Dive — Phase 5: Integrations & Real-world Examples

> **เป้าหมาย:** ใช้ Haystack กับ tools จริง และสร้างระบบ production-ready

---

## 1. Haystack Integrations ทั้งหมด

Haystack มี integration ecosystem กว้างมาก ติดตั้งแยกได้:

```bash
# LLM Providers
pip install haystack-ai openai          # OpenAI
pip install amazon-bedrock-haystack     # AWS Bedrock (Claude, Titan)
pip install cohere-haystack             # Cohere
pip install google-vertex-haystack      # Google Vertex AI (Gemini)
pip install mistral-haystack            # Mistral AI
pip install ollama-haystack             # Ollama (local LLM)

# Vector Databases
pip install qdrant-haystack             # Qdrant
pip install weaviate-haystack           # Weaviate
pip install pinecone-haystack           # Pinecone
pip install opensearch-haystack         # OpenSearch
pip install elasticsearch-haystack      # Elasticsearch
pip install pgvector-haystack           # PostgreSQL + pgvector
pip install mongodb-atlas-haystack      # MongoDB Atlas

# Embedding Models
pip install sentence-transformers       # HuggingFace ST
pip install fastembed-haystack          # FastEmbed (เร็ว)

# Document Converters
pip install pypdf                       # PDF
pip install python-docx                 # DOCX
pip install unstructured-haystack       # ทุกประเภทไฟล์
pip install jina-haystack               # Jina Reader
```

---

## 2. Ollama Integration — ใช้ LLM Local ฟรี

Ollama ให้รัน LLM บนเครื่องตัวเองได้

```bash
# ติดตั้ง Ollama
curl -fsSL https://ollama.ai/install.sh | sh

# ดึง model
ollama pull llama3.2
ollama pull mistral
ollama pull qwen2.5  # ดีสำหรับภาษาไทย
```

```python
from haystack_integrations.components.generators.ollama import OllamaGenerator

generator = OllamaGenerator(
    model="llama3.2",
    url="http://localhost:11434",
    generation_kwargs={
        "temperature": 0.1,
        "num_predict": 500,
    }
)

result = generator.run(prompt="อธิบาย Python ให้ฟัง")
print(result["replies"][0])
```

### Ollama Chat Generator

```python
from haystack_integrations.components.generators.ollama import OllamaChatGenerator
from haystack.dataclasses import ChatMessage

generator = OllamaChatGenerator(model="llama3.2")

messages = [
    ChatMessage.from_system("คุณเป็น AI ผู้ช่วยภาษาไทย"),
    ChatMessage.from_user("สวัสดี คุณช่วยอะไรได้บ้าง?"),
]

result = generator.run(messages=messages)
print(result["replies"][0].content)
```

---

## 3. AWS Bedrock Integration

```python
from haystack_integrations.components.generators.amazon_bedrock import (
    AmazonBedrockGenerator,
    AmazonBedrockChatGenerator,
)
import boto3

# ต้อง configure AWS credentials
# export AWS_ACCESS_KEY_ID=...
# export AWS_SECRET_ACCESS_KEY=...
# export AWS_REGION=us-east-1

generator = AmazonBedrockGenerator(
    model="anthropic.claude-3-sonnet-20240229-v1:0",
    max_length=1000,
)

# หรือ Chat generator
chat_gen = AmazonBedrockChatGenerator(
    model="anthropic.claude-3-haiku-20240307-v1:0",
)
```

---

## 4. Google Gemini Integration

```python
from haystack_integrations.components.generators.google_vertex import (
    VertexAIGeminiGenerator,
    VertexAIGeminiChatGenerator,
)
import os

os.environ["GOOGLE_CLOUD_PROJECT"] = "your-project-id"

generator = VertexAIGeminiGenerator(
    model="gemini-1.5-flash",
    project="your-project-id",
    location="us-central1",
)

result = generator.run(parts=["อธิบาย Haystack ให้ฟัง"])
```

---

## 5. Unstructured.io — แปลงไฟล์ทุกประเภท

```python
from haystack_integrations.components.converters.unstructured import (
    UnstructuredFileConverter,
)

# แปลงได้ทุกประเภท: PDF, DOCX, PPTX, XLS, HTML, Images, ...
converter = UnstructuredFileConverter(
    api_url="https://api.unstructured.io/general/v0/general",
    api_key=os.getenv("UNSTRUCTURED_API_KEY"),
)

result = converter.run(paths=[
    "presentation.pptx",
    "spreadsheet.xlsx", 
    "scanned_document.png",  # OCR!
])
documents = result["documents"]
```

---

## 6. pgvector — ใช้ PostgreSQL เป็น Vector Store

```python
from haystack_integrations.document_stores.pgvector import PgvectorDocumentStore
from haystack_integrations.components.retrievers.pgvector import PgvectorEmbeddingRetriever

# connection string
CONNECTION_STRING = "postgresql://user:password@localhost:5432/haystack_db"

store = PgvectorDocumentStore(
    connection_string=CONNECTION_STRING,
    table_name="documents",
    embedding_dimension=1536,
    vector_function="cosine_similarity",
    recreate_table=False,
)

retriever = PgvectorEmbeddingRetriever(
    document_store=store,
    top_k=5,
)

# ติดตั้ง: pip install pgvector-haystack
# ใน PostgreSQL: CREATE EXTENSION vector;
```

---

## 7. Real-world Example 1: Company Knowledge Base

สร้างระบบถามตอบจากเอกสารบริษัท

```python
import os
from pathlib import Path
from haystack import Pipeline, Document
from haystack.components.converters import PyPDFToDocument, TextFileToDocument
from haystack.components.preprocessors import DocumentCleaner, DocumentSplitter
from haystack.components.routers import FileTypeRouter
from haystack.components.joiners import DocumentJoiner
from haystack.components.embedders import OpenAIDocumentEmbedder, OpenAITextEmbedder
from haystack.components.retrievers.in_memory import InMemoryEmbeddingRetriever
from haystack.components.builders import PromptBuilder
from haystack.components.generators import OpenAIGenerator
from haystack.components.writers import DocumentWriter
from haystack.document_stores.in_memory import InMemoryDocumentStore

os.environ["OPENAI_API_KEY"] = os.getenv("OPENAI_API_KEY", "")
EMBED_MODEL = "text-embedding-3-small"

# ========== INDEXING PIPELINE ==========
store = InMemoryDocumentStore()

indexing = Pipeline()
indexing.add_component("file_router", FileTypeRouter(
    mime_types=["application/pdf", "text/plain"]
))
indexing.add_component("pdf_converter", PyPDFToDocument())
indexing.add_component("txt_converter", TextFileToDocument())
indexing.add_component("joiner", DocumentJoiner())
indexing.add_component("cleaner", DocumentCleaner())
indexing.add_component("splitter", DocumentSplitter(
    split_by="word",
    split_length=300,
    split_overlap=50,
))
indexing.add_component("embedder", OpenAIDocumentEmbedder(model=EMBED_MODEL))
indexing.add_component("writer", DocumentWriter(document_store=store, policy="overwrite"))

indexing.connect("file_router.application/pdf", "pdf_converter.sources")
indexing.connect("file_router.text/plain", "txt_converter.sources")
indexing.connect("pdf_converter.documents", "joiner.documents")
indexing.connect("txt_converter.documents", "joiner.documents")
indexing.connect("joiner.documents", "cleaner.documents")
indexing.connect("cleaner.documents", "splitter.documents")
indexing.connect("splitter.documents", "embedder.documents")
indexing.connect("embedder.documents", "writer.documents")

# Index ไฟล์ทั้งหมด
all_files = list(Path("./knowledge_base").iterdir())
indexing.run({"file_router": {"sources": all_files}})

print(f"✅ Indexed {store.count_documents()} chunks")

# ========== QUERY PIPELINE ==========
template = """คุณเป็น AI Assistant ของบริษัท ตอบคำถามโดยอิงจากเอกสารบริษัทเท่านั้น
หากไม่มีข้อมูลในเอกสาร ให้บอกว่า "ไม่พบข้อมูลในเอกสาร กรุณาติดต่อ HR หรือผู้รับผิดชอบ"
ตอบเป็นภาษาเดียวกับคำถาม

เอกสารอ้างอิง:
{% for doc in documents %}
[{{ loop.index }}] {{ doc.content }}
   (ที่มา: {{ doc.meta.get('file_path', 'unknown') }})
{% endfor %}

คำถาม: {{ question }}
คำตอบ:"""

rag = Pipeline()
rag.add_component("text_embedder", OpenAITextEmbedder(model=EMBED_MODEL))
rag.add_component("retriever", InMemoryEmbeddingRetriever(document_store=store, top_k=5))
rag.add_component("prompt_builder", PromptBuilder(template=template))
rag.add_component("llm", OpenAIGenerator(
    model="gpt-4o-mini",
    generation_kwargs={"temperature": 0.0}
))

rag.connect("text_embedder.embedding", "retriever.query_embedding")
rag.connect("retriever.documents", "prompt_builder.documents")
rag.connect("prompt_builder.prompt", "llm.prompt")

def ask_knowledge_base(question: str) -> dict:
    result = rag.run({
        "text_embedder": {"text": question},
        "prompt_builder": {"question": question},
    })
    return {
        "answer": result["llm"]["replies"][0],
        "sources": [d.meta.get("file_path") for d in result["retriever"]["documents"]],
    }

# ทดสอบ
response = ask_knowledge_base("วันหยุดประจำปีของบริษัทมีกี่วัน?")
print(f"คำตอบ: {response['answer']}")
print(f"แหล่งที่มา: {response['sources']}")
```

---

## 8. Real-world Example 2: Multi-document Summarizer

```python
from haystack import Pipeline
from haystack.components.converters import PyPDFToDocument
from haystack.components.preprocessors import DocumentSplitter
from haystack.components.builders import PromptBuilder
from haystack.components.generators import OpenAIGenerator
from haystack.components.joiners import DocumentJoiner

def summarize_documents(pdf_paths: list, focus: str = "") -> str:
    """สรุปเอกสาร PDF หลายไฟล์"""
    
    focus_instruction = f"โดยเน้นที่: {focus}" if focus else ""
    
    template = f"""สรุปเอกสารต่อไปนี้ {focus_instruction}

เนื้อหาเอกสาร:
{{% for doc in documents %}}
--- ส่วนที่ {{{{ loop.index }}}} ---
{{{{ doc.content }}}}
{{% endfor %}}

สรุป:"""

    pipeline = Pipeline()
    pipeline.add_component("converter", PyPDFToDocument())
    pipeline.add_component("splitter", DocumentSplitter(
        split_by="passage", split_length=5, split_overlap=1
    ))
    pipeline.add_component("prompt_builder", PromptBuilder(template=template))
    pipeline.add_component("llm", OpenAIGenerator(
        model="gpt-4o",
        generation_kwargs={"max_tokens": 2000, "temperature": 0.3}
    ))

    pipeline.connect("converter.documents", "splitter.documents")
    pipeline.connect("splitter.documents", "prompt_builder.documents")
    pipeline.connect("prompt_builder.prompt", "llm.prompt")

    result = pipeline.run({"converter": {"sources": pdf_paths}})
    return result["llm"]["replies"][0]

# ใช้งาน
summary = summarize_documents(
    pdf_paths=["q1_report.pdf", "q2_report.pdf"],
    focus="ผลประกอบการและรายได้"
)
print(summary)
```

---

## 9. Real-world Example 3: REST API ด้วย FastAPI

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from haystack import Pipeline
# ... (import อื่นๆ)

app = FastAPI(title="Haystack RAG API")

# สร้าง pipeline ครั้งเดียวตอน startup
rag_pipeline: Pipeline = None

@app.on_event("startup")
async def startup():
    global rag_pipeline
    rag_pipeline = create_rag_pipeline()  # function สร้าง pipeline
    print("✅ Pipeline ready")

class QueryRequest(BaseModel):
    question: str
    top_k: int = 5
    filters: dict = None

class QueryResponse(BaseModel):
    answer: str
    sources: list[str]
    confidence: float

@app.post("/ask", response_model=QueryResponse)
async def ask(request: QueryRequest):
    try:
        result = rag_pipeline.run({
            "text_embedder": {"text": request.question},
            "retriever": {"top_k": request.top_k},
            "prompt_builder": {"question": request.question},
        })
        
        answer = result["llm"]["replies"][0]
        docs = result["retriever"]["documents"]
        
        return QueryResponse(
            answer=answer,
            sources=[d.meta.get("source", "unknown") for d in docs],
            confidence=docs[0].score if docs else 0.0,
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health():
    return {"status": "ok", "documents": store.count_documents()}

# รัน: uvicorn main:app --reload
```

---

## 10. Real-world Example 4: Slack Bot

```python
from slack_bolt import App
from slack_bolt.adapter.socket_mode import SocketModeHandler
import os

app = App(token=os.getenv("SLACK_BOT_TOKEN"))

# สร้าง RAG pipeline
rag = create_rag_pipeline()

@app.message("ถาม:")
def handle_question(message, say):
    """ตอบคำถามใน Slack"""
    question = message["text"].replace("ถาม:", "").strip()
    
    try:
        result = rag.run({
            "text_embedder": {"text": question},
            "prompt_builder": {"question": question},
        })
        answer = result["llm"]["replies"][0]
        say(f"🤖 *คำตอบ:*\n{answer}")
    except Exception as e:
        say(f"❌ เกิดข้อผิดพลาด: {str(e)}")

if __name__ == "__main__":
    handler = SocketModeHandler(app, os.getenv("SLACK_APP_TOKEN"))
    handler.start()
```

---

## 11. Evaluation — วัดความแม่นของ RAG

```python
from haystack.components.evaluators import (
    ContextRelevanceEvaluator,
    FaithfulnessEvaluator,
    SASEvaluator,
)

# Context Relevance — เอกสารที่ดึงมาเกี่ยวข้องกับคำถามไหม
context_evaluator = ContextRelevanceEvaluator()

# Faithfulness — คำตอบอ้างอิงจากเอกสารหรือแต่งขึ้นมาเอง
faithfulness_evaluator = FaithfulnessEvaluator()

# ประเมินผล
questions = ["Python คืออะไร?", "Haystack รองรับ LLM อะไรบ้าง?"]
answers = []
contexts = []

for q in questions:
    result = rag.run({
        "text_embedder": {"text": q},
        "prompt_builder": {"question": q},
    })
    answers.append(result["llm"]["replies"][0])
    contexts.append([d.content for d in result["retriever"]["documents"]])

# วัด relevance
rel_result = context_evaluator.run(
    questions=questions,
    contexts=contexts,
)
print(f"Context Relevance: {rel_result['score']:.3f}")

# วัด faithfulness
faith_result = faithfulness_evaluator.run(
    questions=questions,
    contexts=contexts,
    responses=answers,
)
print(f"Faithfulness: {faith_result['score']:.3f}")
```

---

## 12. Quick Reference — เลือก Component ยังไง?

### เลือก Document Store

```
ต้องการอะไร?
├── Dev/Testing → InMemoryDocumentStore
├── Keyword search (production) → Elasticsearch / OpenSearch
├── Vector search (production) → Qdrant / Weaviate / Pinecone
├── ใช้ PostgreSQL อยู่แล้ว → pgvector
└── Managed cloud → Pinecone / Weaviate Cloud
```

### เลือก Retriever

```
ต้องการอะไร?
├── Keyword exact match → BM25Retriever
├── Semantic meaning → EmbeddingRetriever
└── ทั้งสองอย่าง → Hybrid (BM25 + Embedding + DocumentJoiner RRF)
```

### เลือก LLM

```
งบประมาณ?
├── ฟรี/Local → Ollama (llama, mistral, qwen)
├── เสียเงิน ต้องการดี → OpenAI GPT-4o
├── เสียเงิน ต้องการถูก → GPT-4o-mini / Claude Haiku
└── Enterprise → AWS Bedrock / Google Vertex AI
```

### เลือก Embedding Model

```
ภาษาที่ใช้?
├── English เท่านั้น → all-mpnet-base-v2 (local) / text-embedding-3-small (API)
├── หลายภาษา / ไทย → multilingual-e5-large (local) / text-embedding-3-small (API)
└── ต้องการเร็วมาก → BAAI/bge-small-en-v1.5
```

---

## 13. Cheat Sheet สรุปทั้งหมด

```python
# ===== QUICK SETUP =====
pip install haystack-ai openai

from haystack import Pipeline, Document
from haystack.document_stores.in_memory import InMemoryDocumentStore
from haystack.components.retrievers.in_memory import InMemoryBM25Retriever
from haystack.components.builders import PromptBuilder
from haystack.components.generators import OpenAIGenerator

# 1. Store + Documents
store = InMemoryDocumentStore()
store.write_documents([Document(content="...")])

# 2. Pipeline
p = Pipeline()
p.add_component("retriever", InMemoryBM25Retriever(document_store=store))
p.add_component("prompt", PromptBuilder(template="{{ documents }} {{ question }}"))
p.add_component("llm", OpenAIGenerator(model="gpt-4o-mini"))

p.connect("retriever.documents", "prompt.documents")
p.connect("prompt.prompt", "llm.prompt")

# 3. Run
result = p.run({
    "retriever": {"query": "คำถาม"},
    "prompt": {"question": "คำถาม"}
})
print(result["llm"]["replies"][0])
```

---

## 14. สรุป Phase 5 และ Series ทั้งหมด

```
Phase 1: Core Concepts
  ✅ Haystack = NLP Pipeline Framework
  ✅ Document, Pipeline, Component คืออะไร

Phase 2: Indexing
  ✅ Converters (PDF, DOCX, HTML, TXT)
  ✅ Cleaner + Splitter (chunk)
  ✅ Embedders (OpenAI, SentenceTransformers)
  ✅ Document Stores (InMemory, Qdrant, ES)

Phase 3: Query Pipeline
  ✅ BM25 Retriever (keyword)
  ✅ Embedding Retriever (semantic)
  ✅ Hybrid Search + DocumentJoiner
  ✅ Ranker + PromptBuilder + Generator

Phase 4: Advanced
  ✅ Custom Components
  ✅ Routers (Metadata, Conditional, FileType)
  ✅ Agent Systems + Tool Use
  ✅ Conversational RAG + Memory

Phase 5: Production
  ✅ Integrations (Ollama, Bedrock, Gemini)
  ✅ Real-world Examples (API, Slack Bot)
  ✅ Evaluation (Relevance, Faithfulness)
  ✅ Component Selection Guide
```

---

## Resources

| ที่ | Link |
|-----|------|
| 📚 Official Docs | https://docs.haystack.deepset.ai |
| 🐙 GitHub | https://github.com/deepset-ai/haystack |
| 🎓 Tutorials | https://haystack.deepset.ai/tutorials |
| 💬 Discord | https://discord.gg/haystack |
| 🔌 Integrations | https://haystack.deepset.ai/integrations |
