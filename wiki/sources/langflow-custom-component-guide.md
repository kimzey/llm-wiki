---
title: "LangFlow Custom Component — คู่มืออธิบายละเอียด"
type: source
source_file: raw/notes/openrag/langchain_component_explained.md
url: ""
published: 2026-01-01
tags: [langflow, langchain, custom-component, redis-cache, openrouter]
related: [wiki/concepts/langflow-visual-workflow.md, wiki/concepts/semantic-caching.md, wiki/concepts/langchain-framework.md]
created: 2026-04-14
updated: 2026-04-14
---

**Full source**: [[../../raw/notes/openrag/langchain_component_explained.md|Original file]]

## สรุป

คู่มืออธิบาย LangFlow Custom Component ตั้งแต่พื้นฐาน — ความสัมพันธ์ระหว่าง LangFlow (GUI) และ LangChain (Python library), วิธีที่ Component base class ทำ auto-binding ของ inputs ไปยัง `self.xxx`, lifecycle ของ component, input types ต่างๆ, outputs, และการใช้ Redis Cache ร่วมกับ ChatOpenAI ผ่าน OpenRouter

## ประเด็นสำคัญ

- **LangFlow vs LangChain**: LangFlow คือ GUI Builder (เหมือน n8n), LangChain คือ Python Library ที่อยู่ข้างใน — `ChatOpenAI`, `RedisCache` ที่ import มาคือ LangChain objects ทั้งนั้น
- **Auto-binding**: `name=` ใน `inputs = [...]` จะกลายเป็น `self.<name>` อัตโนมัติ ผ่าน Component base class magic — ไม่มีข้อยกเว้น
- **Component Lifecycle**: User กรอก UI → LangFlow เก็บค่าใน Input objects → `Component.__init__()` bind ค่าเข้า `self.xxx` → LangFlow เรียก build method → return output ออก port
- **Input Types**: `StrInput`, `SecretStrInput` (password field), `IntInput`, `DropdownInput`, `BoolInput`, `FloatInput`, `MessageTextInput`
- **Output Method**: `method=` ใน outputs ชี้ไปหา method ที่จะ return ค่าออก
- **Redis Cache Flow**: LangChain เช็ค Redis ก่อน → cache hit ตอบทันที, cache miss → ส่งไป API → เก็บใน Redis
- **`advanced=True`**: ซ่อน input ไว้ใน "Advanced" section เพื่อไม่รก UI

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- OpenRouter ใช้เป็น base URL แทน OpenAI โดยตั้ง `openai_api_base="https://openrouter.ai/api/v1"` — ใช้ `ChatOpenAI` เดิมได้เลย ไม่ต้องเปลี่ยน class
- `langchain.llm_cache = RedisCache(...)` ผูก cache แบบ **global** กับ process ปัจจุบัน — ทุก LLM call จะใช้ cache นี้
- `self.log()` ใช้ debug แทน `print()` ใน LangFlow

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่มีข้อขัดแย้ง — เนื้อหาสอดคล้องกับ LangChain framework knowledge เดิม

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/langflow-visual-workflow|Langflow Visual Workflow]] — Component system ที่อธิบายในไฟล์นี้
- [[wiki/concepts/langchain-framework|LangChain Framework]] — LangChain objects ที่ใช้ข้างในทั้งหมด
- [[wiki/concepts/semantic-caching|Semantic Caching]] — Redis Cache ที่ integrate กับ LangFlow component
