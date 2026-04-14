---
title: "การลดต้นทุน AI (Cost Optimization)"
type: concept
tags: [cost-optimization, caching, model-routing, batch-processing, token-optimization, llm]
sources: [raw/notes/ai-engineering/10_cost_optimization.md]
related: [wiki/concepts/semantic-caching, wiki/concepts/ai-production-deployment, wiki/concepts/llm-large-language-model]
created: 2026-04-14
updated: 2026-04-14
---

## สรุปสั้น

AI Cost Optimization คือกลยุทธ์ลดต้นทุนการใช้ LLM โดยใช้ Model Routing (ส่งงานให้โมเดลที่เหมาะสม), Caching (ไม่เรียก API ซ้ำ), Token Optimization (ลด input/output tokens) และ Batch Processing (ราคาถูกกว่า 50%)

## อธิบาย

### ส่วนประกอบของต้นทุน AI

```
Total Cost = LLM Cost + Infrastructure Cost + Development Cost

LLM Cost:
├── Input tokens × ราคา input
├── Output tokens × ราคา output
├── Embedding tokens × ราคา embedding
└── Fine-tuning cost (ถ้ามี)
```

### กลยุทธ์หลัก

#### 1. LLM Routing — ใช้โมเดลให้เหมาะงาน

```
trivial/simple tasks → Claude Haiku (เร็ว ถูก)
moderate/complex tasks → Claude Sonnet (สมดุล)
expert tasks → Claude Opus (ฉลาดสุด แพงสุด)
```

ผลจริง: 60% Haiku + 35% Sonnet + 5% Opus → ประหยัด ~63% จากใช้ Sonnet ทั้งหมด

#### 2. Multi-level Caching

```
L1: Exact Match Cache (Redis, TTL 1hr)      → เร็วที่สุด
L2: Semantic Cache (threshold 0.97)          → คำถามคล้ายๆ กัน
L3: LLM Result Cache (heavy computation)     → ผล RAG สำหรับ document
```

#### 3. Prompt Caching (Anthropic)
วาง static content (long system document) ต้น + mark `cache_control`:
```python
{"type": "text", "text": system_doc, "cache_control": {"type": "ephemeral"}}
# ประหยัด: 100k tokens × $3/1M = $0.30 → ด้วย cache: $0.057 (ประหยัด 81%!)
```

#### 4. Token Optimization

- **Compress History**: summarize messages เก่า แทนเก็บทุก turn
- **Concise System Prompt**: ตัด filler phrases เช่น "แน่นอน!", ไม่ทำซ้ำคำถาม
- **Reduce Output Length**: "ตอบกระชับ ตรงประเด็น" → ประหยัด output 20-40%

#### 5. Batch Processing

- Anthropic Batch API: ถูกกว่า 50% แต่ใช้เวลา 1-24 ชั่วโมง
- เหมาะกับ: classification, data extraction, async analysis tasks

## ประเด็นสำคัญ

- **คำนวณ monthly cost ก่อน optimize**: daily users × conversations × turns × tokens × price
- **Quick Wins (Phase 1)**: exact-match cache + จำกัด output length → ประหยัด 30-50% ทันที
- **Medium Effort (Phase 2)**: Semantic cache + LLM Router + Prompt Caching → 50-70%
- **Strategic (Phase 3)**: Batch API + Fine-tune small model + Self-hosted embedding → 70-90%

### Architecture Decision Guide

| Use Case | โมเดลที่เหมาะ | Pattern |
|----------|--------------|---------|
| Real-time Chat | Sonnet | Streaming SSE |
| Batch Classification | Haiku | Batch API |
| Document Q&A | Sonnet | RAG + Semantic Cache |
| Complex Analysis | Opus | ReAct + Long context |
| High Volume Simple | Haiku | Batching + Heavy Cache |
| Data Extraction | Haiku | Structured Output |

## ตัวอย่าง / กรณีศึกษา

```python
# ตัวอย่างการคำนวณ monthly cost
daily_users = 1000
avg_turns_per_day = 15  # 3 conversations × 5 turns
avg_input_tokens = 500
avg_output_tokens = 200

daily_cost = (
    daily_users * avg_turns_per_day * avg_input_tokens * 3.0/1_000_000 +
    daily_users * avg_turns_per_day * avg_output_tokens * 15.0/1_000_000
)
# monthly ≈ $675 (ด้วย Sonnet ทั้งหมด)
# หลัง routing ≈ $250 (ประหยัด 63%)
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/semantic-caching|Semantic Caching]] — เทคนิค caching สำคัญที่สุดสำหรับ LLM cost
- [[wiki/concepts/ai-production-deployment|AI Production Deployment]] — Cost optimization เป็นส่วนหนึ่งของ production
- [[wiki/concepts/llm-large-language-model|LLM]] — ต้องเข้าใจ tokenization และ pricing model

## แหล่งที่มา

- [[wiki/sources/ai-engineering/cost-optimization|Cost Optimization — ลดต้นทุน AI]]
