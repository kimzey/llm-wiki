---
title: "Production Deployment — ระบบ AI สู่ Production"
type: source
source_file: "raw/notes/ai-engineering/07_production_deployment.md"
tags: [deployment, fastapi, rate-limiting, caching, scaling, monitoring, security]
related: [wiki/concepts/ai-production-deployment, wiki/concepts/api-design, wiki/concepts/semantic-caching]
created: 2026-04-14
updated: 2026-04-14
---

> **Full source**: [[../../../raw/notes/ai-engineering/07_production_deployment.md|Original file]]

## สรุป

คู่มือ Production Deployment ครอบคลุมสถาปัตยกรรม (Load Balancer → API Gateway → Services → LLM API), API Design (REST + Streaming SSE), Rate Limiting (per IP/User/Token), Multi-layer Caching, Error Handling + Retry, Security (Input Sanitization, Auth/JWT), Cost Management และ Horizontal Scaling

## ประเด็นสำคัญ

- **สถาปัตยกรรม**: Internet → Load Balancer → API Gateway → Services (Chat/RAG/Embedding) → LLM + VectorDB + Cache
- **Streaming SSE**: ดีกว่าสำหรับ UX — ผู้ใช้เห็นคำตอบทีละคำแบบ real-time
- **Rate Limiting**: Per IP (ง่าย) vs Per User Token (แม่นยำกว่า)
- **Token-based Rate Limit**: นับ tokens จริงที่ใช้ต่อวัน ตาม plan (free/pro/enterprise)
- **Multi-layer Cache**: Exact Match (Redis) → Semantic Cache → LLM API
- **Semantic Cache**: ค้นหาคำถามที่คล้ายกัน threshold 0.95 → return cached response
- **Exponential Backoff**: retry delay = base × 2^attempt + random jitter
- **Fallback Chain**: Primary LLM → Secondary LLM → Fallback LLM → "ระบบขัดข้อง"
- **Queue-based Processing**: Celery + Redis สำหรับ tasks ที่ใช้เวลานาน

## ข้อมูล / หลักฐาน ที่น่าสนใจ

> [!note] Claude Pricing (ตัวอย่างในเอกสาร)
> | Model | Input | Output |
> |-------|-------|--------|
> | Opus | $15/1M | $75/1M |
> | Sonnet | $3/1M | $15/1M |
> | Haiku | $0.25/1M | $1.25/1M |

```python
# Input Sanitization
injection_patterns = ['ignore (all )?previous instructions', 'disregard (the )?system prompt']
for pattern in injection_patterns:
    if re.search(pattern, text, re.IGNORECASE):
        log.warning("Possible prompt injection detected")
```

- PII Detection: Thai ID, Credit Card, Phone, Email — ต้องตรวจก่อนส่งให้ LLM
- Deployment Checklist: Infrastructure + Security + Reliability + Monitoring + Testing ครบทุกด้าน

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/ai-production-deployment|การ Deploy ระบบ AI สู่ Production]]
- [[wiki/concepts/api-design|API Design]]
- [[wiki/concepts/semantic-caching|Semantic Caching]]
- [[wiki/concepts/prompt-injection|Prompt Injection & AI Security]]
