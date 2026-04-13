# 🌾 Haystack Deep Dive — Phase 4: RAG Pipeline ขั้นสูง

> **เป้าหมาย Phase 4:** สร้าง RAG ที่สมบูรณ์พร้อม Generators, Prompt Engineering, Memory, และ Agents

---

## 1. Generators — สร้างคำตอบด้วย LLM

### 1.1 OpenAIGenerator

```python
from haystack.components.generators import OpenAIGenerator
import os

os.environ["OPENAI_API_KEY"] = "sk-..."

generator = OpenAIGenerator(
    model="gpt-4o",                      # หรือ gpt-4o-mini, gpt-3.5-turbo
    generation_kwargs={
        "max_tokens": 1000,
        "temperature": 0.7,              # 0=แม่นยำ, 1=สร้างสรรค์
        "top_p": 0.95,
        "frequency_penalty": 0.0,
        "presence_penalty": 0.0
    },
    streaming_callback=None,             # callback สำหรับ streaming
    timeout=30,
    api_base_url=None                    # ใช้ custom endpoint ได้
)

result = generator.run(
    prompt="อธิบาย Haystack ให้ฉันฟังหน่อย"
)
print(result["replies"][0])   # คำตอบ
print(result["meta"])         # token usage, model info
```

### 1.2 OpenAIChatGenerator (Multi-turn)

```python
from haystack.components.generators.chat import OpenAIChatGenerator
from haystack.dataclasses import ChatMessage

generator = OpenAIChatGenerator(
    model="gpt-4o",
    generation_kwargs={"temperature": 0.7}
)

messages = [
    ChatMessage.from_system("คุณเป็น AI Assistant ผู้เชี่ยวชาญด้าน Python"),
    ChatMessage.from_user("Python คืออะไร?")
]

result = generator.run(messages=messages)
reply = result["replies"][0]

print(reply.content)  # คำตอบ
print(reply.role)     # "assistant"

# ต่อการสนทนา
messages.append(reply)
messages.append(ChatMessage.from_user("แล้วใช้ Python ทำ AI ยังไง?"))
result = generator.run(messages=messages)
```

### 1.3 HuggingFaceLocalGenerator (Local Model)

```python
# pip install transformers torch
from haystack.components.generators import HuggingFaceLocalGenerator

generator = HuggingFaceLocalGenerator(
    model="google/flan-t5-base",         # หรือ model อื่น
    task="text2text-generation",
    generation_kwargs={
        "max_new_tokens": 500,
        "temperature": 0.1,
        "do_sample": True
    }
)
generator.warm_up()  # โหลด model ลง memory

result = generator.run(prompt="Explain RAG in simple terms")
print(result["replies"][0])
```

### 1.4 HuggingFaceAPIGenerator (HF Inference API)

```python
from haystack.components.generators import HuggingFaceAPIGenerator

generator = HuggingFaceAPIGenerator(
    api_type="serverless_inference_api",
    api_params={
        "model": "HuggingFaceH4/zephyr-7b-beta",
        "token": "hf-token..."
    }
)
```

### 1.5 OllamaGenerator (Local LLM ด้วย Ollama)

```python
# pip install ollama-haystack
# ต้องรัน: ollama pull llama3
from haystack_integrations.components.generators.ollama import OllamaGenerator

generator = OllamaGenerator(
    model="llama3",
    url="http://localhost:11434",
    generation_kwargs={"temperature": 0.3}
)

result = generator.run(prompt="สวัสดี ฉันชื่อ...")
```

### 1.6 AnthropicGenerator

```python
# pip install anthropic-haystack
from haystack_integrations.components.generators.anthropic import AnthropicGenerator

generator = AnthropicGenerator(
    model="claude-3-5-sonnet-20241022",
    generation_kwargs={"max_tokens": 1024}
)
```

---

## 2. PromptBuilder — สร้าง Prompt อย่างมืออาชีพ

PromptBuilder ใช้ **Jinja2 templating**

### 2.1 Basic Template

```python
from haystack.components.builders import PromptBuilder

template = """
คุณเป็น AI Assistant ที่เชี่ยวชาญด้านการค้นหาข้อมูล

บริบทจากเอกสาร:
{% for doc in documents %}
[{{ loop.index }}] {{ doc.content }}
   (แหล่งที่มา: {{ doc.meta.get('file_path', 'unknown') }})
{% endfor %}

คำถาม: {{ question }}

คำตอบ (ตอบเป็นภาษาไทย อ้างอิง [เลข] ที่มา):
"""

prompt_builder = PromptBuilder(template=template)

result = prompt_builder.run(
    documents=retrieved_docs,
    question="Haystack ใช้ทำอะไร?"
)
print(result["prompt"])
```

### 2.2 Template พร้อม Conditional Logic

```python
template = """
{% if system_prompt %}
System: {{ system_prompt }}
{% endif %}

{% if context %}
บริบท:
{% for doc in documents %}
{{ doc.content }}
{% endfor %}
{% endif %}

{% if history %}
ประวัติการสนทนา:
{% for turn in history %}
User: {{ turn.user }}
Assistant: {{ turn.assistant }}
{% endfor %}
{% endif %}

User: {{ question }}
Assistant:
"""

prompt_builder = PromptBuilder(
    template=template,
    required_variables=["question"]   # variable ที่ต้องมีเสมอ
)
```

### 2.3 DynamicPromptBuilder (Template จาก runtime)

```python
from haystack.components.builders import DynamicPromptBuilder

# Template ส่งมาตอน run ได้ (ยืดหยุ่นกว่า)
dynamic_builder = DynamicPromptBuilder(
    runtime_variable_names=["documents", "question"]
)

custom_template = "ตอบคำถาม: {{ question }}"

result = dynamic_builder.run(
    prompt_source=custom_template,
    question="Python คืออะไร?",
    documents=[]
)
```

---

## 3. Complete RAG Pipeline

### 3.1 Standard RAG

```python
from haystack import Pipeline
from haystack.components.embedders import SentenceTransformersTextEmbedder
from haystack.components.retrievers.in_memory import InMemoryEmbeddingRetriever
from haystack.components.builders import PromptBuilder
from haystack.components.generators import OpenAIGenerator
from haystack.document_stores.in_memory import InMemoryDocumentStore

document_store = InMemoryDocumentStore()
# (สมมติ document_store มีข้อมูลแล้ว)

RAG_TEMPLATE = """
ตอบคำถามโดยอ้างอิงจากบริบทต่อไปนี้เท่านั้น
ถ้าไม่มีข้อมูลในบริบท ให้ตอบว่า "ไม่พบข้อมูลในเอกสาร"

บริบท:
{% for doc in documents %}
---
{{ doc.content }}
{% endfor %}

คำถาม: {{ question }}
คำตอบ:
"""

rag_pipeline = Pipeline()
rag_pipeline.add_component("embedder", SentenceTransformersTextEmbedder(
    model="BAAI/bge-m3"
))
rag_pipeline.add_component("retriever", InMemoryEmbeddingRetriever(
    document_store=document_store,
    top_k=5
))
rag_pipeline.add_component("prompt_builder", PromptBuilder(template=RAG_TEMPLATE))
rag_pipeline.add_component("llm", OpenAIGenerator(model="gpt-4o-mini"))

rag_pipeline.connect("embedder.embedding",        "retriever.query_embedding")
rag_pipeline.connect("retriever.documents",       "prompt_builder.documents")
rag_pipeline.connect("prompt_builder.prompt",     "llm.prompt")

result = rag_pipeline.run({
    "embedder":        {"text": "Haystack ใช้ทำอะไร?"},
    "prompt_builder":  {"question": "Haystack ใช้ทำอะไร?"}
})

answer = result["llm"]["replies"][0]
print(answer)
```

### 3.2 RAG with Source Citation

```python
RAG_TEMPLATE_WITH_CITATION = """
ตอบคำถามจากบริบทที่ให้ พร้อมอ้างอิงแหล่งที่มา

{% for doc in documents %}
[{{ loop.index }}] {{ doc.content }}
    📄 แหล่งที่มา: {{ doc.meta.get('file_path', 'N/A') }} หน้า {{ doc.meta.get('page_number', '?') }}
{% endfor %}

คำถาม: {{ question }}

คำตอบ (ระบุ [เลข] อ้างอิง):
"""
```

---

## 4. Streaming Response

```python
from haystack.components.generators import OpenAIGenerator
from haystack.dataclasses import StreamingChunk

def streaming_callback(chunk: StreamingChunk):
    print(chunk.content, end="", flush=True)

generator = OpenAIGenerator(
    model="gpt-4o",
    streaming_callback=streaming_callback
)

result = generator.run(prompt="เล่าเรื่อง Haystack ให้ฟัง...")
# ข้อความจะ print แบบ real-time
```

---

## 5. Conversational RAG (Multi-turn)

```python
from haystack import Pipeline
from haystack.components.builders.chat_prompt_builder import ChatPromptBuilder
from haystack.components.generators.chat import OpenAIChatGenerator
from haystack.components.retrievers.in_memory import InMemoryEmbeddingRetriever
from haystack.components.embedders import SentenceTransformersTextEmbedder
from haystack.dataclasses import ChatMessage

CHAT_TEMPLATE = """
{% for message in messages %}
    {% if message.role == 'system' %}
        {{ message.content }}
    {% elif message.role == 'user' %}
        User: {{ message.content }}
    {% elif message.role == 'assistant' %}
        Assistant: {{ message.content }}
    {% endif %}
{% endfor %}

บริบท:
{% for doc in documents %}
{{ doc.content }}
{% endfor %}

User: {{ question }}
"""

pipeline = Pipeline()
pipeline.add_component("embedder",      SentenceTransformersTextEmbedder(model="BAAI/bge-m3"))
pipeline.add_component("retriever",     InMemoryEmbeddingRetriever(document_store, top_k=3))
pipeline.add_component("prompt_builder", ChatPromptBuilder())
pipeline.add_component("llm",           OpenAIChatGenerator(model="gpt-4o"))

pipeline.connect("embedder.embedding",      "retriever.query_embedding")
pipeline.connect("retriever.documents",     "prompt_builder.documents")
pipeline.connect("prompt_builder.prompt",   "llm.messages")

# สนทนา
history = []
system = ChatMessage.from_system("คุณเป็น AI Assistant ผู้เชี่ยวชาญด้านเอกสาร")

def chat(question: str) -> str:
    messages = [system] + history

    result = pipeline.run({
        "embedder":       {"text": question},
        "prompt_builder": {
            "template": CHAT_TEMPLATE,
            "messages": messages,
            "question": question
        }
    })

    reply = result["llm"]["replies"][0]
    history.append(ChatMessage.from_user(question))
    history.append(reply)
    return reply.content

print(chat("Haystack คืออะไร?"))
print(chat("แล้วมันต่างจาก LangChain ยังไง?"))  # จำบริบทเดิม
```

---

## 6. Pipeline Serialization — บันทึกและโหลด Pipeline

```python
import yaml
from haystack import Pipeline

# สร้าง pipeline
pipeline = Pipeline()
# ... add components ...

# บันทึกเป็น YAML
with open("my_pipeline.yaml", "w") as f:
    pipeline.dump(f)

# โหลดกลับมา
with open("my_pipeline.yaml", "r") as f:
    loaded_pipeline = Pipeline.load(f)

# หรือเป็น dict
pipeline_dict = pipeline.to_dict()
loaded_pipeline = Pipeline.from_dict(pipeline_dict)
```

ตัวอย่าง YAML ที่ได้:

```yaml
version: ignore
components:
  retriever:
    type: haystack.components.retrievers.in_memory.bm25_retriever.InMemoryBM25Retriever
    init_parameters:
      document_store:
        type: haystack.document_stores.in_memory.document_store.InMemoryDocumentStore
      top_k: 5
  llm:
    type: haystack.components.generators.openai.OpenAIGenerator
    init_parameters:
      model: gpt-4o-mini
connections:
  - sender: retriever.documents
    receiver: prompt_builder.documents
```

---

## 7. Pipeline Visualization

```python
# แสดงโครงสร้าง Pipeline
pipeline.draw("pipeline.png")  # ต้องติดตั้ง graphviz

# หรือดูใน terminal
print(pipeline.graph.nodes)
print(pipeline.graph.edges)
```

---

## 8. Error Handling & Retry

```python
from haystack import Pipeline
from haystack.components.routers import MetadataRouter

# Route ตาม condition
router = MetadataRouter(rules={
    "high_confidence": {
        "field": "score",
        "operator": ">=",
        "value": 0.8
    },
    "low_confidence": {
        "field": "score",
        "operator": "<",
        "value": 0.8
    }
})

# ConditionalRouter สำหรับ branch logic
from haystack.components.routers import ConditionalRouter

routes = [
    {
        "condition": "{{ documents|length > 0 }}",
        "output": "{{ documents }}",
        "output_name": "documents_found",
        "output_type": list
    },
    {
        "condition": "{{ documents|length == 0 }}",
        "output": "{{ [] }}",
        "output_name": "no_documents",
        "output_type": list
    }
]

conditional_router = ConditionalRouter(routes=routes)
```

---

## 9. Output Parsers

```python
from haystack.components.converters import OutputAdapter

# แปลง output ก่อนส่งต่อ
adapter = OutputAdapter(
    template="{{ replies[0] }}",           # Jinja2 template
    output_type=str,
    unsafe=False
)

result = adapter.run(replies=["คำตอบ 1", "คำตอบ 2"])
print(result["output"])  # "คำตอบ 1"

# ตัวอย่าง: แปลง reply เป็น dict
json_adapter = OutputAdapter(
    template="{{ replies[0] | from_json }}",
    output_type=dict,
    custom_filters={"from_json": lambda s: __import__('json').loads(s)}
)
```

---

## 10. สรุป Phase 4

| หัวข้อ | สิ่งที่ได้เรียนรู้ |
|---|---|
| Generators | OpenAI, HuggingFace, Ollama, Anthropic |
| PromptBuilder | Jinja2 template สำหรับสร้าง prompt |
| Complete RAG | Embed → Retrieve → Prompt → Generate |
| Streaming | real-time output ด้วย callback |
| Conversational RAG | Multi-turn พร้อม history |
| Serialization | บันทึก/โหลด Pipeline เป็น YAML |
| Conditional Router | Branch logic ใน Pipeline |

---

## 📚 Phase ถัดไป

- **Phase 5:** Custom Components & Production Deployment
