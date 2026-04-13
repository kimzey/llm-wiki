---
title: "Haystack Deep Dive — Phase 4: Advanced Features (Agents, Routers, Conversational RAG)"
type: source
source_file: raw/notes/Hetstack/haystack-phase4-advanced.md
url: ""
published: 2026-04-01
tags: [haystack, agents, tool-calling, router, conversational-rag, custom-component, streaming]
related: [wiki/sources/haystack-phase3-retrieval.md, wiki/sources/haystack-phase5-production.md, wiki/concepts/haystack-framework.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/Hetstack/haystack-phase4-advanced.md|Original file]]
> *ดูเพิ่ม: [[../../raw/notes/Hetstack/haystack-phase4-rag-advanced.md|RAG Advanced alt file]]*

## สรุป

Phase 4 ครอบคลุม features ขั้นสูง — Custom Components ด้วย `@component` decorator, Router systems, Agent Systems ด้วย Tool Use + ToolInvoker, Conversational RAG (multi-turn + history), Streaming, Pipeline Serialization เป็น YAML, และ error handling

## ประเด็นสำคัญ

- **Custom Component**: `@component` decorator + `@component.output_types()` + `run()` method — สร้างง่ายมาก
- **Routers**: MetadataRouter (route ตาม metadata field), ConditionalRouter (Jinja2 expression), FileTypeRouter (MIME type)
- **Agent Systems**: Tool definitions + ToolInvoker หรือ `Agent` component ที่มี built-in loop — `max_agent_steps` ป้องกัน infinite loop
- **Conversational RAG**: ส่ง `chat_history` เข้า DynamicChatPromptBuilder ทุก turn
- **Streaming**: `streaming_callback=fn` ใน OpenAIGenerator — real-time output
- **Serialization**: `pipeline.dump(f)` / `Pipeline.load(f)` เป็น YAML — สำคัญสำหรับ deploy
- **BranchJoiner**: รวม parallel paths กลับเป็น single stream

## ข้อมูล / หลักฐาน ที่น่าสนใจ

**Custom Component template:**
```python
@component
class MyComponent:
    def __init__(self, config: str):
        self.config = config

    @component.output_types(result=str, count=int)
    def run(self, input: str) -> dict:
        return {"result": input, "count": len(input)}
```

**Stateful Component** (เก็บ conversation history):
```python
@component
class ConversationHistory:
    def __init__(self, max_history: int = 10):
        self.history = []  # state คงอยู่ระหว่าง runs

    @component.output_types(history=List[dict])
    def run(self, user_message: str) -> dict:
        self.history.append({"role": "user", "content": user_message})
        return {"history": self.history}
```

**Agent Loop pattern:**
```
LLM → ToolInvoker → LLM (loop จนกว่า LLM ไม่ได้ call tool)
```

**Warm_up สำหรับ Component ที่โหลด Model:**
```python
def warm_up(self):
    """เรียกอัตโนมัติตอน pipeline.warm_up() — โหลด model ครั้งเดียว"""
    self.model = SentenceTransformer(self.model_name)
```

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่มีข้อขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/haystack-framework|Haystack Framework]]
