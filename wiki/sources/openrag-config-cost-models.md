---
title: "OpenRAG — Configuration, Cost & Model Selection"
type: source
source_file: raw/notes/openrag/docs-lean/advanced-config.md
url: ""
published: 2026-03-17
tags: [openrag, configuration, cost, llm-models, embedding, ollama, openai]
related: [wiki/concepts/openrag-platform.md, wiki/concepts/llm-large-language-model.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/openrag/docs-lean/advanced-config.md|Advanced Config]]
> Additional source: [[../../raw/notes/openrag/docs-lean/cost-and-models.md|Cost & Models]]

## สรุป
OpenRAG มี 2 ชั้น config: `.env` (infrastructure) และ `config/config.yaml` (AI settings) เปลี่ยนได้โดยไม่ต้อง restart. ค่าใช้จ่ายหลักมาจาก LLM Chat ไม่ใช่ Embedding (Embedding ถูกมาก ~$1 ต่อเอกสาร 100 ไฟล์)

## ประเด็นสำคัญ

### config/config.yaml โครงสร้าง
```yaml
agent:
  model: gpt-4.1-mini
  provider: openai          # openai | anthropic | watsonx | ollama
  system_prompt: |          # พฤติกรรม AI — เปลี่ยนได้ผ่าน UI
    You are a helpful assistant...

knowledge:
  embedding_model: text-embedding-3-small
  chunk_size: 1000
  chunk_overlap: 200
  ocr_enabled: false
  picture_descriptions: false   # เปิด = AI อธิบายภาพ แต่เพิ่ม cost
```

### Models ที่รองรับ (ครบ)
**OpenAI:** gpt-5, gpt-4o, gpt-4o-mini, gpt-4.1, gpt-4.1-mini (default), gpt-4.1-nano, o4-mini
**Anthropic:** claude-opus-4-5, claude-sonnet-4-5, claude-haiku-4-5
**IBM WatsonX:** ibm/granite-3-8b-instruct, meta-llama/llama-3-70b-instruct
**Ollama (Local):** llama3.1, qwen2.5, deepseek-r1, gemma2

### ค่าใช้จ่าย LLM ต่อเดือน (ทีม 50 คน, 20 queries/คน/วัน = 22,000 queries/เดือน)
| Model | ค่าใช้จ่าย/เดือน |
|-------|-----------------|
| gpt-4o-mini | ~$9 (~315 บาท) ✅ |
| gpt-4o | ~$151 (~5,300 บาท) |
| claude-sonnet | ~$267 (~9,350 บาท) |
| Ollama (local) | $0 (ใช้ hardware เอง) |

### Embedding Cost (ถูกมาก — จ่ายครั้งเดียวตอน ingest)
```
100 ไฟล์ × 20 หน้า × 700 tokens = 1.4M tokens
text-embedding-3-small @ $0.02/1M = $0.028 ≈ ~1 บาท เท่านั้น
```

### Decision Framework เลือก Model
```
1. ข้อมูล confidential? → ใช้ Ollama (local) เท่านั้น
2. Budget? → น้อย: gpt-4o-mini | มาก: gpt-4o
3. ภาษาไทยสำคัญ? → gpt-4o หรือ claude-sonnet (ดีกว่า mini)
4. Volume > 10,000 queries/เดือน? → พิจารณา Ollama
5. Enterprise support? → IBM WatsonX
```

### Custom System Prompt สำหรับองค์กร
```yaml
# HR Bot (ภาษาไทย)
system_prompt: |
  คุณคือผู้ช่วย HR ของบริษัท ตอบคำถามเป็นภาษาไทยเสมอ
  ให้ข้อมูลจากเอกสารเท่านั้น อ้างอิงแหล่งที่มาทุกครั้ง
  ถ้าไม่มีข้อมูล ให้แนะนำติดต่อ HR โดยตรงที่ hr@company.com
```

### Ollama Setup (Local LLM ฟรี)
```bash
ollama pull llama3.1          # 8B params, ~5GB
ollama pull qwen2.5           # ภาษาไทย/จีนดีกว่า
ollama pull nomic-embed-text  # Embedding model

# Hardware: 7B model ต้องการ GPU VRAM 6-8GB minimum
```

### Feature Flags ใน .env
```bash
DISABLE_INGEST_WITH_LANGFLOW=false  # true = bypass Langflow (เร็วกว่า)
INGEST_SAMPLE_DATA=true             # true = load demo docs ตอน startup
DELETE_LANGFLOW_FILE_AFTER_INGESTION=true
DO_NOT_TRACK=false                  # true = ปิด telemetry
LOG_LEVEL=INFO
```

### Langfuse Tracing (Optional)
```bash
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_PUBLIC_KEY=pk-lf-...
# → ดู traces, LLM calls, cost ต่อ query ทั้งหมด
```

### Priority ของ Config
```
Per-request tweaks > Environment Variables > config.yaml defaults
```

## ข้อมูล / หลักฐาน ที่น่าสนใจ
- Embedding cost ถูกมาก (จ่ายครั้งเดียวตอน ingest) — cost หลักคือ LLM call ตอน chat
- `picture_descriptions: true` — AI อธิบายภาพใน PDF ด้วย Vision model แต่เพิ่ม API cost มาก
- Task cleanup: records เก่ากว่า `TASK_CLEANUP_DAYS=7` วันจะถูกลบอัตโนมัติ
- Config สามารถแก้ผ่าน API ได้: `PUT /settings` (ไม่ต้อง restart)

## Concepts ที่เกี่ยวข้อง
- [[wiki/concepts/openrag-platform.md|OpenRAG Platform]]
- [[wiki/concepts/llm-large-language-model.md|LLM]]
