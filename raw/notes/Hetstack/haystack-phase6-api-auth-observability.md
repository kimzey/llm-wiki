# 🌾 Haystack Deep Dive — Phase 6: API, Auth, Observability & Ecosystem

> **เป้าหมาย Phase 6:** ทำ Production-ready Stack ครบวงจร — API Layer, Authentication, Tracing, Monitoring
> และเข้าใจ Ecosystem ของ Haystack เทียบกับ LangChain

---

## 0. ภาพรวม: Haystack vs LangChain Ecosystem

```
┌─────────────────────────────────────────────────────────────────┐
│                    ECOSYSTEM COMPARISON                         │
├─────────────────────┬───────────────────────────────────────────┤
│   LangChain         │   Haystack                                │
├─────────────────────┼───────────────────────────────────────────┤
│ LangServe (API)     │ FastAPI (DIY) / Hayhooks ✅               │
│ LangSmith (Trace)   │ Langfuse ✅ / OpenTelemetry ✅            │
│ LangFuse (Monitor)  │ Langfuse ✅ (ใช้ร่วมกันได้)              │
│ LangGraph (Agent)   │ Agent built-in + Custom Loop              │
│ Hub (Prompt Mgmt)   │ deepset Cloud / Prompt YAML               │
│ Built-in Auth       │ ❌ (ใช้ Framework เอง)                    │
│ Guardrails          │ ✅ via Component                          │
└─────────────────────┴───────────────────────────────────────────┘

สรุปตรงๆ:
- LangChain: Ecosystem ครบกว่า "out of the box"
- Haystack:  Production pipeline แม่นยำกว่า แต่ต้อง wire เอง
             ข้อดีคือ "ไม่ lock-in" กับ tool ของตัวเอง
```

---

## 1. API Layer — ทำ REST API จาก Haystack Pipeline

### Option A: Hayhooks (Official — แนะนำ ⭐)

**Hayhooks** คือ official tool จาก deepset สำหรับ serve Haystack pipeline เป็น REST API

```bash
pip install hayhooks
```

**วิธีใช้:**

```
my_project/
├── pipelines/
│   ├── rag_pipeline.yaml         # pipeline ที่ serialize ไว้
│   └── indexing_pipeline.yaml
└── main.py
```

```bash
# รัน Hayhooks server
hayhooks run --pipelines-dir ./pipelines --host 0.0.0.0 --port 1416
```

Auto-generate REST API จาก pipeline:

```
POST /rag_pipeline/run
POST /indexing_pipeline/run
GET  /status
GET  /pipelines
```

```python
# เรียกใช้
import requests

response = requests.post("http://localhost:1416/rag_pipeline/run", json={
    "embedder": {"text": "Haystack คืออะไร?"},
    "prompt_builder": {"question": "Haystack คืออะไร?"}
})

print(response.json())
```

**Deploy ด้วย Docker:**

```dockerfile
FROM python:3.11-slim

RUN pip install haystack-ai hayhooks

COPY pipelines/ /app/pipelines/
WORKDIR /app

EXPOSE 1416
CMD ["hayhooks", "run", "--pipelines-dir", "/app/pipelines", "--host", "0.0.0.0"]
```

---

### Option B: FastAPI (Full Control)

สำหรับ production ที่ต้องการ custom logic, middleware, auth

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI, HTTPException, Depends, BackgroundTasks
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import yaml
from haystack import Pipeline

# ── State ────────────────────────────────────────────
pipelines: dict[str, Pipeline] = {}

@asynccontextmanager
async def lifespan(app: FastAPI):
    """โหลด Pipeline ตอน startup"""
    with open("pipelines/rag.yaml") as f:
        pipelines["rag"] = Pipeline.load(f)
    pipelines["rag"].warm_up()
    print("✅ Pipelines loaded")
    yield
    # cleanup ตอน shutdown
    print("🔴 Shutting down")

app = FastAPI(title="Haystack RAG API", lifespan=lifespan)

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# ── Models ────────────────────────────────────────────
class QueryRequest(BaseModel):
    question: str
    filters: dict = {}
    top_k: int = 5

class Source(BaseModel):
    content: str
    file_path: str
    score: float

class QueryResponse(BaseModel):
    answer: str
    sources: list[Source]
    latency_ms: float

# ── Routes ────────────────────────────────────────────
@app.post("/query", response_model=QueryResponse)
async def query(request: QueryRequest):
    import time
    start = time.time()

    try:
        result = pipelines["rag"].run({
            "embedder":       {"text": request.question},
            "retriever":      {"top_k": request.top_k, "filters": request.filters or None},
            "prompt_builder": {"question": request.question}
        })
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

    answer = result["llm"]["replies"][0]
    docs = result.get("retriever", {}).get("documents", [])

    return QueryResponse(
        answer=answer,
        sources=[
            Source(
                content=doc.content[:200],
                file_path=doc.meta.get("file_path", ""),
                score=doc.score or 0.0
            )
            for doc in docs
        ],
        latency_ms=round((time.time() - start) * 1000, 2)
    )

@app.get("/health")
async def health():
    return {"status": "ok", "pipelines": list(pipelines.keys())}
```

---

## 2. Authentication & Authorization

### 2.1 API Key Auth (Simple)

```python
from fastapi import Security
from fastapi.security.api_key import APIKeyHeader
import secrets

API_KEY_HEADER = APIKeyHeader(name="X-API-Key", auto_error=True)
VALID_KEYS = {"key-abc123", "key-xyz789"}  # จริงๆ เก็บใน DB หรือ env

async def verify_api_key(api_key: str = Security(API_KEY_HEADER)):
    if api_key not in VALID_KEYS:
        raise HTTPException(status_code=403, detail="Invalid API Key")
    return api_key

# ใส่ dependency
@app.post("/query", dependencies=[Depends(verify_api_key)])
async def query(request: QueryRequest):
    ...
```

### 2.2 JWT Auth (Production)

```bash
pip install python-jose[cryptography] passlib[bcrypt]
```

```python
from datetime import datetime, timedelta
from jose import JWTError, jwt
from passlib.context import CryptContext
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm

SECRET_KEY = "your-secret-key-min-32-chars"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 60

pwd_context = CryptContext(schemes=["bcrypt"])
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/token")

def create_access_token(data: dict, expires_delta: timedelta = None):
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode["exp"] = expire
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

async def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username = payload.get("sub")
        if not username:
            raise HTTPException(status_code=401, detail="Invalid token")
        return username
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")

# Login endpoint
@app.post("/auth/token")
async def login(form: OAuth2PasswordRequestForm = Depends()):
    # verify user จาก DB
    # ถ้า valid:
    token = create_access_token({"sub": form.username}, timedelta(minutes=60))
    return {"access_token": token, "token_type": "bearer"}

# Protected route
@app.post("/query")
async def query(request: QueryRequest, user: str = Depends(get_current_user)):
    ...
```

### 2.3 Role-based Access (RBAC)

```python
from enum import Enum

class Role(str, Enum):
    ADMIN = "admin"
    USER  = "user"
    READONLY = "readonly"

def require_role(*roles: Role):
    async def checker(user=Depends(get_current_user)):
        user_role = get_user_role(user)  # ดึงจาก DB
        if user_role not in roles:
            raise HTTPException(status_code=403, detail="Permission denied")
        return user
    return checker

# Admin เท่านั้น index เอกสารได้
@app.post("/index", dependencies=[Depends(require_role(Role.ADMIN))])
async def index_documents(files: list[str]):
    ...

# User ทั่วไป query ได้
@app.post("/query", dependencies=[Depends(require_role(Role.USER, Role.ADMIN))])
async def query(request: QueryRequest):
    ...
```

---

## 3. Observability — Tracing แบบ Production

### 3.1 OpenTelemetry (แนะนำ ⭐ — Standard)

```bash
pip install opentelemetry-sdk opentelemetry-exporter-otlp
```

```python
# tracing_setup.py
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
import haystack.tracing

def setup_tracing(service_name: str = "haystack-rag"):
    provider = TracerProvider()

    # Export ไป Jaeger / Grafana Tempo / OTLP-compatible
    exporter = OTLPSpanExporter(endpoint="http://jaeger:4317", insecure=True)
    provider.add_span_processor(BatchSpanProcessor(exporter))

    trace.set_tracer_provider(provider)

    # เชื่อมกับ Haystack
    from haystack.tracing import OpenTelemetryTracer
    haystack_tracer = OpenTelemetryTracer(
        tracer=trace.get_tracer(service_name)
    )
    haystack.tracing.enable_tracing(haystack_tracer)

    print(f"✅ Tracing enabled → {service_name}")

setup_tracing()
```

**สิ่งที่ trace อัตโนมัติ:**
- ชื่อ Component ที่รัน
- Input/Output แต่ละ Component
- Latency แต่ละ step
- Error ถ้ามี

### 3.2 Langfuse (แนะนำ ⭐ — ดีมากสำหรับ LLM)

Langfuse เป็น open-source LLM Observability ที่ใช้ได้ทั้ง LangChain และ Haystack

```bash
pip install langfuse
```

```python
# langfuse_setup.py
import os
os.environ["LANGFUSE_PUBLIC_KEY"] = "pk-lf-..."
os.environ["LANGFUSE_SECRET_KEY"] = "sk-lf-..."
os.environ["LANGFUSE_HOST"] = "https://cloud.langfuse.com"
# หรือ self-host: "http://localhost:3000"

from langfuse.haystack import LangfuseConnector

# เพิ่มเข้า Pipeline — ง่ายมาก 1 บรรทัด!
rag_pipeline.add_component("langfuse", LangfuseConnector("RAG Query"))

# ไม่ต้อง connect เพิ่ม — มันดัก event อัตโนมัติ
```

**Langfuse จะ track:**
- Input query
- Retrieved documents + scores
- Prompt ที่ส่งไป LLM
- LLM response + token count + cost
- Latency ทั้ง pipeline
- User feedback

**Self-host Langfuse ด้วย Docker:**

```yaml
# docker-compose.langfuse.yml
services:
  langfuse-db:
    image: postgres:15
    environment:
      POSTGRES_DB: langfuse
      POSTGRES_USER: langfuse
      POSTGRES_PASSWORD: secret

  langfuse:
    image: ghcr.io/langfuse/langfuse:latest
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://langfuse:secret@langfuse-db:5432/langfuse
      NEXTAUTH_SECRET: your-secret
      NEXTAUTH_URL: http://localhost:3000
      SALT: your-salt
    depends_on:
      - langfuse-db
```

### 3.3 Arize Phoenix (Free, Local)

```bash
pip install arize-phoenix openinference-instrumentation-haystack
```

```python
import phoenix as px
from openinference.instrumentation.haystack import HaystackInstrumentor

# เปิด Phoenix UI
px.launch_app()  # http://localhost:6006

# Instrument Haystack
HaystackInstrumentor().instrument()

# หลังจากนี้ ทุก pipeline.run() จะ log ไป Phoenix อัตโนมัติ
result = rag_pipeline.run({"embedder": {"text": "query"}})
```

---

## 4. Monitoring — วัด Health & Performance

### 4.1 Prometheus + Grafana Stack

```bash
pip install prometheus-client
```

```python
# metrics.py
from prometheus_client import Counter, Histogram, Gauge, start_http_server
import time

# Define metrics
QUERY_COUNTER = Counter(
    "rag_queries_total",
    "Total number of RAG queries",
    ["status"]  # success / error
)
QUERY_LATENCY = Histogram(
    "rag_query_duration_seconds",
    "RAG query latency",
    buckets=[0.1, 0.5, 1.0, 2.0, 5.0, 10.0]
)
RETRIEVAL_DOCS = Histogram(
    "rag_retrieved_docs_count",
    "Number of documents retrieved",
    buckets=[1, 3, 5, 10, 20]
)
TOKEN_COUNTER = Counter(
    "llm_tokens_total",
    "Total LLM tokens used",
    ["type"]  # prompt / completion
)
ACTIVE_REQUESTS = Gauge(
    "rag_active_requests",
    "Current active requests"
)

# Middleware ใน FastAPI
from fastapi import Request

@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    ACTIVE_REQUESTS.inc()
    start = time.time()

    try:
        response = await call_next(request)
        QUERY_COUNTER.labels(status="success").inc()
        return response
    except Exception as e:
        QUERY_COUNTER.labels(status="error").inc()
        raise
    finally:
        QUERY_LATENCY.observe(time.time() - start)
        ACTIVE_REQUESTS.dec()

# Expose metrics endpoint
from prometheus_client import make_asgi_app

metrics_app = make_asgi_app()
app.mount("/metrics", metrics_app)
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'haystack-rag'
    static_configs:
      - targets: ['app:8000']
    metrics_path: '/metrics'
    scrape_interval: 15s
```

**Grafana Dashboard แนะนำ:**

```
Panel 1: Query Rate (req/sec)
Panel 2: P50/P95/P99 Latency
Panel 3: Error Rate
Panel 4: Token Usage (cost estimation)
Panel 5: Cache Hit Rate
Panel 6: Retrieval Score Distribution
```

### 4.2 Full Docker Compose Stack

```yaml
# docker-compose.prod.yml
version: "3.8"

services:
  # ── App ──────────────────────────────────
  app:
    build: .
    ports:
      - "8000:8000"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - QDRANT_URL=http://qdrant:6333
      - LANGFUSE_PUBLIC_KEY=${LANGFUSE_PUBLIC_KEY}
      - LANGFUSE_SECRET_KEY=${LANGFUSE_SECRET_KEY}
    depends_on:
      - qdrant

  # ── Vector DB ────────────────────────────
  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"
    volumes:
      - qdrant_data:/qdrant/storage

  # ── Observability ────────────────────────
  langfuse-db:
    image: postgres:15
    environment:
      POSTGRES_DB: langfuse
      POSTGRES_USER: langfuse
      POSTGRES_PASSWORD: ${LANGFUSE_DB_PASSWORD}
    volumes:
      - langfuse_db:/var/lib/postgresql/data

  langfuse:
    image: ghcr.io/langfuse/langfuse:latest
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://langfuse:${LANGFUSE_DB_PASSWORD}@langfuse-db/langfuse
      NEXTAUTH_SECRET: ${NEXTAUTH_SECRET}
      NEXTAUTH_URL: http://localhost:3000
    depends_on:
      - langfuse-db

  # ── Metrics ──────────────────────────────
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - prometheus

volumes:
  qdrant_data:
  langfuse_db:
  grafana_data:
```

---

## 5. Rate Limiting & Guardrails

### 5.1 Rate Limiting

```bash
pip install slowapi
```

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.post("/query")
@limiter.limit("20/minute")  # 20 requests per minute per IP
async def query(request: Request, body: QueryRequest):
    ...
```

### 5.2 Input/Output Guardrails

```python
from haystack import component
from typing import List

@component
class InputGuardrail:
    """ตรวจสอบ query ก่อน process"""

    BLOCKED_PATTERNS = ["ignore previous", "jailbreak", "prompt injection"]

    @component.output_types(query=str, blocked=bool)
    def run(self, query: str) -> dict:
        query_lower = query.lower()
        for pattern in self.BLOCKED_PATTERNS:
            if pattern in query_lower:
                return {"query": "", "blocked": True}
        if len(query) > 2000:
            return {"query": query[:2000], "blocked": False}
        return {"query": query, "blocked": False}

@component
class OutputGuardrail:
    """ตรวจสอบ response ก่อน return"""

    PII_PATTERNS = [
        r'\b\d{13}\b',           # Thai ID
        r'\b\d{4}[-\s]\d{4}\b'  # Card number pattern
    ]

    @component.output_types(response=str, flagged=bool)
    def run(self, response: str) -> dict:
        import re
        for pattern in self.PII_PATTERNS:
            if re.search(pattern, response):
                return {"response": "[REDACTED - contains PII]", "flagged": True}
        return {"response": response, "flagged": False}
```

---

## 6. Haystack Ecosystem สรุป

```
┌─────────────────────────────────────────────────────────────┐
│                 HAYSTACK FULL ECOSYSTEM                     │
├─────────────────┬───────────────────────────────────────────┤
│ Category        │ Tools                                     │
├─────────────────┼───────────────────────────────────────────┤
│ API Serving     │ Hayhooks (official) ★                    │
│                 │ FastAPI (DIY, full control)               │
├─────────────────┼───────────────────────────────────────────┤
│ Authentication  │ FastAPI + python-jose (JWT)               │
│                 │ API Key middleware                         │
│                 │ Keycloak / Auth0 (enterprise)             │
├─────────────────┼───────────────────────────────────────────┤
│ Tracing         │ Langfuse ★ (LLM-specific, free self-host) │
│                 │ OpenTelemetry + Jaeger/Tempo              │
│                 │ Arize Phoenix (local, free)               │
│                 │ Datadog APM (paid)                        │
├─────────────────┼───────────────────────────────────────────┤
│ Monitoring      │ Prometheus + Grafana ★                    │
│                 │ Datadog / New Relic (paid)                │
├─────────────────┼───────────────────────────────────────────┤
│ Evaluation      │ Haystack Evaluators (built-in)            │
│                 │ RAGAS (open source)                       │
│                 │ Langfuse (evals + tracing รวม)           │
├─────────────────┼───────────────────────────────────────────┤
│ Vector DB       │ Qdrant ★ / Weaviate / Pinecone / Chroma  │
├─────────────────┼───────────────────────────────────────────┤
│ Managed Cloud   │ deepset Cloud (Haystack as a Service)     │
├─────────────────┼───────────────────────────────────────────┤
│ Local LLM       │ Ollama ★ / vLLM / LM Studio              │
└─────────────────┴───────────────────────────────────────────┘
```

---

## 7. ยากมั้ย? — ประเมินตรงๆ

```
┌──────────────────────────────────────────────────────────────┐
│              ความยากเทียบ LangChain                         │
├───────────────────┬──────────────┬──────────────────────────┤
│ งาน               │ LangChain    │ Haystack                 │
├───────────────────┼──────────────┼──────────────────────────┤
│ API Serving       │ ⭐ ง่ายมาก   │ ⭐⭐ ปานกลาง             │
│                   │ LangServe    │ Hayhooks หรือ FastAPI DIY │
├───────────────────┼──────────────┼──────────────────────────┤
│ Auth              │ ⭐⭐ ปานกลาง  │ ⭐⭐ เท่ากัน             │
│                   │ ต้อง DIY     │ ต้อง DIY เหมือนกัน      │
├───────────────────┼──────────────┼──────────────────────────┤
│ Tracing/Observe   │ ⭐ ง่ายมาก   │ ⭐ ง่ายเหมือนกัน        │
│                   │ LangSmith    │ Langfuse ใช้ร่วมกันได้   │
├───────────────────┼──────────────┼──────────────────────────┤
│ Monitoring        │ ⭐⭐ ปานกลาง  │ ⭐⭐ ปานกลาง             │
│                   │ Prometheus   │ Prometheus เหมือนกัน     │
├───────────────────┼──────────────┼──────────────────────────┤
│ Pipeline Debug    │ ⭐⭐ ปานกลาง  │ ⭐ ง่ายกว่า              │
│                   │ Chain debug  │ Type-safe, validation    │
└───────────────────┴──────────────┴──────────────────────────┘

ข้อสรุป:
✅ Langfuse ใช้ได้กับ ทั้งสองฝั่ง — ไม่ต้องเลือก
✅ Auth ยากพอๆ กัน — ทั้งคู่ต้อง DIY
⚠️  API Serving: LangChain ง่ายกว่าเล็กน้อยด้วย LangServe
✅ Pipeline Type-safety: Haystack ชนะ — หา bug ง่ายกว่ามาก
```

---

## 8. Recommended Production Stack (2025)

```
Pipeline:        Haystack 2.x
Vector DB:       Qdrant (self-host) หรือ Qdrant Cloud
API:             FastAPI + Hayhooks
Auth:            JWT (python-jose) หรือ Auth0
Tracing:         Langfuse (self-host ด้วย Docker)
Metrics:         Prometheus + Grafana
Deployment:      Docker Compose → Kubernetes
CI/CD:           GitHub Actions
Eval:            RAGAS + Langfuse Evals
```

---

## 9. Quick Start — Production Stack ใน 15 นาที

```bash
# 1. Clone template
git clone https://github.com/deepset-ai/haystack-cookbook
cd haystack-cookbook/production-template

# 2. Copy env
cp .env.example .env
# แก้ไข API keys

# 3. รัน full stack
docker compose -f docker-compose.prod.yml up -d

# Services ที่ได้:
# :8000  → Haystack API
# :3000  → Langfuse (Tracing)
# :3001  → Grafana (Metrics)
# :6333  → Qdrant
# :9090  → Prometheus
```

---

> 💡 **TL;DR สำหรับทีม:**
> - ถ้าต้องการ **เร็วมาก** → LangChain + LangServe + LangSmith
> - ถ้าต้องการ **pipeline ที่เชื่อถือได้, type-safe, production-grade** → Haystack + Langfuse
> - **Langfuse เป็น tool กลาง** ที่ใช้ได้ทั้งคู่ ไม่ต้องเลือกข้างนี้
