# Cost & Models — ค่าใช้จ่าย, เลือก Model ยังไง

## ทำความเข้าใจ Token ก่อน

```
Token ≈ คำ (word) หรือส่วนหนึ่งของคำ
  1 token ≈ 4 ตัวอักษรภาษาอังกฤษ
  1 token ≈ 1-2 ตัวอักษรภาษาไทย/จีน/ญี่ปุ่น

ตัวอย่าง:
  "Hello World"                → ~2 tokens
  "สวัสดีครับ"                 → ~6-8 tokens
  หน้า A4 เต็ม (500 คำ)       → ~700 tokens
  PDF 10 หน้า                 → ~7,000 tokens
```

---

## ค่าใช้จ่ายเกิดขึ้น 2 ส่วน

```
┌──────────────────────────────────────────────────────┐
│  1. Embedding Cost (ตอน Ingest เอกสาร)               │
│     = แปลง text → vector                             │
│     = จ่ายครั้งเดียวต่อเอกสาร                        │
│                                                      │
│  2. LLM Cost (ตอน Chat)                              │
│     = input: query + retrieved chunks                │
│     = output: คำตอบที่ generate                      │
│     = จ่ายทุกครั้งที่ chat                           │
└──────────────────────────────────────────────────────┘
```

---

## Embedding Cost (Ingestion)

### OpenAI Embedding Pricing (ราคาโดยประมาณ):

| Model | ราคาต่อ 1M tokens | Dimension |
|-------|------------------|-----------|
| text-embedding-3-small | $0.02 | 1536 |
| text-embedding-3-large | $0.13 | 3072 |
| text-embedding-ada-002 | $0.10 | 1536 |

### ตัวอย่างการคำนวณ Embedding Cost:

```
สมมติ: Ingest เอกสาร 100 ไฟล์, เฉลี่ย 20 หน้าต่อไฟล์

tokens ต่อหน้า: ~700 tokens
tokens รวม: 100 files × 20 pages × 700 = 1,400,000 tokens = 1.4M tokens

ค่าใช้จ่าย (text-embedding-3-small @ $0.02/1M):
  = 1.4 × $0.02 = $0.028 ≈ ~1 บาท

ค่าใช้จ่าย (text-embedding-3-large @ $0.13/1M):
  = 1.4 × $0.13 = $0.182 ≈ ~7 บาท

→ Embedding cost ถูกมาก แทบไม่มีนัยสำคัญ
```

---

## LLM Cost (Chat)

### OpenAI Pricing (ราคาโดยประมาณ):

| Model | Input ($/1M) | Output ($/1M) | Speed |
|-------|-------------|--------------|-------|
| gpt-4.1-nano | $0.10 | $0.40 | เร็วมาก |
| gpt-4o-mini | $0.15 | $0.60 | เร็วมาก |
| gpt-4.1-mini | $0.40 | $1.60 | เร็ว |
| gpt-4o | $2.50 | $10.00 | ปานกลาง |
| gpt-4.1 | $2.00 | $8.00 | ปานกลาง |
| o4-mini | $1.10 | $4.40 | ช้า (reasoning) |
| gpt-5 | สูงมาก | สูงมาก | ปานกลาง |

### Anthropic Claude Pricing (ราคาโดยประมาณ):

| Model | Input ($/1M) | Output ($/1M) |
|-------|-------------|--------------|
| claude-haiku-4-5 | $0.80 | $4.00 |
| claude-sonnet-4-5 | $3.00 | $15.00 |
| claude-opus-4-5 | $15.00 | $75.00 |

### ตัวอย่างการคำนวณ Chat Cost ต่อเดือน:

```
สมมติ: ทีมงาน 50 คน, ถามเฉลี่ย 20 คำถามต่อคนต่อวัน

ต่อ Query:
  - System prompt: ~500 tokens
  - Retrieved chunks (5 chunks × 200 tokens): ~1,000 tokens
  - User question: ~50 tokens
  - Input รวม: ~1,550 tokens
  - Output (คำตอบ): ~300 tokens

ต่อวัน:
  50 คน × 20 queries = 1,000 queries/day
  Input: 1,000 × 1,550 = 1,550,000 tokens = 1.55M
  Output: 1,000 × 300 = 300,000 tokens = 0.3M

ต่อเดือน (22 working days):
  Input: 1.55M × 22 = 34.1M tokens
  Output: 0.3M × 22 = 6.6M tokens

ค่าใช้จ่าย:
  gpt-4o-mini:
    Input: 34.1 × $0.15 = $5.12
    Output: 6.6 × $0.60 = $3.96
    รวม: ~$9/เดือน ≈ ~315 บาท/เดือน ✅ ถูกมาก

  gpt-4o:
    Input: 34.1 × $2.50 = $85.25
    Output: 6.6 × $10.00 = $66.00
    รวม: ~$151/เดือน ≈ ~5,300 บาท/เดือน

  Ollama (Local):
    $0/เดือน (ใช้ hardware ของตัวเอง) ✅
```

---

## เปรียบเทียบ Provider ทั้งหมด

### สำหรับ Chat LLM:

| | OpenAI | Anthropic | WatsonX | Ollama |
|--|--------|-----------|---------|--------|
| **คุณภาพ** | ดีมาก | ดีมาก | ดี | ปานกลาง |
| **ราคา** | ปานกลาง | แพง | ปานกลาง | ฟรี |
| **Privacy** | ข้อมูลออกนอก | ข้อมูลออกนอก | On-prem option | ไม่ออกเลย |
| **ภาษาไทย** | ดีมาก | ดีมาก | พอใช้ | ขึ้นกับ model |
| **Setup** | ง่ายมาก | ง่ายมาก | ปานกลาง | ต้อง GPU |
| **Rate Limit** | มี | มี | มี | ไม่มี |
| **Reliability** | สูง | สูง | สูง | ขึ้นกับ hardware |

### สำหรับ Embedding:

| | OpenAI | WatsonX | Ollama |
|--|--------|---------|--------|
| **คุณภาพ** | ดีมาก | ดี | ดี |
| **ราคา** | ถูกมาก | ปานกลาง | ฟรี |
| **Privacy** | ข้อมูลออก | ขึ้นกับ config | ไม่ออกเลย |

---

## คำแนะนำตาม Use Case

### Startup / ทดสอบ:
```yaml
# ถูกที่สุด ดีพอสำหรับ POC
agent:
  provider: openai
  model: gpt-4o-mini

knowledge:
  embedding_model: text-embedding-3-small

# ค่าใช้จ่าย: ~$5-10/เดือน สำหรับทีมเล็ก
```

### องค์กรกลาง (ข้อมูลไม่ sensitive):
```yaml
# Balance ระหว่าง cost และ quality
agent:
  provider: openai
  model: gpt-4o

knowledge:
  embedding_model: text-embedding-3-small

# ค่าใช้จ่าย: ~$50-200/เดือน แล้วแต่ usage
```

### องค์กรที่ข้อมูล sensitive / ต้องการ Privacy สูงสุด:
```yaml
# ข้อมูลไม่ออกจาก server เลย
agent:
  provider: ollama
  model: llama3.1   # หรือ qwen2.5 สำหรับภาษาไทย

knowledge:
  embedding_model: nomic-embed-text  # Ollama embedding

# ต้องการ: GPU server (แนะนำ NVIDIA RTX 3090+ หรือ A100)
# ค่าใช้จ่าย: $0 API แต่ค่า hardware / cloud GPU
```

### Enterprise (IBM WatsonX):
```yaml
agent:
  provider: watsonx
  model: ibm/granite-3-8b-instruct

knowledge:
  embedding_model: ibm/slate-125m-english-rtrvr

# ข้อดี: on-premises option, enterprise support, compliance
```

---

## Ollama Setup (Local LLM ฟรี)

```bash
# 1. ติดตั้ง Ollama
curl -fsSL https://ollama.ai/install.sh | sh

# 2. Pull models
ollama pull llama3.1          # LLM หลัก (8B params, ~5GB)
ollama pull llama3.1:70b      # LLM คุณภาพสูง (70B params, ~40GB, ต้องการ RAM มาก)
ollama pull qwen2.5           # ภาษาไทย/จีนดีกว่า llama
ollama pull nomic-embed-text  # Embedding model

# 3. ตรวจสอบ
curl http://localhost:11434/api/tags
# → รายการ models ที่ pull แล้ว

# 4. ตั้งค่าใน OpenRAG .env
OLLAMA_ENDPOINT=http://host.docker.internal:11434

# 5. ตั้งค่าใน config.yaml
agent:
  provider: ollama
  model: llama3.1

knowledge:
  embedding_model: nomic-embed-text
```

**Hardware Requirements สำหรับ Ollama:**
```
7B model (llama3.1):
  RAM: 8GB minimum (16GB แนะนำ)
  GPU: ไม่จำเป็น แต่ช้ามากถ้าใช้ CPU
  GPU VRAM: 6GB minimum (8GB แนะนำ)

13B model:
  GPU VRAM: 10GB+

70B model (คุณภาพดีสุด):
  GPU VRAM: 40GB+ (RTX 3090 × 2 หรือ A100)
  หรือ: CPU กับ RAM 64GB (ช้ามาก)
```

---

## Cost Optimization Tips

### 1. ใช้ Model เล็กสำหรับ Query ง่ายๆ
```python
# ใช้ gpt-4o-mini สำหรับ FAQ ทั่วไป
# เปลี่ยนเป็น gpt-4o เฉพาะ query ที่ซับซ้อน
# (ทำได้ผ่าน Langflow flow หรือ custom routing)
```

### 2. ลด Retrieved Chunks
```python
# ลด k จาก 10 → 5 = ลด input tokens 50%
# แต่ accuracy อาจลดลงเล็กน้อย
```

### 3. Optimize Chunk Size
```yaml
# Chunk เล็กลง = chunks per query น้อยลง = ค่าถูกลง
# แต่ต้อง balance กับ context quality
knowledge:
  chunk_size: 600      # ลดจาก 1000
  chunk_overlap: 100
```

### 4. Cache Embeddings
```
Embedding ถูกสร้างครั้งเดียวตอน ingest
ไม่มีค่าใช้จ่าย embedding ตอน query
→ embedding cost ถูกมากอยู่แล้ว ไม่ต้อง optimize มาก
```

### 5. ใช้ Ollama สำหรับ High Volume
```
ถ้า query > 10,000 ครั้ง/เดือน
→ Ollama (self-hosted) คุ้มกว่า OpenAI แน่นอน
```

---

## ตัวอย่าง Cost Calculator

```python
def estimate_monthly_cost(
    users: int,
    queries_per_user_per_day: int,
    working_days: int = 22,
    model: str = "gpt-4o-mini",
    avg_input_tokens: int = 1500,
    avg_output_tokens: int = 300
):
    pricing = {
        "gpt-4o-mini":  {"input": 0.15, "output": 0.60},
        "gpt-4o":       {"input": 2.50, "output": 10.00},
        "gpt-4.1":      {"input": 2.00, "output": 8.00},
        "gpt-4.1-mini": {"input": 0.40, "output": 1.60},
        "claude-haiku": {"input": 0.80, "output": 4.00},
        "claude-sonnet":{"input": 3.00, "output": 15.00},
    }

    total_queries = users * queries_per_user_per_day * working_days
    total_input_tokens = total_queries * avg_input_tokens / 1_000_000
    total_output_tokens = total_queries * avg_output_tokens / 1_000_000

    price = pricing.get(model, pricing["gpt-4o-mini"])
    cost_usd = (total_input_tokens * price["input"] +
                total_output_tokens * price["output"])
    cost_thb = cost_usd * 35  # approximate USD to THB

    return {
        "queries_per_month": total_queries,
        "cost_usd": round(cost_usd, 2),
        "cost_thb": round(cost_thb, 0)
    }

# ตัวอย่าง:
print(estimate_monthly_cost(users=50, queries_per_user_per_day=20, model="gpt-4o-mini"))
# → {'queries_per_month': 22000, 'cost_usd': 9.24, 'cost_thb': 323.0}

print(estimate_monthly_cost(users=50, queries_per_user_per_day=20, model="gpt-4o"))
# → {'queries_per_month': 22000, 'cost_usd': 148.5, 'cost_thb': 5197.0}
```

---

## สรุปการเลือก Model

```
ถามตัวเองก่อน:

1. ข้อมูลที่ ingest เป็น confidential ไหม?
   YES → ใช้ Ollama (local) เท่านั้น
   NO  → ใช้ OpenAI หรือ Claude ได้

2. Budget เท่าไร?
   น้อย → gpt-4o-mini หรือ Ollama
   มาก  → gpt-4o หรือ claude-sonnet

3. ภาษาไทยสำคัญแค่ไหน?
   สำคัญมาก → gpt-4o หรือ claude-sonnet (ภาษาไทยดีกว่า mini)
   พอใช้ได้  → gpt-4o-mini

4. Volume สูงแค่ไหน?
   < 5,000 queries/เดือน → API (OpenAI/Claude) ง่ายกว่า
   > 10,000 queries/เดือน → พิจารณา Ollama ถ้า hardware พร้อม

5. ต้องการ Enterprise support?
   YES → IBM WatsonX
```

---

*กลับไป: [advanced-config.md](./advanced-config.md)*
*ต่อไป: [document-best-practices.md](./document-best-practices.md)*
