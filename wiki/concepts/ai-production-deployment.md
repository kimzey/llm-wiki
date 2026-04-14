---
title: "การ Deploy ระบบ AI สู่ Production"
type: concept
tags: [deployment, production, fastapi, rate-limiting, caching, scaling, monitoring, security]
sources: [raw/notes/ai-engineering/07_production_deployment.md]
related: [wiki/concepts/semantic-caching, wiki/concepts/api-design, wiki/concepts/prompt-injection, wiki/concepts/ai-cost-optimization]
created: 2026-04-14
updated: 2026-04-14
---

## สรุปสั้น

การ Deploy ระบบ AI สู่ Production ต้องจัดการหลายด้านพร้อมกัน ทั้ง API Design, Rate Limiting, Caching, Error Handling, Security, Cost Management และ Scaling — ระบบที่ดีบน paper ต้องรอดจริงใน production ด้วย

## อธิบาย

### สถาปัตยกรรม Production AI

```
Internet → [Load Balancer]
               ↓
         [API Gateway] (Rate Limit, Auth, Routing)
               ↓
    ┌──────────┼──────────┐
    ↓          ↓          ↓
[Chat API] [RAG API] [Embed API]
    ↓          ↓          ↓
[LLM API]  [VectorDB]  [Cache]
    ↓
[Logging & Monitoring]
```

### API Design

**Standard (non-streaming)**: รอ response ทั้งหมดก่อนส่ง
**Streaming SSE**: ส่งทีละ token — UX ดีกว่า ผู้ใช้เห็นคำตอบทันที

### Rate Limiting

- **Per IP**: ง่าย แต่ไม่แม่นยำ — VPN ได้เปรียบ
- **Per User Token**: แม่นยำกว่า — นับ tokens จริงที่ใช้ต่อวัน
- **Plans**: free (10k tokens/day) → pro (100k) → enterprise (1M)

### Caching Strategy

```
Request → [Exact Match Cache (Redis, 1hr)] 
       ↓ miss
       → [Semantic Cache (threshold=0.95)]
       ↓ miss
       → [LLM API] → เก็บทั้ง 2 layers
```

### Error Handling

**Exponential Backoff**:
```
Rate limit error → delay = base × 2^attempt + random jitter
Network error → delay = base × 1.5^attempt
Server error (5xx) → retry
Client error (4xx) → ไม่ retry
```

**Fallback Chain**:
```
Primary (Opus) → Secondary (Sonnet) → Fallback (Haiku) → "ระบบขัดข้อง"
```

## ประเด็นสำคัญ

- **Streaming SSE** ดีสำหรับ chat — ผู้ใช้ไม่รู้สึกรอ
- **Queue-based Processing (Celery)** สำหรับ long tasks เช่น document analysis (1-5 นาที)
- **Input Sanitization**: ตรวจ injection patterns + PII ก่อนส่งให้ LLM
- **JWT Authentication**: verify token → ดึง user → check permissions → check plan limits
- **Budget Alert**: ถ้าใช้ไป 90% → แจ้งเตือน; 100% → suspend
- **Horizontal Scaling**: หลาย instances + Redis shared cache + connection pool

### Production Checklist

```
Infrastructure: Load balancer, Auto-scaling, Redis, DB pool
Security: API keys ใน env, HTTPS, Rate limiting, Input validation
Reliability: Retry logic, Fallback model, Health check endpoint
Monitoring: Error rate, Latency (p50/p95/p99), Cost tracking, Alerts
```

## ตัวอย่าง / กรณีศึกษา

```python
# Token-based Rate Limiter
key = f"token_usage:{user_id}:{datetime.now().date()}"
current = int(redis.get(key) or 0)
if current + tokens > limit:
    raise HTTPException(status_code=429, detail="Token limit exceeded")
# Atomic increment + expire ตอนเที่ยงคืน
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/semantic-caching|Semantic Caching]] — เทคนิค caching ขั้นสูงสำหรับ LLM responses
- [[wiki/concepts/api-design|API Design]] — การออกแบบ API สำหรับ AI systems
- [[wiki/concepts/prompt-injection|Prompt Injection & AI Security]] — ภัยคุกคามที่ต้องป้องกันใน production
- [[wiki/concepts/ai-cost-optimization|AI Cost Optimization]] — ลดต้นทุนใน production

## แหล่งที่มา

- [[wiki/sources/ai-engineering/production-deployment|Production Deployment — ระบบ AI สู่ Production]]
