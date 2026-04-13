# 🌾 Haystack Deep Dive — Phase 4: Advanced Features

> **เป้าหมาย:** สร้าง Custom Component, Agent Systems, Routers และ Production-ready Pipeline

---

## 1. Custom Components

Haystack ให้สร้าง Component เองได้ง่ายมาก

### 1.1 โครงสร้างพื้นฐาน

```python
from haystack import component
from haystack import Document
from typing import List, Optional

@component
class MyCustomComponent:
    """Custom component ของฉัน"""
    
    def __init__(self, prefix: str = "📄"):
        """ตั้งค่าตอนสร้าง component"""
        self.prefix = prefix
    
    @component.output_types(documents=List[Document])
    def run(self, documents: List[Document]) -> dict:
        """
        ทำงานหลัก — Haystack จะเรียก method นี้
        
        Args:
            documents: input จาก component ก่อนหน้า
            
        Returns:
            dict ที่มี key ตรงกับ output_types
        """
        processed = []
        for doc in documents:
            new_doc = Document(
                content=f"{self.prefix} {doc.content}",
                meta=doc.meta,
                id=doc.id,
            )
            processed.append(new_doc)
        
        return {"documents": processed}
```

### 1.2 Multiple Inputs และ Outputs

```python
from haystack import component
from typing import List, Optional

@component
class DocumentFilter:
    """กรองเอกสารตาม score threshold"""
    
    def __init__(self, threshold: float = 0.5):
        self.threshold = threshold
    
    @component.output_types(
        passed=List[Document],    # เอกสารที่ผ่าน
        filtered=List[Document],  # เอกสารที่ไม่ผ่าน
    )
    def run(
        self,
        documents: List[Document],
        threshold: Optional[float] = None,  # override threshold แบบ runtime
    ) -> dict:
        cutoff = threshold or self.threshold
        passed = [d for d in documents if (d.score or 0) >= cutoff]
        filtered = [d for d in documents if (d.score or 0) < cutoff]
        
        return {"passed": passed, "filtered": filtered}


# ใน Pipeline
pipeline.add_component("filter", DocumentFilter(threshold=0.7))
pipeline.connect("retriever.documents", "filter.documents")
pipeline.connect("filter.passed", "prompt_builder.documents")
```

### 1.3 Stateful Component

```python
from haystack import component, default_from_dict, default_to_dict
from typing import List

@component
class ConversationHistory:
    """เก็บ history การสนทนา"""
    
    def __init__(self, max_history: int = 10):
        self.max_history = max_history
        self.history: List[dict] = []
    
    @component.output_types(history=List[dict], current=str)
    def run(self, user_message: str) -> dict:
        self.history.append({"role": "user", "content": user_message})
        
        # ตัด history ถ้าเกิน max
        if len(self.history) > self.max_history:
            self.history = self.history[-self.max_history:]
        
        return {
            "history": self.history.copy(),
            "current": user_message
        }
    
    def reset(self):
        self.history = []
    
    # serialize/deserialize สำหรับบันทึก pipeline
    def to_dict(self):
        return default_to_dict(self, max_history=self.max_history)
    
    @classmethod
    def from_dict(cls, data):
        return default_from_dict(cls, data)
```

---

## 2. Routers — แยกทาง Pipeline

### 2.1 MetadataRouter

```python
from haystack.components.routers import MetadataRouter

router = MetadataRouter(
    rules={
        "thai": {
            "field": "meta.language",
            "operator": "==",
            "value": "th"
        },
        "english": {
            "field": "meta.language",
            "operator": "==",
            "value": "en"
        },
    }
)

# Pipeline แบบ branch
pipeline.add_component("router", MetadataRouter(...))
pipeline.add_component("thai_embedder", ThaiEmbedder())
pipeline.add_component("en_embedder", EnglishEmbedder())

pipeline.connect("router.thai", "thai_embedder.documents")
pipeline.connect("router.english", "en_embedder.documents")
```

### 2.2 ConditionalRouter (ใช้ expression)

```python
from haystack.components.routers import ConditionalRouter

router = ConditionalRouter(
    routes=[
        {
            "condition": "{{ documents | length > 0 }}",  # มีเอกสาร
            "output": "{{ documents }}",
            "output_name": "has_documents",
            "output_type": List[Document],
        },
        {
            "condition": "{{ documents | length == 0 }}",  # ไม่มีเอกสาร
            "output": "{{ [] }}",
            "output_name": "no_documents",
            "output_type": List[Document],
        },
    ]
)
```

### 2.3 FileTypeRouter

```python
from haystack.components.routers import FileTypeRouter

router = FileTypeRouter(
    mime_types=["application/pdf", "text/plain", "text/html"]
)

pipeline.add_component("router", FileTypeRouter(
    mime_types=["application/pdf", "text/plain"]
))
pipeline.add_component("pdf_converter", PyPDFToDocument())
pipeline.add_component("text_converter", TextFileToDocument())

pipeline.connect("router.application/pdf", "pdf_converter.sources")
pipeline.connect("router.text/plain", "text_converter.sources")
```

---

## 3. Agent Systems

### 3.1 OpenAI Function Calling / Tool Use

```python
from haystack.components.generators.chat import OpenAIChatGenerator
from haystack.components.tools import ToolInvoker
from haystack.dataclasses import ChatMessage, Tool

# กำหนด Tool ที่ Agent ใช้ได้
def search_documents(query: str) -> str:
    """ค้นหาเอกสารจาก knowledge base"""
    # เรียก retriever จริง
    results = retriever.run(query=query)["documents"]
    return "\n".join([d.content for d in results[:3]])

def get_weather(city: str) -> str:
    """ดูอากาศของเมือง"""
    return f"อากาศใน {city}: 28°C มีเมฆบางส่วน"

# สร้าง Tool definitions
tools = [
    Tool(
        name="search_documents",
        description="ค้นหาข้อมูลจาก knowledge base ของบริษัท",
        parameters={
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "คำค้นหา"}
            },
            "required": ["query"]
        },
        function=search_documents,
    ),
    Tool(
        name="get_weather",
        description="ดูสภาพอากาศของเมืองที่ระบุ",
        parameters={
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "ชื่อเมือง"}
            },
            "required": ["city"]
        },
        function=get_weather,
    ),
]

# สร้าง Agent Pipeline
from haystack import Pipeline

agent_pipeline = Pipeline()
agent_pipeline.add_component(
    "llm",
    OpenAIChatGenerator(model="gpt-4o-mini", tools=tools)
)
agent_pipeline.add_component("tool_invoker", ToolInvoker(tools=tools))

# loop: llm → tool_invoker → llm (ถ้ายังมี tool call)
agent_pipeline.connect("llm.replies", "tool_invoker.messages")
agent_pipeline.connect("tool_invoker.messages", "llm.messages")
```

### 3.2 Agent Loop แบบง่าย (Manual)

```python
from haystack.components.generators.chat import OpenAIChatGenerator
from haystack.dataclasses import ChatMessage
import json

def run_agent(user_question: str, tools: list, max_steps: int = 5):
    """Agent loop อย่างง่าย"""
    
    llm = OpenAIChatGenerator(
        model="gpt-4o-mini",
        tools=tools,
        generation_kwargs={"temperature": 0}
    )
    
    messages = [
        ChatMessage.from_system("คุณเป็น AI Assistant ที่ช่วยตอบคำถาม"),
        ChatMessage.from_user(user_question),
    ]
    
    for step in range(max_steps):
        result = llm.run(messages=messages)
        reply = result["replies"][0]
        messages.append(reply)
        
        # ถ้า LLM ไม่ได้เรียก tool → จบ
        if not reply.tool_calls:
            return reply.content
        
        # เรียก tool แต่ละตัว
        for tool_call in reply.tool_calls:
            tool_name = tool_call.tool_name
            tool_args = tool_call.arguments
            
            # หา function และเรียก
            tool_func = {t.name: t.function for t in tools}[tool_name]
            tool_result = tool_func(**tool_args)
            
            # เพิ่มผลลัพธ์กลับเข้า messages
            messages.append(ChatMessage.from_tool(
                tool_result=str(tool_result),
                origin=tool_call,
            ))
    
    return "ถึง max steps แล้ว"

# ใช้งาน
answer = run_agent(
    "หาข้อมูลเรื่อง Python แล้วสรุปให้หน่อย",
    tools=tools
)
print(answer)
```

---

## 4. OutputAdapter — แปลง Output Format

```python
from haystack.components.converters import OutputAdapter

# แปลง list of Document → string
adapter = OutputAdapter(
    template="{{ documents | map(attribute='content') | join('\n') }}",
    output_type=str,
)

pipeline.add_component("adapter", OutputAdapter(
    template="{{ replies[0] }}",  # เอาแค่ reply แรก
    output_type=str,
))
pipeline.connect("llm.replies", "adapter.replies")
```

---

## 5. AnswerBuilder — สร้าง Structured Answer

```python
from haystack.components.builders import AnswerBuilder

pipeline.add_component("answer_builder", AnswerBuilder())

pipeline.connect("llm.replies", "answer_builder.replies")
pipeline.connect("retriever.documents", "answer_builder.documents")
pipeline.connect("llm.meta", "answer_builder.meta")

# Output จะเป็น GeneratedAnswer object
result = pipeline.run({...})
answer = result["answer_builder"]["answers"][0]

print(answer.data)        # ข้อความคำตอบ
print(answer.documents)   # เอกสารที่ใช้อ้างอิง
print(answer.meta)        # metadata
```

---

## 6. Conversational RAG (Chat with Memory)

```python
from haystack import Pipeline
from haystack.components.generators.chat import OpenAIChatGenerator
from haystack.components.builders import DynamicChatPromptBuilder
from haystack.components.retrievers.in_memory import InMemoryBM25Retriever
from haystack.dataclasses import ChatMessage
from haystack.document_stores.in_memory import InMemoryDocumentStore

store = InMemoryDocumentStore()
# (index documents แล้ว)

pipeline = Pipeline()
pipeline.add_component("retriever", InMemoryBM25Retriever(document_store=store))
pipeline.add_component("prompt_builder", DynamicChatPromptBuilder())
pipeline.add_component("llm", OpenAIChatGenerator(model="gpt-4o-mini"))

pipeline.connect("retriever.documents", "prompt_builder.documents")
pipeline.connect("prompt_builder.prompt", "llm.messages")

# เก็บ history
chat_history = []

def chat(question: str) -> str:
    system_msg = ChatMessage.from_system(
        "คุณเป็น Assistant ตอบคำถามจากเอกสาร ใช้ภาษาไทย"
    )
    user_msg = ChatMessage.from_user(question)
    
    result = pipeline.run({
        "retriever": {"query": question},
        "prompt_builder": {
            "template": [
                system_msg,
                *chat_history,  # ใส่ history
                ChatMessage.from_user("""
บริบท: {% for doc in documents %}{{ doc.content }}{% endfor %}

{{ question }}"""),
            ],
            "template_variables": {"question": question}
        }
    })
    
    reply = result["llm"]["replies"][0]
    
    # อัพเดต history
    chat_history.append(ChatMessage.from_user(question))
    chat_history.append(reply)
    
    return reply.content

# ทดสอบ
print(chat("Haystack คืออะไร?"))
print(chat("แล้วติดตั้งยังไง?"))  # จะรู้บริบทจากข้อความก่อน
```

---

## 7. Pipeline Branching & Merging

```python
from haystack import Pipeline
from haystack.components.joiners import BranchJoiner

# สร้าง pipeline ที่แยกสาย แล้วรวมกลับ
pipeline = Pipeline()

# สาย A
pipeline.add_component("retriever_a", BM25Retriever(store_a))
# สาย B  
pipeline.add_component("retriever_b", BM25Retriever(store_b))

# รวมกลับ
pipeline.add_component("joiner", BranchJoiner(type_=List[Document]))
pipeline.add_component("prompt_builder", PromptBuilder(template=...))

pipeline.connect("retriever_a.documents", "joiner")
pipeline.connect("retriever_b.documents", "joiner")
pipeline.connect("joiner", "prompt_builder.documents")
```

---

## 8. Serialization & Pipeline Management

### บันทึก Pipeline

```python
import yaml

# บันทึกเป็น YAML
with open("my_pipeline.yaml", "w") as f:
    pipeline.dump(f)

# โหลดกลับ
with open("my_pipeline.yaml", "r") as f:
    loaded = Pipeline.load(f)
```

### Pipeline YAML ตัวอย่าง

```yaml
components:
  retriever:
    type: haystack.components.retrievers.in_memory.bm25_retriever.InMemoryBM25Retriever
    init_parameters:
      top_k: 5
  prompt_builder:
    type: haystack.components.builders.prompt_builder.PromptBuilder
    init_parameters:
      template: "{{ question }}"
  llm:
    type: haystack.components.generators.openai.OpenAIGenerator
    init_parameters:
      model: gpt-4o-mini

connections:
  - sender: retriever.documents
    receiver: prompt_builder.documents
  - sender: prompt_builder.prompt
    receiver: llm.prompt

metadata:
  version: "2.0"
```

---

## 9. Error Handling

```python
from haystack import Pipeline
from haystack.core.errors import (
    PipelineError,
    PipelineConnectError,
    PipelineRuntimeError,
)

try:
    result = pipeline.run({"retriever": {"query": "test"}})
except PipelineRuntimeError as e:
    print(f"Pipeline error: {e}")
    # log และ fallback
except Exception as e:
    print(f"Unexpected error: {e}")

# Graceful fallback component
@component
class FallbackComponent:
    @component.output_types(answer=str)
    def run(self, documents: List[Document]) -> dict:
        if not documents:
            return {"answer": "ขออภัย ไม่พบข้อมูลที่เกี่ยวข้อง"}
        return {"answer": "processing..."}
```

---

## 10. Tracing & Monitoring

```python
# ใช้ Haystack Tracing (OpenTelemetry compatible)
from haystack.tracing import tracer
from haystack.tracing.tracer import ProxyTracer

# Enable tracing
import haystack

haystack.tracing.enable_tracing(
    haystack.tracing.OpenTelemetryTracer()
)

# หรือใช้ Langfuse
from haystack.components.others import HaystackTracer

# Log pipeline runs
pipeline.run(
    {"retriever": {"query": "test"}},
    debug=True  # print debug info
)
```

---

## 11. Production Tips

### ✅ Do's

```python
# 1. ใช้ environment variables สำหรับ API keys
import os
from dotenv import load_dotenv
load_dotenv()
api_key = os.getenv("OPENAI_API_KEY")

# 2. ตั้ง timeout
generator = OpenAIGenerator(
    model="gpt-4o-mini",
    timeout=30,  # seconds
    max_retries=3,
)

# 3. ใช้ batch processing สำหรับ indexing ขนาดใหญ่
doc_embedder = SentenceTransformersDocumentEmbedder(
    model=MODEL,
    batch_size=64,  # process 64 docs พร้อมกัน
    progress_bar=True,
)

# 4. Cache embedding results
from haystack.components.caching import EmbeddingCachingRetriever

# 5. ใช้ async สำหรับ high throughput
import asyncio

async def answer_async(questions: list) -> list:
    tasks = [pipeline.run_async({"retriever": {"query": q}}) for q in questions]
    return await asyncio.gather(*tasks)
```

### ❌ Don'ts

```python
# ❌ อย่า hardcode API keys
generator = OpenAIGenerator(api_key="sk-hardcoded")  # BAD

# ❌ อย่า index เอกสารซ้ำทุกครั้ง
# ใช้ policy="skip" หรือเก็บ document IDs ที่ index แล้ว

# ❌ อย่าใช้ InMemoryDocumentStore ใน production
# ข้อมูลจะหายเมื่อ restart — ใช้ Qdrant/Elasticsearch แทน
```

---

## 12. สรุป Phase 4

```
✅ Custom Component — @component decorator สร้างได้ง่าย
✅ Multiple inputs/outputs ใน single component
✅ Routers — MetadataRouter, ConditionalRouter, FileTypeRouter
✅ Agent Systems — Tool definitions + ToolInvoker
✅ Agent Loop — LLM เรียก Tool วนซ้ำจนได้คำตอบ
✅ OutputAdapter แปลง format ระหว่าง components
✅ Conversational RAG — ใส่ chat history
✅ Serialization — save/load pipeline เป็น YAML
✅ Production: env vars, timeout, batch, async
```

---

**➡️ ต่อไป: Phase 5 — Integrations & Real-world Examples**
