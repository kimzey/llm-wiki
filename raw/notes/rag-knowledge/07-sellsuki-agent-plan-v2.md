# Sellsuki RAG Agent — Plan v2 (Optimized)

> อัปเดตจาก Plan v1 โดยอ้างอิง production case study (Elysia Docs RAG)
> และแนวทาง cost optimization ที่ใช้จริงในปัจจุบัน

---

## สารบัญ

1. [สิ่งที่เปลี่ยนจาก Plan v1 และเหตุผล](#1-สิ่งที่เปลี่ยนจาก-plan-v1)
2. [เปรียบเทียบ 3 Strategy](#2-เปรียบเทียบ-3-strategy)
3. [Stack เปรียบเทียบละเอียด](#3-stack-เปรียบเทียบละเอียด)
4. [Cost Optimization Playbook](#4-cost-optimization-playbook)
5. [Architecture ใหม่](#5-architecture-ใหม่)
6. [Implementation Plan v2](#6-implementation-plan-v2)
7. [เปรียบเทียบ Plan v1 vs v2](#7-เปรียบเทียบ-plan-v1-vs-v2)

---

## 1. สิ่งที่เปลี่ยนจาก Plan v1

### ปัญหาของ Plan v1

```
Plan v1 มีจุดที่ยังไม่ดีพอ:

❌ Cost สูงเกินไป ($50-70/เดือน) สำหรับ internal tool
   → ใช้ OpenAI เป็นหลัก ทั้ง embedding + LLM
   → ไม่มี caching strategy
   → ทุก request เรียก LLM + embedding ใหม่หมด

❌ Latency ไม่ได้ optimize
   → Database อยู่คนละ server กับ app (network hop)
   → ไม่มี cache layer
   → ไม่ได้พิจารณา Time-to-First-Token (TTFT)

❌ ไม่มี anti-abuse
   → ไม่มี rate limiting strategy
   → ไม่มี PoW / CAPTCHA
   → เปิด API โล่งๆ

❌ Framework-heavy
   → แนะนำ LangChain ซึ่ง overhead เยอะ
   → สำหรับ use case ง่ายๆ เขียนเองเร็วกว่า
   → Dependencies เยอะ = break บ่อย

❌ Chunking strategy ยังเบสิก
   → ยังไม่ได้ทำ summarized content
   → ไม่ได้ทำ incremental indexing (เทียบ diff)
   → Re-index ทั้งหมดทุกครั้ง
```

### สิ่งที่เพิ่มใน Plan v2

```
✅ 3 Strategy ให้เลือกตาม budget/scale
✅ Cost optimization techniques จาก production case
✅ Semantic Caching (ลด AI cost 40-60%)
✅ BM25 + Cosine hybrid search (แม่นกว่า vector อย่างเดียว)
✅ Incremental indexing (diff-based, ไม่ re-index ทั้งหมด)
✅ Summarized content (ลด context size)
✅ Anti-abuse (PoW + Turnstile + rate limit)
✅ LLM Provider alternatives (Groq, Cerebras, Gemini Flash)
✅ Co-location strategy (ลด latency)
✅ DIY vs Framework decision matrix
✅ Observability & logging
```

---

## 2. เปรียบเทียบ 3 Strategy

### Strategy A: "Enterprise Safe" — ใช้ managed services

```
เหมาะกับ: บริษัทที่มี budget, ไม่อยากจัดการ infra เอง
Cost:     $50-100/เดือน
Latency:  400-800ms
Effort:   ★★★☆☆ (ปานกลาง)

Stack:
  LLM        → OpenAI GPT-4o-mini / Claude Sonnet
  Embedding  → OpenAI text-embedding-3-small
  Vector DB  → Supabase (pgvector managed) / Neon
  Framework  → LangChain / LlamaIndex
  Hosting    → Google Cloud Run / Railway
  Cache      → Upstash Redis (managed)

ข้อดี:
  + Setup ง่าย managed หมด
  + มี support
  + Scale อัตโนมัติ
  + เชื่อถือได้ เหมาะ production

ข้อเสีย:
  - แพงสุด
  - Vendor lock-in
  - Latency สูง (หลาย network hops)
  - ยากที่จะ optimize cost ลงอีก
```

### Strategy B: "Balanced" — self-host บางส่วน ← แนะนำ

```
เหมาะกับ: ทีมมี dev 1-2 คน, อยากคุม cost แต่ไม่ hardcore
Cost:     $20-35/เดือน
Latency:  150-400ms
Effort:   ★★★★☆ (ใช้เวลาหน่อย)

Stack:
  LLM        → Groq (Llama 3.3 70B) / Gemini Flash
               fallback → GPT-4o-mini
  Embedding  → Gemini text-embedding-004 (free tier)
               หรือ local: fastembed / BGE-M3
  Vector DB  → PostgreSQL + pgvector (self-host)
               หรือ ParadeDB (pgvector + BM25 built-in)
  Framework  → DIY (FastAPI + httpx + psycopg)
               หรือ LlamaIndex (ถ้าอยากมี framework)
  Hosting    → Hetzner / DigitalOcean VPS
  Cache      → Dragonfly (Redis-compatible, เร็วกว่า)
  Deploy     → Coolify / Dokku (self-host PaaS)

ข้อดี:
  + Cost ลดลง 50-60% จาก Strategy A
  + คุม performance ได้
  + Co-location (DB + App อยู่ที่เดียว = latency ต่ำ)
  + ไม่ติด vendor lock-in

ข้อเสีย:
  - ต้องจัดการ server เอง
  - ต้องทำ monitoring เอง
  - ใช้เวลา setup นานกว่า
```

### Strategy C: "Cost Killer" — optimize ทุกอย่าง (Elysia-style)

```
เหมาะกับ: ทีมที่ต้องการ cut cost สุดๆ, มี dev skill แข็ง
Cost:     $12-25/เดือน
Latency:  100-300ms
Effort:   ★★★★★ (ใช้เวลาเยอะ)

Stack:
  LLM        → Groq (Llama 3.3 70B / GPT OSS 120B)
               flex mode, retry with exponential backoff
               fallback reasoning model สำหรับ complex queries
  Embedding  → Self-host fastembed / BGE-M3 (ฟรี)
               หรือ Gemini embedding (free tier)
  Vector DB  → ParadeDB (Postgres + pgvector + BM25 ในตัว)
  Search     → Hybrid: BM25 (FTS) + Cosine Similarity
  Framework  → DIY ทั้งหมด ไม่ใช้ framework
  Hosting    → Hetzner (ถูกสุด) / OVH
  Cache      → Dragonfly + Semantic Cache + Prompt Cache
               + DB Query Cache + Embedding Cache
  Deploy     → Coolify (self-host PaaS)
  Anti-abuse → PoW Challenge + Turnstile + rate limit
  Indexing   → Diff-based incremental + summarized content

ข้อดี:
  + ถูกที่สุด
  + เร็วที่สุด (co-location + cache ทุกชั้น)
  + เข้าใจ system ทุกส่วน ไม่มี black box
  + Scale ได้ชัดเจน (horizontal + read replica)

ข้อเสีย:
  - ใช้เวลา build เยอะ
  - ต้อง maintain เอง
  - ต้องมี skill ระดับหนึ่ง
  - Model อาจไม่ฉลาดเท่า GPT-4o (แต่ถูกกว่า 10x)
```

---

## 3. Stack เปรียบเทียบละเอียด

### 3.1 LLM Providers

```
┌─────────────────┬──────────┬───────────┬──────────┬──────────┬───────────┐
│ Provider/Model  │ TTFT     │ Speed     │ Cost/1M  │ ความฉลาด │ Free Tier │
│                 │ (ms)     │ (tok/s)   │ tokens   │          │           │
├─────────────────┼──────────┼───────────┼──────────┼──────────┼───────────┤
│ GPT-4o          │ 300-500  │ ~100      │ $2.50 in │ ★★★★★   │ ❌        │
│                 │          │           │ $10 out  │          │           │
├─────────────────┼──────────┼───────────┼──────────┼──────────┼───────────┤
│ GPT-4o-mini     │ 200-400  │ ~150      │ $0.15 in │ ★★★★☆   │ ❌        │
│                 │          │           │ $0.60 out│          │           │
├─────────────────┼──────────┼───────────┼──────────┼──────────┼───────────┤
│ Claude Sonnet   │ 300-600  │ ~90       │ $3.00 in │ ★★★★★   │ ❌        │
│ 4.5             │          │           │ $15 out  │          │           │
├─────────────────┼──────────┼───────────┼──────────┼──────────┼───────────┤
│ Gemini 2.0      │ 150-300  │ ~200      │ $0.10 in │ ★★★★☆   │ ✅ มี     │
│ Flash           │          │           │ $0.40 out│          │ 1500 req/ │
│                 │          │           │          │          │ day       │
├─────────────────┼──────────┼───────────┼──────────┼──────────┼───────────┤
│ Groq            │ 50-150   │ ~500      │ $0.05 in │ ★★★★☆   │ ✅ มี     │
│ Llama 3.3 70B   │          │           │ $0.08 out│ (ไทยโอเค)│ 30 req/   │
│                 │          │           │          │          │ min       │
├─────────────────┼──────────┼───────────┼──────────┼──────────┼───────────┤
│ Groq            │ 80-200   │ ~300      │ ~$0.10   │ ★★★★☆   │ ✅        │
│ GPT OSS 120B    │          │           │          │          │           │
├─────────────────┼──────────┼───────────┼──────────┼──────────┼───────────┤
│ Cerebras        │ 30-100   │ ~800+     │ $0.10 in │ ★★★☆☆   │ ✅ มี     │
│ Llama 3.3 70B   │          │           │ $0.10 out│(ไทยพอได้)│           │
├─────────────────┼──────────┼───────────┼──────────┼──────────┼───────────┤
│ Sambanova       │ 50-150   │ ~400      │ Free     │ ★★★☆☆   │ ✅ ฟรี    │
│                 │          │           │ (beta)   │          │           │
└─────────────────┴──────────┴───────────┴──────────┴──────────┴───────────┘

สำหรับภาษาไทย:
  ดีมาก: GPT-4o, Claude Sonnet, Gemini Flash
  ดี:    GPT-4o-mini, Groq Llama 3.3 70B
  พอได้: Cerebras, Sambanova (ไทยอาจสะดุดบ้าง)

แนะนำ Strategy:
  Primary:  Groq Llama 3.3 70B (ถูก + เร็ว + ไทยโอเค)
  Fallback: Gemini Flash (free tier) หรือ GPT-4o-mini (ฉลาดกว่า)
  Complex:  GPT-4o / Claude (เฉพาะ query ที่ model ถูกตอบไม่ได้)
```

### 3.2 Embedding Models

```
┌──────────────────────────┬──────┬──────────┬───────────┬──────────┐
│ Model                    │ มิติ  │ ภาษาไทย   │ Cost/1M   │ ที่ตั้ง    │
│                          │      │          │ tokens    │          │
├──────────────────────────┼──────┼──────────┼───────────┼──────────┤
│ OpenAI                   │ 1536 │ ★★★★☆   │ $0.02     │ Cloud    │
│ text-embedding-3-small   │      │          │           │          │
├──────────────────────────┼──────┼──────────┼───────────┼──────────┤
│ OpenAI                   │ 3072 │ ★★★★★   │ $0.13     │ Cloud    │
│ text-embedding-3-large   │      │          │           │          │
├──────────────────────────┼──────┼──────────┼───────────┼──────────┤
│ Google                   │ 768  │ ★★★★☆   │ FREE      │ Cloud    │
│ text-embedding-004       │      │          │ (free tier│          │
│                          │      │          │ generous) │          │
├──────────────────────────┼──────┼──────────┼───────────┼──────────┤
│ Cohere                   │ 1024 │ ★★★★★   │ $0.10     │ Cloud    │
│ embed-multilingual-v3    │      │          │           │          │
├──────────────────────────┼──────┼──────────┼───────────┼──────────┤
│ BGE-M3                   │ 1024 │ ★★★★☆   │ FREE      │ Local    │
│ (BAAI, open source)      │      │          │ self-host │          │
├──────────────────────────┼──────┼──────────┼───────────┼──────────┤
│ fastembed                │ 384  │ ★★★☆☆   │ FREE      │ Local    │
│ (Qdrant, open source)    │      │          │ self-host │ เร็วมาก  │
├──────────────────────────┼──────┼──────────┼───────────┼──────────┤
│ multilingual-e5-large    │ 1024 │ ★★★★☆   │ FREE      │ Local    │
│ (Microsoft, open source) │      │          │ self-host │          │
└──────────────────────────┴──────┴──────────┴───────────┴──────────┘

แนะนำ:
  Budget:    Google text-embedding-004 (ฟรี + คุณภาพดี)
  Self-host: BGE-M3 (multilingual ดีมาก + ฟรี)
  Best:      Cohere embed-multilingual-v3 (ไทยดีที่สุด)
  Quick:     OpenAI text-embedding-3-small (ง่ายสุด)
```

### 3.3 Database

```
┌─────────────────┬──────────┬───────────┬──────────┬──────────┬──────────┐
│ Database        │ Vector   │ BM25/FTS  │ Cost     │ ความง่าย  │ หมายเหตุ │
├─────────────────┼──────────┼───────────┼──────────┼──────────┼──────────┤
│ PostgreSQL      │ ✅       │ ✅ built-in│ Self-host│ ★★★★☆   │ ต้อง     │
│ + pgvector      │ extension│ ts_vector │ ฟรี      │          │ setup    │
│                 │          │ (ไม่ใช่   │          │          │ เอง     │
│                 │          │ BM25 จริง)│          │          │          │
├─────────────────┼──────────┼───────────┼──────────┼──────────┼──────────┤
│ ParadeDB        │ ✅       │ ✅ BM25    │ Self-host│ ★★★★★   │ Postgres │
│                 │ pgvector │ จริงๆ     │ ฟรี /    │          │ ที่ setup│
│                 │ built-in │ (pg_search│ managed  │          │ มาแล้ว  │
│                 │          │ extension)│ มี       │          │ ครบ     │
├─────────────────┼──────────┼───────────┼──────────┼──────────┼──────────┤
│ Supabase        │ ✅       │ ❌ FTS    │ Free tier│ ★★★★★   │ Managed  │
│ (Postgres)      │ pgvector │ เบสิก    │ → $25/mo │          │ ง่ายมาก │
├─────────────────┼──────────┼───────────┼──────────┼──────────┼──────────┤
│ Neon            │ ✅       │ ❌        │ Free tier│ ★★★★★   │ Serverless│
│ (Serverless PG) │ pgvector │           │ → $19/mo │          │ Postgres │
├─────────────────┼──────────┼───────────┼──────────┼──────────┼──────────┤
│ Qdrant          │ ✅       │ ❌        │ Self-host│ ★★★★☆   │ Rust,    │
│                 │ native   │           │ ฟรี /    │          │ เร็วมาก │
│                 │          │           │ cloud มี │          │          │
├─────────────────┼──────────┼───────────┼──────────┼──────────┼──────────┤
│ ChromaDB        │ ✅       │ ❌        │ ฟรี      │ ★★★★★   │ เหมาะ   │
│                 │ native   │           │          │          │ prototype│
│                 │          │           │          │          │ เท่านั้น │
├─────────────────┼──────────┼───────────┼──────────┼──────────┼──────────┤
│ LanceDB        │ ✅       │ ✅ FTS     │ ฟรี      │ ★★★★☆   │ Embedded │
│                │ native   │ built-in  │ embedded │          │ ไม่ต้อง  │
│                │          │           │          │          │ server   │
└─────────────────┴──────────┴───────────┴──────────┴──────────┴──────────┘

ทำไม Hybrid Search (BM25 + Vector) ดีกว่า:

  Vector Search อย่างเดียว:
    "ลาพักร้อน" → หา vectors ที่ใกล้เคียง
    ✅ เข้าใจ "หยุดพัก" = "ลาพักร้อน" (semantic)
    ❌ ค้นคำเฉพาะทาง/ชื่อเฉพาะได้ไม่ดี

  BM25 (Full-Text Search) อย่างเดียว:
    "ลาพักร้อน" → หาคำที่ตรงกัน
    ✅ ค้นคำเฉพาะเจาะจงได้แม่น
    ❌ ไม่เข้าใจ synonyms

  Hybrid (BM25 + Vector):
    รวมทั้งสอง → จัดอันดับรวมกัน
    ✅ ได้ทั้ง semantic + exact match
    ✅ แม่นยำกว่าอย่างมีนัยสำคัญ
    ✅ งานวิจัยปี 2024-2025 ยืนยันว่าดีกว่า vector อย่างเดียว

แนะนำ:
  ง่ายสุด:     Supabase (managed pgvector, free tier เริ่มต้น)
  คุ้มค่าสุด:   ParadeDB self-host (pgvector + BM25 ครบ)
  Prototype:  ChromaDB / LanceDB (ไม่ต้อง setup)
```

### 3.4 Framework vs DIY

```
┌──────────────┬──────────────────────────────────────────────────┐
│              │ ใช้เมื่อ                                         │
├──────────────┼──────────────────────────────────────────────────┤
│ LangChain    │ • ต้องการ agent ที่ซับซ้อน multi-tool           │
│              │ • ต้อง chain หลาย LLM calls                     │
│              │ • ต้องการ built-in integrations เยอะ             │
│              │ • ทีมคุ้นเคยอยู่แล้ว                              │
│              │                                                  │
│              │ ❌ อย่าใช้ถ้า: use case ง่ายๆ (overhead เยอะ)     │
│              │ ❌ Dependencies เยอะ break บ่อย                   │
│              │ ❌ Abstraction หนาเกิน debug ยาก                 │
├──────────────┼──────────────────────────────────────────────────┤
│ LlamaIndex   │ • Focus เรื่อง RAG โดยเฉพาะ                     │
│              │ • มี document type หลายแบบ                      │
│              │ • ต้องการ advanced retrieval (re-rank, fusion)   │
│              │                                                  │
│              │ ❌ อย่าใช้ถ้า: ต้องการ agent ซับซ้อน              │
├──────────────┼──────────────────────────────────────────────────┤
│ Google ADK   │ • ใช้ Gemini + Google Cloud อยู่แล้ว             │
│              │ • ต้องการ deploy บน Vertex AI                    │
│              │                                                  │
│              │ ❌ อย่าใช้ถ้า: ไม่ได้อยู่ใน Google ecosystem     │
├──────────────┼──────────────────────────────────────────────────┤
│ Dify         │ • ต้องการ low-code / มี UI จัดการ               │
│              │ • Non-dev ต้องจัดการ knowledge base              │
│              │ • Prototype เร็ว                                 │
│              │                                                  │
│              │ ❌ อย่าใช้ถ้า: ต้อง customize ลึกมาก             │
├──────────────┼──────────────────────────────────────────────────┤
│ DIY          │ • Use case ตรงไปตรงมา (Q&A, search)             │
│  (ไม่ใช้     │ • ต้องการ optimize cost/latency สุดๆ            │
│   framework) │ • ต้องการเข้าใจทุก component                     │
│              │ • ไม่ต้องการ dependencies เยอะ                   │
│              │                                                  │
│              │ ❌ อย่าใช้ถ้า: ต้อง multi-agent ซับซ้อน          │
│              │ ❌ ทีมไม่มีเวลาเขียนเอง                          │
└──────────────┴──────────────────────────────────────────────────┘

สำหรับ Sellsuki Agent (internal Q&A + search):
  → use case ตรงไปตรงมา
  → DIY หรือ LlamaIndex เหมาะสุด
  → ถ้าต้องการ low-code → Dify
```

### 3.5 Hosting

```
┌──────────────┬──────────┬──────────┬───────────┬──────────────────┐
│ Provider     │ ราคา/เดือน│ Region   │ ได้อะไร    │ หมายเหตุ          │
├──────────────┼──────────┼──────────┼───────────┼──────────────────┤
│ Hetzner      │ $4-12    │ EU       │ 2C/4G/40G │ ถูกสุด คุ้มสุด    │
│              │          │ (Finland)│ SSD       │ ใกล้ Groq EU     │
├──────────────┼──────────┼──────────┼───────────┼──────────────────┤
│ DigitalOcean │ $6-24    │ SG/US/EU │ 1C/2G →   │ มี SG region     │
│              │          │          │ 4C/8G     │ ใกล้ไทย          │
├──────────────┼──────────┼──────────┼───────────┼──────────────────┤
│ Vultr        │ $5-20    │ SG/JP/US │ 1C/1G →   │ มี Asia region   │
│              │          │          │ 4C/8G     │                  │
├──────────────┼──────────┼──────────┼───────────┼──────────────────┤
│ Google       │ $5-30    │ SG/TW/   │ Auto-scale│ ดี scale auto    │
│ Cloud Run    │ (usage)  │ JP       │           │ แต่ cold start    │
├──────────────┼──────────┼──────────┼───────────┼──────────────────┤
│ Railway      │ $5-20    │ US/EU    │ Auto-scale│ ง่ายสุด git push │
├──────────────┼──────────┼──────────┼───────────┼──────────────────┤
│ Fly.io       │ $5-15    │ SG/HK/   │ Edge      │ Deploy ใกล้ user │
│              │          │ JP       │ computing │                  │
└──────────────┴──────────┴──────────┴───────────┴──────────────────┘

Co-location Strategy (สำคัญมาก):
  คอขวดของ RAG = Database Latency
  → App + Database + Cache ต้องอยู่ server/region เดียวกัน
  → ลด network hop = ลด latency 50-80%
  
  ถ้าใช้ Groq (EU server):
    → Hetzner Finland = latency ต่ำสุด
    → App + ParadeDB + Dragonfly อยู่ server เดียว
    → Groq API อยู่ EU ใกล้กัน

  ถ้า user อยู่ไทย + ใช้ Gemini:
    → DigitalOcean/Vultr Singapore
    → Google Gemini Asia = latency ต่ำ
```

### 3.6 Cache Layer

```
┌──────────────┬──────────────────────────────────────────────────┐
│ Cache Type   │ ทำอะไร                                          │
├──────────────┼──────────────────────────────────────────────────┤
│ Semantic     │ เก็บ embedding ของ prompt → ถ้า prompt ใหม่      │
│ Cache        │ cosine similarity > 0.95 กับ prompt เก่า        │
│              │ → ใช้คำตอบเก่าเลย ไม่ต้องเรียก LLM              │
│              │                                                  │
│              │ ประหยัด: 40-60% ของ LLM cost                    │
│              │ เหมาะ: คำถามซ้ำๆ (FAQ-style)                    │
│              │                                                  │
│              │ ⚠️ ต้องมีปุ่ม "ข้ามcache" ให้ user               │
│              │    กรณีคำตอบไม่ตรง                               │
├──────────────┼──────────────────────────────────────────────────┤
│ Embedding    │ เก็บ embedding ที่เคยคำนวณแล้ว                   │
│ Cache        │ prompt เดิม → ไม่ต้องเรียก embedding API ซ้ำ     │
│              │                                                  │
│              │ ประหยัด: embedding cost ทั้งหมดสำหรับ query ซ้ำ   │
├──────────────┼──────────────────────────────────────────────────┤
│ DB Query     │ Cache ผลลัพธ์ vector search                      │
│ Cache        │ query เดิม → ไม่ต้อง search DB ซ้ำ               │
│              │                                                  │
│              │ ประหยัด: DB latency (ลด 50-100ms ต่อ request)    │
├──────────────┼──────────────────────────────────────────────────┤
│ Summarized   │ ตอน index → ให้ AI สรุปแต่ละ chunk ไว้           │
│ Content      │ ส่ง summary เข้า LLM แทน full content            │
│              │                                                  │
│              │ ประหยัด: 30-50% ของ token count ต่อ request      │
│              │ = ลด cost + ลด latency                           │
└──────────────┴──────────────────────────────────────────────────┘

ใช้ Dragonfly (Redis-compatible) เป็น cache layer:
  - เร็วกว่า Redis 25x ในบาง workload
  - Drop-in replacement (ใช้ redis client เดิมได้)
  - Memory efficient กว่า
  - Single binary, deploy ง่าย
```

---

## 4. Cost Optimization Playbook

### 4.1 Techniques ทั้งหมด เรียงตาม Impact

```
┌────┬───────────────────────┬──────────┬───────────┬──────────────┐
│ #  │ Technique             │ ลด Cost  │ Effort    │ ควรทำเมื่อ    │
├────┼───────────────────────┼──────────┼───────────┼──────────────┤
│ 1  │ Semantic Caching      │ 40-60%   │ ★★★☆☆    │ เสมอ (must)  │
├────┼───────────────────────┼──────────┼───────────┼──────────────┤
│ 2  │ ใช้ Groq/Cerebras     │ 80-90%   │ ★★☆☆☆    │ เสมอ (must)  │
│    │ แทน OpenAI            │ vs GPT-4o│           │              │
├────┼───────────────────────┼──────────┼───────────┼──────────────┤
│ 3  │ Free embedding model  │ 100%     │ ★★★☆☆    │ cost-conscious│
│    │ (Gemini/self-host)    │ embed    │           │              │
├────┼───────────────────────┼──────────┼───────────┼──────────────┤
│ 4  │ Summarized content    │ 30-50%   │ ★★★★☆    │ docs ยาวๆ    │
│    │ (ลด token count)      │ tokens   │           │              │
├────┼───────────────────────┼──────────┼───────────┼──────────────┤
│ 5  │ Flex mode + retry     │ 20-40%   │ ★★☆☆☆    │ non-realtime │
│    │                       │ LLM cost │           │              │
├────┼───────────────────────┼──────────┼───────────┼──────────────┤
│ 6  │ PoW + rate limit      │ ลด abuse │ ★★★☆☆    │ public API   │
│    │                       │ 80%+     │           │              │
├────┼───────────────────────┼──────────┼───────────┼──────────────┤
│ 7  │ Tool call แทน         │ 20-40%   │ ★★★★☆    │ context ใหญ่  │
│    │ stuff all context     │ tokens   │           │              │
├────┼───────────────────────┼──────────┼───────────┼──────────────┤
│ 8  │ Diff-based indexing   │ 70-90%   │ ★★★★☆    │ docs update  │
│    │                       │ embed    │           │ บ่อย          │
├────┼───────────────────────┼──────────┼───────────┼──────────────┤
│ 9  │ Embedding cache       │ 30-50%   │ ★★☆☆☆    │ เสมอ         │
│    │                       │ embed    │           │              │
├────┼───────────────────────┼──────────┼───────────┼──────────────┤
│ 10 │ DB query cache        │ ลด       │ ★★☆☆☆    │ เสมอ         │
│    │                       │ latency  │           │              │
├────┼───────────────────────┼──────────┼───────────┼──────────────┤
│ 11 │ ไม่ใช้ re-ranking     │ ลด 1     │ ★☆☆☆☆    │ docs น้อย    │
│    │                       │ API call │ (ไม่ทำ)   │ (< 100 pages)│
├────┼───────────────────────┼──────────┼───────────┼──────────────┤
│ 12 │ Tiered model          │ 50-70%   │ ★★★☆☆    │ mixed        │
│    │ (ง่าย→ถูก, ยาก→แพง)   │ LLM cost │           │ complexity   │
└────┴───────────────────────┴──────────┴───────────┴──────────────┘
```

### 4.2 Technique: Tool Call แทน Stuff All Context

```
Plan v1 (stuff ทุกอย่าง):
  User ถาม "ลาพักร้อนกี่วัน?"
  → search 5 chunks
  → ยัดทั้ง 5 chunks เข้า prompt (อาจ 3000+ tokens)
  → LLM อ่านทั้งหมด แล้วตอบ
  → จ่ายค่า token ทั้งหมด

Plan v2 (tool call):
  User ถาม "ลาพักร้อนกี่วัน?"
  → ส่งแค่คำถาม + รายการ tools ที่มี
  → LLM ตัดสินใจเอง: "ต้อง search เรื่องลา"
  → เรียก tool → ได้ chunk ที่เกี่ยวข้อง 1-2 อัน
  → LLM อ่านแค่ที่จำเป็น แล้วตอบ
  → จ่าย token น้อยกว่า

  ข้อดี: LLM เลือก context เอง = แม่นกว่า + ถูกกว่า
  ข้อเสีย: อาจ loop หลายรอบ (แต่ Groq เร็วมาก ไม่ค่อยรู้สึก)
```

### 4.3 Technique: Diff-based Incremental Indexing

```
Plan v1 (re-index ทั้งหมด):
  เอกสาร update → ลบ embeddings ทั้งหมด → สร้างใหม่
  100 pages → 100 pages ต้อง embed ใหม่
  เสีย embedding cost ทุกครั้ง

Plan v2 (diff-based):
  เอกสาร update → เทียบ hash ของแต่ละ chunk
  100 pages แก้ 3 pages → embed ใหม่แค่ 3 chunks
  ประหยัด embedding cost 97%

  วิธีทำ:
  1. ตอน index เก็บ content_hash (SHA256) ของแต่ละ chunk
  2. ตอน re-index → hash ใหม่ vs hash เก่า
  3. เฉพาะ hash ที่เปลี่ยน → embed ใหม่ + update DB
  4. hash ที่หายไป → delete จาก DB
  5. hash ที่เหมือนเดิม → ข้าม
```

```python
import hashlib

def incremental_index(new_chunks: list[dict]):
    """Index เฉพาะ chunks ที่เปลี่ยน"""
    
    # ดึง hash เก่าจาก DB
    cur.execute("SELECT content_hash, id FROM documents WHERE source = %s", [source])
    existing = {row[0]: row[1] for row in cur.fetchall()}
    
    to_insert = []
    to_delete = set(existing.keys())  # สมมติลบหมดก่อน
    
    for chunk in new_chunks:
        content_hash = hashlib.sha256(chunk["content"].encode()).hexdigest()
        
        if content_hash in existing:
            # ไม่เปลี่ยน → ข้าม + อย่าลบ
            to_delete.discard(content_hash)
        else:
            # ใหม่ หรือ เปลี่ยน → ต้อง embed
            to_insert.append(chunk)
            to_delete.discard(content_hash)
    
    # ลบ chunks ที่หายไป
    for old_hash in to_delete:
        cur.execute("DELETE FROM documents WHERE id = %s", [existing[old_hash]])
    
    # Insert chunks ใหม่
    if to_insert:
        embeddings = get_embeddings_batch([c["content"] for c in to_insert])
        insert_chunks_with_embeddings(to_insert, embeddings)
    
    print(f"Inserted: {len(to_insert)}, Deleted: {len(to_delete)}, "
          f"Unchanged: {len(new_chunks) - len(to_insert)}")
```

### 4.4 Technique: Summarized Content

```python
def index_with_summary(chunk: dict) -> dict:
    """ตอน index สร้าง summary ไว้ด้วย — ลด tokens ตอน query"""
    
    # สร้าง summary (ทำตอน index ครั้งเดียว)
    summary_response = llm_client.chat.completions.create(
        model="gpt-4o-mini",  # ใช้ model ถูกตอน index ได้
        messages=[
            {"role": "system", "content": "สรุปข้อมูลนี้ให้สั้นกระชับ เก็บ key facts ทั้งหมด ภาษาไทย 2-3 ประโยค"},
            {"role": "user", "content": chunk["content"]}
        ],
        max_tokens=200,
    )
    
    chunk["summary"] = summary_response.choices[0].message.content
    # เก็บทั้ง content (full) และ summary (short) ลง DB
    return chunk

# ตอน query:
# - ส่ง summary เข้า LLM (ลด tokens 50-70%)
# - ถ้า LLM ต้องการ detail เพิ่ม → เรียก tool ดึง full content
```

### 4.5 Cost Comparison Table

```
สมมติ 1,000 requests/เดือน, เฉลี่ย 500 tokens/request

Plan v1 (ไม่ optimize):
  LLM (GPT-4o-mini):     1000 req × 500 tok = $0.30 + $0.30 = ~$0.60
  Embedding (OpenAI):     1000 req × query embed = ~$0.10
  Embedding (index):      ทุกครั้ง re-index = ~$2.00
  Database (Cloud SQL):   $25/เดือน
  Hosting (Cloud Run):    $10/เดือน
  ──────────────────────
  รวม: ~$38/เดือน

Plan v2 Strategy B (balanced):
  LLM (Groq):            600 req (40% cached) × 500 tok = ~$0.05
  Embedding (Gemini):     Free tier
  Embedding (index):      Diff-based = ~$0.10
  Database (self-host):   $0 (อยู่ใน server)
  Cache (Dragonfly):      $0 (อยู่ใน server)
  Hosting (Hetzner):      $12/เดือน
  ──────────────────────
  รวม: ~$12-15/เดือน (ลด 60%)

Plan v2 Strategy C (aggressive):
  LLM (Groq flex):       400 req (60% cached) = ~$0.03
  Embedding:             Self-host BGE-M3 = $0
  Database:              $0 (co-located)
  Cache:                 $0 (co-located)
  Hosting (Hetzner):     $8/เดือน (smaller instance)
  ──────────────────────
  รวม: ~$8-12/เดือน (ลด 75%)
```

---

## 5. Architecture ใหม่

### Strategy B Architecture (แนะนำ)

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Hetzner VPS ($12/เดือน)                         │
│                     (App + DB + Cache co-located)                   │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                    Coolify (Self-host PaaS)                    │ │
│  │                                                                │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐│ │
│  │  │ Agent API    │  │ ParadeDB     │  │ Dragonfly            ││ │
│  │  │ (FastAPI)    │  │ (Postgres +  │  │ (Redis-compatible)   ││ │
│  │  │              │  │  pgvector +  │  │                      ││ │
│  │  │ • /chat      │  │  BM25)       │  │ • Semantic cache     ││ │
│  │  │ • /search    │◄─┤              │  │ • Embedding cache    ││ │
│  │  │ • /ingest    │  │ • vectors    │  │ • DB query cache     ││ │
│  │  │ • /policy    │  │ • BM25 index │  │ • Session store      ││ │
│  │  │ • webhooks   │  │ • metadata   │  │                      ││ │
│  │  └──────┬───────┘  └──────────────┘  └──────────────────────┘│ │
│  │         │                                                     │ │
│  │  ┌──────▼───────┐  ┌──────────────┐  ┌──────────────────────┐│ │
│  │  │ MCP Server   │  │ Cron Jobs    │  │ Webhook Handler      ││ │
│  │  │ (for Claude  │  │              │  │                      ││ │
│  │  │  Code/Cursor)│  │ • Diff index │  │ • LINE webhook       ││ │
│  │  │              │  │ • Digest     │  │ • Slack webhook      ││ │
│  │  │              │  │ • Alerts     │  │ • Doc repo webhook   ││ │
│  │  └──────────────┘  └──────────────┘  └──────────────────────┘│ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                     │
└────────────────┬──────────────────────────────────────┬─────────────┘
                 │                                      │
      ┌──────────▼──────────┐              ┌────────────▼────────────┐
      │ LLM Providers       │              │ Channels                │
      │                     │              │                         │
      │ Primary: Groq       │              │ • LINE Bot              │
      │ Fallback: Gemini    │              │ • Slack Bot             │
      │ Complex: GPT-4o-mini│              │ • Web Chat              │
      │                     │              │ • Claude Code (MCP)     │
      │ Embed: Gemini Free  │              │ • Cursor IDE (MCP)      │
      └─────────────────────┘              │ • API (third-party)     │
                                           └─────────────────────────┘
```

### Request Flow (ละเอียด)

```
User ส่งคำถาม "ลาพักร้อนกี่วัน?"
│
▼
[1] Anti-abuse Layer
    ├── Rate limit check (Dragonfly)
    ├── PoW challenge verify (ถ้า public)
    └── Turnstile verify (ถ้า web)
│
▼
[2] Semantic Cache Check
    ├── Embed คำถาม (check embedding cache ก่อน)
    ├── Search Dragonfly: cosine similarity กับ cached prompts
    ├── ถ้า similarity > 0.95 → return cached answer ✅ (จบ)
    └── ถ้า < 0.95 → continue
│
▼
[3] LLM Call (with tool definitions)
    ├── ส่งคำถาม + tool list ไป Groq
    ├── Groq ตัดสินใจ: "ต้อง search เรื่องลา"
    └── Return tool_call: search("นโยบายลาพักร้อน")
│
▼
[4] Hybrid Search (ParadeDB)
    ├── Check DB query cache (Dragonfly)
    ├── ถ้า cache hit → return cached results
    ├── ถ้า miss:
    │   ├── BM25 search (exact keyword match)
    │   ├── Vector search (cosine similarity)
    │   ├── Reciprocal Rank Fusion (รวม score)
    │   └── Return top 3-5 results
    └── Cache results ใน Dragonfly
│
▼
[5] LLM Generate Answer
    ├── ส่ง search results (summary version) ให้ Groq
    ├── Groq สรุปคำตอบ
    ├── ถ้าต้องการ detail → tool_call อีกรอบ (loop)
    └── Return final answer
│
▼
[6] Cache & Response
    ├── Cache: prompt embedding + answer ใน Dragonfly
    ├── Log: query + answer + sources + latency
    └── Return response to user
```

---

## 6. Implementation Plan v2

### Timeline Overview

```
Week 1:    Foundation — Server + DB + ทดสอบ LLM providers
Week 2:    Data Pipeline — Extract + Chunk + Hybrid Index
Week 3:    Agent Core — Search + LLM + Cache layers
Week 4:    Channels — LINE + Slack + Web + MCP
Week 5:    Optimize — Cache tuning + monitoring + load test
Week 6:    Launch — UAT + go-live + feedback loop
```

### Week 1: Foundation

```
Day 1-2: Setup Infrastructure
  □ เช่า Hetzner VPS (CX22: 2vCPU, 4GB RAM, $4.5/mo)
    หรือ DigitalOcean SG ถ้าต้องการ latency ต่ำจากไทย
  □ ติดตั้ง Coolify (self-host PaaS)
    curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
  □ Deploy ParadeDB via Coolify
    docker image: paradedb/paradedb:latest
  □ Deploy Dragonfly via Coolify
    docker image: docker.dragonflydb.io/dragonflydb/dragonfly
  □ Setup domain + SSL (agent.sellsuki.com)

Day 3-4: ทดสอบ LLM Providers
  □ สมัคร accounts: Groq, Google AI Studio, OpenAI
  □ ทดสอบแต่ละ provider ด้วยคำถามภาษาไทย 20 ข้อ
  □ วัด: TTFT, speed, quality, cost
  □ ตัดสินใจ primary/fallback model

Day 5: Setup Database Schema
  □ สร้าง tables + indexes (pgvector + BM25)
  □ ทดสอบ hybrid search ด้วย sample data
  □ Benchmark: query latency target < 50ms
```

```sql
-- ParadeDB Schema (pgvector + pg_search built-in)
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_search;

CREATE TABLE documents (
    id BIGSERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    summary TEXT,                        -- summarized version
    embedding vector(768),               -- 768 for Gemini / BGE-M3
    content_hash VARCHAR(64) NOT NULL,   -- SHA256 for diff indexing
    
    source VARCHAR(500),
    category VARCHAR(100),
    department VARCHAR(100),
    title VARCHAR(500),
    doc_type VARCHAR(50),
    last_updated TIMESTAMP,
    chunk_index INTEGER,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT NOW()
);

-- Vector index (HNSW)
CREATE INDEX idx_docs_embedding ON documents
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- BM25 index (ParadeDB pg_search)
CALL paradedb.create_bm25_index(
    index_name => 'idx_docs_bm25',
    table_name => 'documents',
    key_field => 'id',
    text_fields => paradedb.field('content') || paradedb.field('title') || paradedb.field('summary')
);

-- Metadata indexes
CREATE INDEX idx_docs_category ON documents(category);
CREATE INDEX idx_docs_hash ON documents(content_hash);
CREATE INDEX idx_docs_source ON documents(source);
```

### Week 2: Data Pipeline

```
Day 6-7: Document Extraction
  □ รวบรวมเอกสารทั้งหมด
  □ เขียน extractors (PDF, Docs, Markdown, CSV)
  □ Clean + add metadata

Day 8-9: Chunking Strategy
  □ Chunk by file → sub-chunk by headers
  □ สร้าง content_hash ทุก chunk
  □ สร้าง summary ทุก chunk

Day 10: Indexing Pipeline
  □ Embed ด้วย Gemini / BGE-M3
  □ Insert ลง ParadeDB
  □ ทดสอบ hybrid search กับ real data
  □ สร้าง cron: diff-based re-index
```

```python
# chunking strategy (file → header-based sub-chunks)
def chunk_document(filepath: str, source_meta: dict) -> list[dict]:
    """
    Strategy จาก Elysia case:
    1. แบ่ง chunk เป็น file ก่อน
    2. ซอย sub-section ผ่าน headers อีก
    3. เก็บ content_hash สำหรับ diff
    4. สร้าง summary สำหรับ context optimization
    """
    raw_text = extract_file(filepath)
    
    # Split by headers (Markdown-aware)
    sections = split_by_headers(raw_text)
    
    chunks = []
    for i, section in enumerate(sections):
        content = section["content"]
        
        # ถ้า section ยาวเกิน → recursive split
        if len(content) > 1500:
            sub_chunks = recursive_split(content, chunk_size=800, overlap=150)
        else:
            sub_chunks = [content]
        
        for j, sub in enumerate(sub_chunks):
            content_hash = hashlib.sha256(sub.encode()).hexdigest()
            chunks.append({
                "content": sub,
                "content_hash": content_hash,
                "title": section.get("header", source_meta["title"]),
                "metadata": {
                    **source_meta,
                    "section_header": section.get("header", ""),
                    "chunk_index": f"{i}.{j}",
                }
            })
    
    return chunks
```

### Week 3: Agent Core

```
Day 11-12: Search Engine
  □ Hybrid search: BM25 + Cosine
  □ Reciprocal Rank Fusion
  □ DB query cache layer

Day 13-14: LLM Integration
  □ Tool-based approach (LLM เลือก search เอง)
  □ Multi-provider fallback (Groq → Gemini → GPT-4o-mini)
  □ Streaming response (SSE)

Day 15: Cache Layers
  □ Semantic cache (embed + cosine in Dragonfly)
  □ Embedding cache
  □ Response cache
  □ "Skip cache" mechanism
```

```python
# Hybrid Search with Reciprocal Rank Fusion
async def hybrid_search(
    query: str,
    top_k: int = 5,
    category: str = None,
    alpha: float = 0.5,  # balance: 0=BM25 only, 1=vector only, 0.5=balanced
) -> list[dict]:
    """
    BM25 + Cosine Similarity → Reciprocal Rank Fusion
    """
    
    # Check DB query cache
    cache_key = f"dbq:{hashlib.md5(f'{query}:{category}:{top_k}'.encode()).hexdigest()}"
    cached = await dragonfly.get(cache_key)
    if cached:
        return json.loads(cached)
    
    # Get query embedding
    query_embedding = await get_embedding_cached(query)
    
    # BM25 Search (ParadeDB)
    bm25_sql = """
        SELECT id, content, title, summary, paradedb.score(id) as bm25_score
        FROM documents.search(
            query => paradedb.parse($1),
            limit_rows => $2
        )
    """
    if category:
        bm25_sql += f" WHERE category = '{category}'"
    
    bm25_results = await db.fetch(bm25_sql, query, top_k * 2)
    
    # Vector Search
    vec_sql = """
        SELECT id, content, title, summary,
               1 - (embedding <=> $1::vector) as vec_score
        FROM documents
    """
    if category:
        vec_sql += f" WHERE category = '{category}'"
    vec_sql += " ORDER BY embedding <=> $1::vector LIMIT $2"
    
    vec_results = await db.fetch(vec_sql, str(query_embedding), top_k * 2)
    
    # Reciprocal Rank Fusion
    k = 60  # constant
    scores = {}
    
    for rank, row in enumerate(bm25_results):
        doc_id = row["id"]
        scores[doc_id] = scores.get(doc_id, {"data": row, "score": 0})
        scores[doc_id]["score"] += (1 - alpha) * (1 / (k + rank + 1))
    
    for rank, row in enumerate(vec_results):
        doc_id = row["id"]
        scores[doc_id] = scores.get(doc_id, {"data": row, "score": 0})
        scores[doc_id]["score"] += alpha * (1 / (k + rank + 1))
    
    # Sort by combined score
    results = sorted(scores.values(), key=lambda x: x["score"], reverse=True)[:top_k]
    
    output = [{
        "id": r["data"]["id"],
        "content": r["data"]["content"],
        "summary": r["data"]["summary"],
        "title": r["data"]["title"],
        "score": round(r["score"], 4),
    } for r in results]
    
    # Cache result (TTL 5 min)
    await dragonfly.setex(cache_key, 300, json.dumps(output))
    
    return output
```

```python
# Semantic Cache
async def check_semantic_cache(query: str, threshold: float = 0.95) -> dict | None:
    """
    ถ้าคำถามคล้ายกับที่เคยถามมาก → ใช้คำตอบเก่า
    """
    query_embedding = await get_embedding_cached(query)
    
    # ดึง cached prompts ทั้งหมด (เก็บใน Dragonfly sorted set)
    cached_entries = await dragonfly.zrange("semantic_cache:entries", 0, -1, withscores=True)
    
    best_match = None
    best_similarity = 0
    
    for entry_key, _ in cached_entries:
        cached_data = json.loads(await dragonfly.get(f"semantic_cache:data:{entry_key}"))
        cached_embedding = cached_data["embedding"]
        
        similarity = cosine_similarity(query_embedding, cached_embedding)
        
        if similarity > threshold and similarity > best_similarity:
            best_similarity = similarity
            best_match = cached_data
    
    if best_match:
        return {
            "answer": best_match["answer"],
            "sources": best_match["sources"],
            "cached": True,
            "similarity": round(best_similarity, 4),
        }
    
    return None


# Multi-provider LLM with fallback
async def call_llm(messages: list, tools: list = None) -> dict:
    """Groq → Gemini → GPT-4o-mini fallback chain"""
    
    providers = [
        {"name": "groq", "model": "llama-3.3-70b-versatile", "client": groq_client},
        {"name": "gemini", "model": "gemini-2.0-flash", "client": gemini_client},
        {"name": "openai", "model": "gpt-4o-mini", "client": openai_client},
    ]
    
    for provider in providers:
        try:
            response = await provider["client"].chat.completions.create(
                model=provider["model"],
                messages=messages,
                tools=tools,
                temperature=0,
                timeout=15,
            )
            return {"provider": provider["name"], "response": response}
        except Exception as e:
            print(f"Provider {provider['name']} failed: {e}")
            continue
    
    raise Exception("All LLM providers failed")
```

### Week 4: Channels

```
Day 16-17: LINE Bot + Slack Bot
  □ LINE webhook handler + Flex Message
  □ Slack event handler + slash command
  □ ทั้งสองเรียก /api/v1/chat

Day 18: Web Chat + MCP Server
  □ Streamlit/Chainlit สำหรับ internal web chat
  □ MCP Server สำหรับ Claude Code + Cursor

Day 19-20: API + Anti-abuse
  □ API key management
  □ Rate limiting (Dragonfly-based)
  □ PoW challenge (สำหรับ public endpoints)
  □ Swagger docs (/docs)
```

### Week 5: Optimize

```
Day 21-22: Cache Tuning
  □ วัด cache hit rate (target > 40%)
  □ ปรับ semantic cache threshold
  □ ปรับ DB query cache TTL
  □ Monitor memory usage

Day 23-24: Observability
  □ Structured logging (query, answer, latency, provider, cache_hit)
  □ เก็บใน PostgreSQL table (query_logs)
  □ สร้าง dashboard ง่ายๆ (Streamlit)
  □ Alert: error rate > 5%, latency > 2s

Day 25: Load Testing
  □ ทดสอบ 50 concurrent users
  □ ทดสอบ 1000 requests
  □ ระบุ bottleneck + fix
  □ วัด P50, P95, P99 latency
```

```python
# Observability: Structured Logging
import time
from datetime import datetime

async def log_query(
    query: str,
    answer: str,
    sources: list,
    latency_ms: float,
    provider: str,
    cache_hit: bool,
    session_id: str,
    channel: str,  # line, slack, web, mcp, api
):
    await db.execute("""
        INSERT INTO query_logs 
        (query, answer, sources, latency_ms, provider, cache_hit, 
         session_id, channel, created_at)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
    """, query, answer, json.dumps(sources), latency_ms, 
         provider, cache_hit, session_id, channel, datetime.utcnow())

# ตัวอย่าง dashboard query
# คำถามที่ถามบ่อย:
#   SELECT query, COUNT(*) FROM query_logs GROUP BY query ORDER BY count DESC LIMIT 10
# 
# Cache hit rate:
#   SELECT cache_hit, COUNT(*) FROM query_logs GROUP BY cache_hit
#
# Average latency per provider:
#   SELECT provider, AVG(latency_ms) FROM query_logs GROUP BY provider
#
# คำถามที่ตอบไม่ได้ (answer contains "ไม่พบข้อมูล"):
#   SELECT query, COUNT(*) FROM query_logs WHERE answer LIKE '%ไม่พบข้อมูล%' GROUP BY query
```

### Week 6: Launch

```
Day 26-27: UAT
  □ ให้ 10-20 คนทดสอบจริง
  □ รวบรวม feedback
  □ ปรับ system prompt ตาม feedback
  □ เพิ่มเอกสารที่ขาด (จาก unanswered queries)

Day 28-29: Fine-tune
  □ ปรับ chunk size ถ้าจำเป็น
  □ ปรับ cache threshold
  □ ปรับ LLM prompt
  □ เพิ่ม Quick Reply / Rich Menu (LINE)

Day 30: Go Live
  □ ประกาศทั้งบริษัท
  □ สร้าง #suki-bot-feedback channel ใน Slack
  □ Schedule: weekly review query logs
  □ Schedule: monthly re-evaluate costs
```

---

## 7. เปรียบเทียบ Plan v1 vs v2

```
┌────────────────────┬────────────────────┬────────────────────────────┐
│                    │ Plan v1            │ Plan v2 (Strategy B)       │
├────────────────────┼────────────────────┼────────────────────────────┤
│ LLM                │ GPT-4o-mini only   │ Groq primary + fallback    │
│ Cost/month (LLM)   │ $5-20              │ $0.50-5                    │
├────────────────────┼────────────────────┼────────────────────────────┤
│ Embedding          │ OpenAI ($0.02/1M)  │ Gemini Free / Self-host    │
│ Cost/month         │ $1-5               │ $0                         │
├────────────────────┼────────────────────┼────────────────────────────┤
│ Database           │ Cloud SQL ($25)    │ ParadeDB self-host ($0)    │
├────────────────────┼────────────────────┼────────────────────────────┤
│ Search             │ Vector only        │ Hybrid: BM25 + Vector      │
│ Quality            │ ★★★☆☆             │ ★★★★★                     │
├────────────────────┼────────────────────┼────────────────────────────┤
│ Hosting            │ Cloud Run ($10)    │ Hetzner ($12, all-in-one)  │
├────────────────────┼────────────────────┼────────────────────────────┤
│ Cache              │ ❌ ไม่มี           │ ✅ 4 layers                │
│ Cache Hit Rate     │ 0%                 │ 40-60%                     │
├────────────────────┼────────────────────┼────────────────────────────┤
│ Indexing           │ Full re-index      │ Diff-based incremental     │
├────────────────────┼────────────────────┼────────────────────────────┤
│ Framework          │ LangChain          │ DIY (lighter, faster)      │
├────────────────────┼────────────────────┼────────────────────────────┤
│ Anti-abuse         │ ❌ ไม่มี           │ ✅ PoW + rate limit        │
├────────────────────┼────────────────────┼────────────────────────────┤
│ Observability      │ Basic logs         │ Structured + analytics     │
├────────────────────┼────────────────────┼────────────────────────────┤
│ Latency (P50)      │ 500-800ms          │ 150-300ms                  │
├────────────────────┼────────────────────┼────────────────────────────┤
│ Total Cost         │ $40-70/mo          │ $12-25/mo                  │
├────────────────────┼────────────────────┼────────────────────────────┤
│ Channels           │ LINE, Slack, Web   │ + Claude Code, Cursor,     │
│                    │                    │   MCP, Automation, API     │
├────────────────────┼────────────────────┼────────────────────────────┤
│ Setup Time         │ 6 weeks            │ 6 weeks (same)             │
│ Complexity         │ ★★★☆☆             │ ★★★★☆ (worth it)          │
└────────────────────┴────────────────────┴────────────────────────────┘
```

### Final Recommendation

```
สำหรับ Sellsuki:

เริ่มด้วย Strategy B (Balanced) เพราะ:
  1. Cost ลด 60% จาก Plan v1 ($12-25 vs $50-70)
  2. Latency ดีขึ้น 2-3x (co-location + cache)
  3. Search แม่นขึ้นมาก (hybrid BM25 + vector)
  4. ต่อกับ dev tools ได้ (MCP)
  5. Scale ได้ชัดเจน
  6. ไม่ hardcore เกินไป ทีม maintain ได้

ถ้าอนาคตอยาก optimize เพิ่ม → ค่อย shift ไป Strategy C
(เพิ่ม semantic cache, self-host embedding, flex mode)

Stack สรุป:
  Server:    Hetzner CX22 ($12) + Coolify
  DB:        ParadeDB (pgvector + BM25)
  Cache:     Dragonfly
  LLM:       Groq Llama 3.3 70B → fallback Gemini Flash
  Embedding: Gemini text-embedding-004 (free)
  Framework: DIY (FastAPI + httpx)
  Channels:  LINE + Slack + Web Chat + MCP
  Deploy:    Coolify (git push → auto deploy)
```
