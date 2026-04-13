# Advanced RAG Deep Dive
## Hybrid Search, Re-ranking, Multi-query และเทคนิคขั้นสูงทั้งหมด

> เอกสารนี้ครอบคลุมทุกเทคนิค Advanced RAG ที่ใช้ใน Production จริง
> ทุกตัวอย่างรันได้จริง ใช้ Python + LangChain + Gemini

---

## สารบัญ

1. [ปัญหาของ Naive RAG](#1-ปัญหาของ-naive-rag)
2. [Hybrid Search — รวม Keyword + Semantic](#2-hybrid-search)
3. [Re-ranking — จัดอันดับผลลัพธ์ใหม่](#3-re-ranking)
4. [Multi-Query Retrieval — ถามหลายมุม](#4-multi-query-retrieval)
5. [Query Transformation — ปรับคำถามก่อนค้นหา](#5-query-transformation)
6. [Parent Document Retriever — ค้นเล็ก ดึงใหญ่](#6-parent-document-retriever)
7. [Self-Query Retriever — แปลงคำถามเป็น Filter](#7-self-query-retriever)
8. [Contextual Compression — บีบอัด Context](#8-contextual-compression)
9. [RAPTOR — Recursive Abstractive Processing](#9-raptor)
10. [Corrective RAG (CRAG)](#10-corrective-rag)
11. [Self-RAG — AI ตรวจสอบตัวเอง](#11-self-rag)
12. [Adaptive RAG — เลือก Strategy อัตโนมัติ](#12-adaptive-rag)
13. [รวมทุกเทคนิค — Production RAG Pipeline](#13-production-pipeline)
14. [เปรียบเทียบเทคนิคทั้งหมด](#14-comparison)

---

## 1. ปัญหาของ Naive RAG

Naive RAG (แบบพื้นฐาน) มีปัญหาหลายอย่าง:

```
❌ ปัญหาของ Naive RAG:

1. Retrieval ไม่แม่น
   - Semantic search อย่างเดียวพลาดคำเฉพาะทาง (ชื่อคน, รหัสสินค้า, ตัวย่อ)
   - ดึง chunks ที่ซ้ำๆ กันมา (ขาดความหลากหลาย)
   - คำถามซับซ้อน → ค้นหาไม่ครอบคลุม

2. Context คุณภาพต่ำ
   - Chunks ที่ดึงมามีส่วนที่ไม่เกี่ยวข้องปนอยู่
   - ลำดับ chunks ไม่เหมาะ (อันที่ดีที่สุดอาจไม่ได้อยู่ต้นๆ)

3. Generation ไม่ดี
   - LLM หลอน (Hallucinate) ตอบนอกเหนือ context
   - ไม่ตรวจสอบว่าคำตอบสอดคล้องกับ context จริงไหม
   - ไม่สามารถบอกได้ว่า "ไม่มีข้อมูล"
```

```
✅ Advanced RAG แก้ปัญหาเหล่านี้ด้วย:

1. Hybrid Search → แม่นทั้งคำเฉพาะทางและความหมาย
2. Re-ranking → จัดลำดับผลลัพธ์ใหม่ให้แม่นยำ
3. Multi-query → ค้นหาหลายมุมมอง
4. Compression → ตัดส่วนไม่เกี่ยวข้องออก
5. Self-RAG / CRAG → ตรวจสอบคุณภาพก่อนตอบ
```

---

## 2. Hybrid Search

### 2.1 แนวคิด

Hybrid Search = **Keyword Search (BM25) + Semantic Search (Vector)** รวมกัน

```
คำถาม: "นโยบาย WFH ปี 2025"

Keyword Search (BM25):                    Semantic Search (Vector):
✅ เก่งเรื่องคำเฉพาะ "WFH", "2025"       ✅ เก่งเรื่องความหมาย
❌ ไม่เข้าใจ "ทำงานจากบ้าน" = "WFH"      ❌ อาจพลาดคำเฉพาะเจาะจง
                        ↘               ↙
                     Hybrid Search
                  ✅ ได้ทั้งสองอย่าง
```

### 2.2 ติดตั้ง

```bash
pip install -U langchain-chroma langchain-google-genai rank_bm25
```

### 2.3 สร้าง Hybrid Search ด้วย EnsembleRetriever

```python
import os
from dotenv import load_dotenv
from langchain_google_genai import ChatGoogleGenerativeAI, GoogleGenerativeAIEmbeddings
from langchain_chroma import Chroma
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_core.documents import Document
from langchain_community.retrievers import BM25Retriever
from langchain.retrievers import EnsembleRetriever

load_dotenv()

# ======== เตรียมข้อมูล ========
documents = [
    Document(
        page_content="""นโยบาย Work From Home (WFH) ปี 2025
พนักงานสามารถ WFH ได้สูงสุด 2 วัน/สัปดาห์
ต้องแจ้งหัวหน้างานล่วงหน้าอย่างน้อย 1 วัน
ระหว่าง WFH ต้องออนไลน์ใน Slack ตลอดเวลาทำการ 9:00-18:00
ต้องเข้าประชุมทุกครั้งผ่าน Google Meet""",
        metadata={"source": "hr-policy-2025.pdf", "category": "wfh"}
    ),
    Document(
        page_content="""ระเบียบการลา บริษัท ABC
ลาพักร้อน 15 วัน/ปี สะสมได้ไม่เกิน 5 วัน
ลาป่วย 30 วัน/ปี ลาเกิน 3 วันต้องมีใบรับรองแพทย์
ลากิจ 5 วัน/ปี แจ้งล่วงหน้า 1 วัน
ลาคลอด 98 วัน ได้รับค่าจ้าง 45 วัน""",
        metadata={"source": "leave-policy.pdf", "category": "leave"}
    ),
    Document(
        page_content="""สวัสดิการด้านสุขภาพ
ประกันสุขภาพกลุ่ม AIA คุ้มครองพนักงานและครอบครัว
OPD วงเงิน 3,000 บาท/ครั้ง สูงสุด 30 ครั้ง/ปี
IPD วงเงิน 200,000 บาท/ครั้ง
ทันตกรรม 5,000 บาท/ปี
ตรวจสุขภาพประจำปี ฟรี ที่ รพ.กรุงเทพ""",
        metadata={"source": "health-benefits.pdf", "category": "health"}
    ),
    Document(
        page_content="""IT Security Policy
รหัสผ่านต้องมีอย่างน้อย 12 ตัวอักษร ประกอบด้วยตัวพิมพ์ใหญ่ ตัวพิมพ์เล็ก ตัวเลข และอักขระพิเศษ
เปลี่ยนรหัสผ่านทุก 90 วัน ห้ามใช้รหัสผ่าน 5 ชุดล่าสุดซ้ำ
ใช้ VPN FortiClient เมื่อเชื่อมต่อจากภายนอก: vpn.abc-company.com
MFA (Multi-Factor Authentication) บังคับใช้กับทุกระบบตั้งแต่ 1 ม.ค. 2025""",
        metadata={"source": "it-security.pdf", "category": "it"}
    ),
    Document(
        page_content="""ระเบียบการเบิกค่าใช้จ่าย
ค่าอาหาร: 100 บาท/วันทำการ (22 วัน/เดือน = 2,200 บาท/เดือน)
ค่าเดินทาง: เหมาจ่าย 2,500 บาท/เดือน
ค่าเดินทางไปต่างจังหวัด: เบิกตามจริง แนบใบเสร็จ สูงสุด 5,000 บาท/ครั้ง
ค่าที่พักต่างจังหวัด: สูงสุด 1,500 บาท/คืน
ค่าโทรศัพท์: 500 บาท/เดือน (ระดับ Senior ขึ้นไป)
ยื่นเบิกภายใน 30 วันหลังเกิดค่าใช้จ่าย ผ่านระบบ SAP""",
        metadata={"source": "expense-policy.pdf", "category": "expense"}
    ),
]

# Chunking
splitter = RecursiveCharacterTextSplitter(chunk_size=300, chunk_overlap=60)
chunks = splitter.split_documents(documents)
print(f"✂️  ตัดได้ {len(chunks)} chunks")

# ======== สร้าง Retrievers ========

# 1. Semantic Retriever (Vector)
embeddings = GoogleGenerativeAIEmbeddings(model="models/text-embedding-004")
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=embeddings,
    collection_name="hybrid-demo",
)
semantic_retriever = vectorstore.as_retriever(search_kwargs={"k": 4})

# 2. Keyword Retriever (BM25)
bm25_retriever = BM25Retriever.from_documents(chunks)
bm25_retriever.k = 4

# 3. Hybrid Retriever (รวมกัน)
hybrid_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, semantic_retriever],
    weights=[0.4, 0.6],  # ให้น้ำหนัก semantic มากกว่า
    # weights=[0.5, 0.5]  # หรือเท่ากัน
)

# ======== ทดสอบเปรียบเทียบ ========

test_queries = [
    "WFH 2025",                           # มีคำเฉพาะ → BM25 เก่ง
    "ทำงานจากบ้านได้ไหม",                    # ไม่มีคำว่า WFH → Semantic เก่ง
    "OPD วงเงินเท่าไหร่",                    # ผสม: OPD (keyword) + วงเงิน (semantic)
    "เบิกค่าเดินทางไปเชียงใหม่ยังไง",          # ต้องเข้าใจความหมาย
]

for query in test_queries:
    print(f"\n{'='*60}")
    print(f"🔍 Query: {query}")
    print(f"{'='*60}")

    # BM25 only
    bm25_results = bm25_retriever.invoke(query)
    print(f"\n📝 BM25 (Keyword):")
    for doc in bm25_results[:2]:
        print(f"   [{doc.metadata.get('source', '?')}] {doc.page_content[:80]}...")

    # Semantic only
    semantic_results = semantic_retriever.invoke(query)
    print(f"\n🧠 Semantic (Vector):")
    for doc in semantic_results[:2]:
        print(f"   [{doc.metadata.get('source', '?')}] {doc.page_content[:80]}...")

    # Hybrid
    hybrid_results = hybrid_retriever.invoke(query)
    print(f"\n⚡ Hybrid (BM25 + Semantic):")
    for doc in hybrid_results[:2]:
        print(f"   [{doc.metadata.get('source', '?')}] {doc.page_content[:80]}...")
```

**Output:**
```
============================================================
🔍 Query: WFH 2025
============================================================

📝 BM25 (Keyword):
   [hr-policy-2025.pdf] นโยบาย Work From Home (WFH) ปี 2025 พนักงานสามารถ WFH ได้สูงสุด 2...
   [it-security.pdf] MFA (Multi-Factor Authentication) บังคับใช้กับทุกระบบตั้งแต่ 1 ม.ค. 2025...

🧠 Semantic (Vector):
   [hr-policy-2025.pdf] นโยบาย Work From Home (WFH) ปี 2025 พนักงานสามารถ WFH ได้สูงสุด 2...
   [leave-policy.pdf] ระเบียบการลา บริษัท ABC ลาพักร้อน 15 วัน/ปี...

⚡ Hybrid (BM25 + Semantic):
   [hr-policy-2025.pdf] นโยบาย Work From Home (WFH) ปี 2025 พนักงานสามารถ WFH ได้สูงสุด 2...
   [it-security.pdf] MFA (Multi-Factor Authentication) บังคับใช้กับทุกระบบตั้งแต่ 1 ม.ค. 2025...

============================================================
🔍 Query: ทำงานจากบ้านได้ไหม
============================================================

📝 BM25 (Keyword):
   [expense-policy.pdf] ค่าอาหาร: 100 บาท/วันทำการ...         ← ❌ ไม่เกี่ยว!
   [leave-policy.pdf] ระเบียบการลา...                          ← ❌ ไม่เกี่ยว!

🧠 Semantic (Vector):
   [hr-policy-2025.pdf] นโยบาย Work From Home (WFH) ปี 2025... ← ✅ ถูกต้อง!
   [hr-policy-2025.pdf] ระหว่าง WFH ต้องออนไลน์ใน Slack...     ← ✅ ถูกต้อง!

⚡ Hybrid (BM25 + Semantic):
   [hr-policy-2025.pdf] นโยบาย Work From Home (WFH) ปี 2025... ← ✅ ถูกต้อง!
   [hr-policy-2025.pdf] ระหว่าง WFH ต้องออนไลน์ใน Slack...     ← ✅ ถูกต้อง!
```

### 2.4 ปรับ Weights

```python
# ข้อมูลที่มีศัพท์เทคนิคเยอะ → เพิ่มน้ำหนัก BM25
technical_hybrid = EnsembleRetriever(
    retrievers=[bm25_retriever, semantic_retriever],
    weights=[0.6, 0.4],  # BM25 มากกว่า
)

# ข้อมูลทั่วไป → เพิ่มน้ำหนัก Semantic
general_hybrid = EnsembleRetriever(
    retrievers=[bm25_retriever, semantic_retriever],
    weights=[0.3, 0.7],  # Semantic มากกว่า
)
```

---

## 3. Re-ranking

### 3.1 แนวคิด

Re-ranking = ดึง chunks มาเยอะๆ ก่อน (เช่น 20 อัน) → จัดอันดับใหม่ด้วย model ที่แม่นยำกว่า → เลือก top N

```
Query → Retriever (k=20) → 20 chunks → Re-ranker → Top 3 chunks → LLM
                            (เร็ว, หยาบ)            (ช้า, แม่นยำ)
```

### 3.2 Re-ranking ด้วย Cross-Encoder

```bash
pip install sentence-transformers
```

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import CrossEncoderReranker
from langchain_community.cross_encoders import HuggingFaceCrossEncoder

# สร้าง Cross-Encoder model
cross_encoder = HuggingFaceCrossEncoder(
    model_name="cross-encoder/ms-marco-MiniLM-L-6-v2"
)
compressor = CrossEncoderReranker(
    model=cross_encoder,
    top_n=3,  # เลือก top 3 หลัง rerank
)

# สร้าง base retriever ที่ดึงมาเยอะก่อน
base_retriever = vectorstore.as_retriever(search_kwargs={"k": 15})

# รวมกัน
reranking_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=base_retriever,
)

# ทดสอบ
query = "สวัสดิการด้านสุขภาพมีอะไรบ้าง"

# ก่อน rerank (15 chunks, อาจมีอันไม่เกี่ยวปน)
raw_results = base_retriever.invoke(query)
print(f"📥 Before reranking: {len(raw_results)} chunks")
for i, doc in enumerate(raw_results[:5]):
    print(f"   {i+1}. [{doc.metadata['source']}] {doc.page_content[:60]}...")

# หลัง rerank (3 chunks, แม่นยำ)
reranked_results = reranking_retriever.invoke(query)
print(f"\n📊 After reranking: {len(reranked_results)} chunks")
for i, doc in enumerate(reranked_results):
    print(f"   {i+1}. [{doc.metadata['source']}] {doc.page_content[:60]}...")
```

**Output:**
```
📥 Before reranking: 15 chunks
   1. [health-benefits.pdf] ประกันสุขภาพกลุ่ม AIA คุ้มครองพนักงาน...    ← ✅
   2. [leave-policy.pdf] ลาป่วย 30 วัน/ปี ลาเกิน 3 วัน...              ← ⚠️ เกี่ยวบ้าง
   3. [expense-policy.pdf] ค่าอาหาร: 100 บาท/วันทำการ...               ← ❌ ไม่เกี่ยว
   4. [health-benefits.pdf] ทันตกรรม 5,000 บาท/ปี ตรวจสุขภาพ...         ← ✅
   5. [hr-policy-2025.pdf] WFH ได้สูงสุด 2 วัน/สัปดาห์...              ← ❌ ไม่เกี่ยว

📊 After reranking: 3 chunks
   1. [health-benefits.pdf] ประกันสุขภาพกลุ่ม AIA คุ้มครองพนักงาน...    ← ✅ เกี่ยวมาก
   2. [health-benefits.pdf] ทันตกรรม 5,000 บาท/ปี ตรวจสุขภาพ...         ← ✅ เกี่ยวมาก
   3. [health-benefits.pdf] OPD วงเงิน 3,000 บาท/ครั้ง...              ← ✅ เกี่ยวมาก
```

### 3.3 Re-ranking ด้วย LLM (ถ้าไม่อยากลง model)

```python
from langchain_google_genai import ChatGoogleGenerativeAI

rerank_model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)

def llm_rerank(query: str, documents: list, top_n: int = 3) -> list:
    """ใช้ LLM จัดอันดับ documents"""

    # สร้างรายการเอกสารให้ LLM ดู
    doc_list = ""
    for i, doc in enumerate(documents):
        doc_list += f"\n[{i}] {doc.page_content[:200]}"

    prompt = f"""จากคำถาม: "{query}"
จัดอันดับเอกสารต่อไปนี้ตามความเกี่ยวข้อง (เกี่ยวข้องมากที่สุดก่อน):
{doc_list}

ตอบเฉพาะหมายเลขเอกสาร คั่นด้วยจุลภาค เช่น: 2,0,4"""

    response = rerank_model.invoke(prompt)

    # Parse ผลลัพธ์
    try:
        indices = [int(x.strip()) for x in response.content.split(",")]
        reranked = [documents[i] for i in indices[:top_n] if i < len(documents)]
        return reranked
    except:
        return documents[:top_n]

# ใช้งาน
raw = base_retriever.invoke("สวัสดิการด้านสุขภาพ")
reranked = llm_rerank("สวัสดิการด้านสุขภาพ", raw, top_n=3)
```

### 3.4 Hybrid + Re-ranking (Best Combo)

```python
# Step 1: Hybrid Search → ดึงมาเยอะ (k=15)
hybrid_base = EnsembleRetriever(
    retrievers=[
        BM25Retriever.from_documents(chunks, k=15),
        vectorstore.as_retriever(search_kwargs={"k": 15}),
    ],
    weights=[0.4, 0.6],
)

# Step 2: Re-rank → เลือก top 4
hybrid_rerank_retriever = ContextualCompressionRetriever(
    base_compressor=CrossEncoderReranker(model=cross_encoder, top_n=4),
    base_retriever=hybrid_base,
)

# ได้ผลลัพธ์ที่ดีที่สุดจากทั้ง 2 เทคนิครวมกัน
results = hybrid_rerank_retriever.invoke("OPD วงเงินเท่าไหร่")
```

---

## 4. Multi-Query Retrieval

### 4.1 แนวคิด

ให้ LLM สร้างคำถามหลายเวอร์ชันจากคำถามเดียว → ค้นหาแต่ละเวอร์ชัน → รวมผลลัพธ์

```
คำถามเดิม: "สวัสดิการด้านสุขภาพมีอะไรบ้าง"
                    │
                    ▼ (LLM สร้าง 3 variations)
    ┌───────────────┼───────────────┐
    ▼               ▼               ▼
"ประกันสุขภาพ    "ค่ารักษา        "ตรวจสุขภาพ
 บริษัทมีอะไร"   พยาบาลเท่าไหร่"  ประจำปี"
    │               │               │
    ▼               ▼               ▼
  [Docs A]       [Docs B]       [Docs C]
    │               │               │
    └───────────────┼───────────────┘
                    ▼
            Deduplicated Results
           (ครอบคลุมทุกมุมมอง)
```

### 4.2 ใช้ MultiQueryRetriever

```python
from langchain.retrievers.multi_query import MultiQueryRetriever

model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0.3)

# สร้าง MultiQueryRetriever
multi_retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(search_kwargs={"k": 4}),
    llm=model,
)

# ทดสอบ
import logging
logging.basicConfig()
logging.getLogger("langchain.retrievers.multi_query").setLevel(logging.INFO)

results = multi_retriever.invoke("สวัสดิการด้านสุขภาพมีอะไรบ้าง")

# Log จะแสดง queries ที่ LLM สร้างขึ้น:
# INFO: Generated queries:
# 1. ประกันสุขภาพของบริษัทครอบคลุมอะไรบ้าง
# 2. วงเงินค่ารักษาพยาบาลเท่าไหร่
# 3. บริษัทมีสวัสดิการด้านทันตกรรมและตรวจสุขภาพไหม

print(f"ดึงมาได้ {len(results)} chunks (deduplicated)")
for doc in results:
    print(f"  [{doc.metadata['source']}] {doc.page_content[:80]}...")
```

### 4.3 Custom Multi-Query ที่ควบคุมได้มากกว่า

```python
from langchain_core.prompts import ChatPromptTemplate

# Custom prompt สำหรับสร้าง queries
multi_query_prompt = ChatPromptTemplate.from_messages([
    ("system", """คุณคือผู้เชี่ยวชาญด้านการค้นหาข้อมูล
จากคำถามที่ได้รับ ให้สร้างคำถาม 3 เวอร์ชันที่แตกต่างกัน เพื่อค้นหาข้อมูลให้ครอบคลุมที่สุด:

กฎ:
- คำถามแต่ละข้อต้องมุ่งเป้าไปที่มุมมองที่แตกต่างกัน
- ใช้คำศัพท์ที่หลากหลาย (synonyms, related terms)
- ตอบแค่ 3 คำถาม คั่นด้วยบรรทัดใหม่ ไม่ต้องมีหมายเลข"""),
    ("human", "คำถาม: {question}"),
])

def custom_multi_query_retrieve(question: str, retriever, model, k_per_query=3):
    """Multi-query retrieval ที่ควบคุมเอง"""

    # Step 1: สร้าง queries
    formatted = multi_query_prompt.invoke({"question": question})
    response = model.invoke(formatted)
    queries = [q.strip() for q in response.content.strip().split("\n") if q.strip()]
    queries.append(question)  # เพิ่มคำถามเดิมด้วย

    print(f"📝 Generated queries:")
    for i, q in enumerate(queries, 1):
        print(f"   {i}. {q}")

    # Step 2: ค้นหาแต่ละ query
    all_docs = []
    seen_contents = set()

    for query in queries:
        results = retriever.invoke(query)
        for doc in results:
            content_hash = hash(doc.page_content)
            if content_hash not in seen_contents:
                seen_contents.add(content_hash)
                all_docs.append(doc)

    print(f"\n📊 ดึงได้ {len(all_docs)} unique chunks (จาก {len(queries)} queries)")
    return all_docs

# ใช้งาน
base_retriever = vectorstore.as_retriever(search_kwargs={"k": 3})
results = custom_multi_query_retrieve(
    "การเบิกค่าใช้จ่ายเดินทาง",
    base_retriever,
    model
)
```

**Output:**
```
📝 Generated queries:
   1. ขั้นตอนการเบิกค่าเดินทางไปต่างจังหวัด
   2. ค่าเดินทางและค่าที่พักเบิกได้เท่าไหร่
   3. ยื่นเบิกค่าใช้จ่ายผ่านระบบอะไร ภายในกี่วัน
   4. การเบิกค่าใช้จ่ายเดินทาง

📊 ดึงได้ 7 unique chunks (จาก 4 queries)
```

---

## 5. Query Transformation

### 5.1 HyDE (Hypothetical Document Embeddings)

ให้ LLM สร้าง "คำตอบสมมติ" แล้วใช้คำตอบนั้นค้นหาแทนคำถาม:

```python
from langchain_core.prompts import ChatPromptTemplate

def hyde_retrieve(question: str, retriever, model):
    """HyDE: ค้นหาด้วยคำตอบสมมติ"""

    # Step 1: ให้ LLM สร้างคำตอบสมมติ
    hyde_prompt = ChatPromptTemplate.from_messages([
        ("system", "เขียนย่อหน้าสั้นๆ ที่น่าจะเป็นคำตอบของคำถามนี้ (ไม่ต้องถูกต้อง 100% แค่เขียนในทำนองที่เอกสารจะเขียน)"),
        ("human", "{question}")
    ])
    hypothetical_answer = model.invoke(hyde_prompt.invoke({"question": question}))

    print(f"📝 Hypothetical answer: {hypothetical_answer.content[:100]}...")

    # Step 2: ใช้คำตอบสมมตินั้นค้นหา (vector จะคล้ายกับเอกสารจริงมากกว่าคำถาม)
    results = retriever.invoke(hypothetical_answer.content)
    return results

# ทดสอบ: คำถามที่ semantic search อาจพลาด
results = hyde_retrieve(
    "บริษัทจ่ายเงินสมทบกองทุนให้เท่าไหร่",
    base_retriever,
    model
)
```

### 5.2 Step-back Prompting

```python
def stepback_retrieve(question: str, retriever, model):
    """Step-back: ถอยออกมาถามคำถามกว้างกว่า"""

    stepback_prompt = ChatPromptTemplate.from_messages([
        ("system", """จากคำถามที่ได้รับ ให้สร้างคำถามที่กว้างกว่า (step-back question)
ที่จะช่วยให้ค้นหาข้อมูลพื้นฐานที่จำเป็นสำหรับการตอบคำถามเดิม
ตอบเพียง 1 คำถาม"""),
        ("human", "คำถาม: {question}")
    ])

    stepback = model.invoke(stepback_prompt.invoke({"question": question}))
    print(f"🔙 Step-back question: {stepback.content}")

    # ค้นหาทั้งคำถามเดิมและ step-back
    original_docs = retriever.invoke(question)
    stepback_docs = retriever.invoke(stepback.content)

    # รวม deduplicate
    all_docs = original_docs + [d for d in stepback_docs
                                 if d.page_content not in [x.page_content for x in original_docs]]
    return all_docs

# ตัวอย่าง
# คำถาม: "ถ้าลาป่วย 5 วันต้องทำอะไรบ้าง"
# Step-back: "นโยบายการลาป่วยของบริษัทเป็นอย่างไร"
results = stepback_retrieve("ถ้าลาป่วย 5 วันต้องทำอะไรบ้าง", base_retriever, model)
```

---

## 6. Parent Document Retriever

### 6.1 แนวคิด

ค้นหาจาก **child chunks (เล็ก, แม่นยำ)** แต่ส่ง **parent chunks (ใหญ่, มีบริบทครบ)** ให้ LLM:

```
เอกสารเดิม (2000 ตัวอักษร)
    ├── Parent chunk 1 (800 chars)
    │   ├── Child chunk 1a (200 chars)  ← ใช้ค้นหา (vector search)
    │   ├── Child chunk 1b (200 chars)
    │   ├── Child chunk 1c (200 chars)
    │   └── Child chunk 1d (200 chars)
    └── Parent chunk 2 (800 chars)
        ├── Child chunk 2a (200 chars)
        └── ...

ค้นหาเจอ child 1b → ส่ง parent 1 ให้ LLM (ได้บริบทครบกว่า)
```

### 6.2 Implementation

```python
from langchain.retrievers import ParentDocumentRetriever
from langchain.storage import InMemoryStore
from langchain_text_splitters import RecursiveCharacterTextSplitter

# Child splitter (เล็ก สำหรับค้นหา)
child_splitter = RecursiveCharacterTextSplitter(chunk_size=150, chunk_overlap=30)

# Parent splitter (ใหญ่ สำหรับส่ง LLM)
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=600, chunk_overlap=100)

# Store สำหรับเก็บ parent documents
docstore = InMemoryStore()

# สร้าง Parent Document Retriever
parent_retriever = ParentDocumentRetriever(
    vectorstore=Chroma(embedding_function=embeddings, collection_name="parent-child"),
    docstore=docstore,
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)

# Index เอกสาร
parent_retriever.add_documents(documents)

# ค้นหา
results = parent_retriever.invoke("OPD วงเงิน")

for doc in results:
    print(f"📄 Parent chunk ({len(doc.page_content)} chars):")
    print(f"   {doc.page_content[:200]}...")
    print()

# จะได้ parent chunk ที่ใหญ่กว่า → LLM มีบริบทมากขึ้น
```

---

## 7. Self-Query Retriever

### 7.1 แนวคิด

ให้ LLM แปลง natural language query → structured query + metadata filter อัตโนมัติ:

```
คำถาม: "นโยบายลาพักร้อนจากเอกสาร HR"
                    │
                    ▼ (LLM แปลง)
        query: "นโยบายลาพักร้อน"
        filter: {"category": "leave"}
```

### 7.2 Implementation

```python
from langchain.retrievers.self_query.base import SelfQueryRetriever
from langchain.chains.query_constructor.schema import AttributeInfo

# กำหนดข้อมูล metadata ที่มี
metadata_field_info = [
    AttributeInfo(
        name="source",
        description="ชื่อไฟล์ต้นทาง เช่น hr-policy-2025.pdf, it-security.pdf",
        type="string",
    ),
    AttributeInfo(
        name="category",
        description="หมวดหมู่เอกสาร: wfh, leave, health, it, expense",
        type="string",
    ),
]

document_content_description = "เอกสารนโยบายภายในบริษัท ABC เกี่ยวกับ HR, IT, สวัสดิการ, ค่าใช้จ่าย"

self_query_retriever = SelfQueryRetriever.from_llm(
    llm=model,
    vectorstore=vectorstore,
    document_contents=document_content_description,
    metadata_field_info=metadata_field_info,
    verbose=True,
)

# ทดสอบ
results = self_query_retriever.invoke("นโยบาย IT security")
# LLM จะแปลงเป็น: query="นโยบาย security", filter={"category": "it"}

results = self_query_retriever.invoke("เอกสาร leave-policy.pdf พูดถึงอะไรบ้าง")
# LLM จะแปลงเป็น: query="", filter={"source": "leave-policy.pdf"}
```

---

## 8. Contextual Compression

### 8.1 แนวคิด

ตัดส่วนที่ไม่เกี่ยวข้องออกจาก chunk ก่อนส่งให้ LLM:

```
Chunk เดิม (300 chars):
"ประกันสุขภาพกลุ่ม AIA คุ้มครองพนักงานและครอบครัว
 OPD วงเงิน 3,000 บาท/ครั้ง สูงสุด 30 ครั้ง/ปี
 IPD วงเงิน 200,000 บาท/ครั้ง
 ทันตกรรม 5,000 บาท/ปี
 ตรวจสุขภาพประจำปี ฟรี ที่ รพ.กรุงเทพ"

คำถาม: "OPD วงเงินเท่าไหร่"

Chunk หลัง Compress (สั้นลง, ตรงประเด็น):
"OPD วงเงิน 3,000 บาท/ครั้ง สูงสุด 30 ครั้ง/ปี"
```

### 8.2 Implementation

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import LLMChainExtractor

# ใช้ LLM ดึงเฉพาะส่วนที่เกี่ยวข้อง
compressor = LLMChainExtractor.from_llm(model)

compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 5}),
)

results = compression_retriever.invoke("OPD วงเงินเท่าไหร่")
for doc in results:
    print(f"📄 Compressed: {doc.page_content}")
    # จะได้แค่ส่วนที่เกี่ยวกับ OPD
```

---

## 9. RAPTOR

### 9.1 แนวคิด

RAPTOR = **Recursive Abstractive Processing for Tree-Organized Retrieval**

สร้าง "สรุป" ในหลายระดับ เพื่อตอบคำถามได้ทั้งเจาะลึกและภาพรวม:

```
Level 0 (Raw chunks):     [c1] [c2] [c3] [c4] [c5] [c6] [c7] [c8]
                             \  /       \  /       \  /       \  /
Level 1 (Cluster+Summarize): [s1]      [s2]      [s3]      [s4]
                                \      /              \      /
Level 2 (Cluster+Summarize):    [S1]                  [S2]
                                   \                  /
Level 3 (Top summary):            [SUMMARY]

คำถามเจาะลึก → ค้นหาที่ Level 0
คำถามภาพรวม → ค้นหาที่ Level 2-3
```

### 9.2 Implementation (Simplified)

```python
def build_raptor_tree(chunks: list, model, embeddings, levels=2):
    """สร้าง RAPTOR tree แบบ simplified"""
    import numpy as np
    from sklearn.cluster import KMeans

    tree = {"level_0": chunks}

    current_chunks = chunks
    for level in range(1, levels + 1):
        # Step 1: Embed chunks
        texts = [c.page_content for c in current_chunks]
        vectors = embeddings.embed_documents(texts)

        # Step 2: Cluster
        n_clusters = max(len(current_chunks) // 3, 1)
        kmeans = KMeans(n_clusters=n_clusters, random_state=42)
        labels = kmeans.fit_predict(vectors)

        # Step 3: สรุปแต่ละ cluster
        summaries = []
        for cluster_id in range(n_clusters):
            cluster_texts = [texts[i] for i, l in enumerate(labels) if l == cluster_id]
            combined = "\n".join(cluster_texts)

            summary_response = model.invoke(
                f"สรุปข้อมูลต่อไปนี้ให้กระชับ:\n\n{combined}"
            )
            summaries.append(Document(
                page_content=summary_response.content,
                metadata={"level": level, "cluster": cluster_id}
            ))

        tree[f"level_{level}"] = summaries
        current_chunks = summaries
        print(f"Level {level}: {len(summaries)} summaries")

    return tree

# สร้าง tree
raptor_tree = build_raptor_tree(chunks, model, embeddings, levels=2)

# Index ทุก level เข้า vector store
all_nodes = []
for level_name, nodes in raptor_tree.items():
    all_nodes.extend(nodes)

raptor_vectorstore = Chroma.from_documents(all_nodes, embeddings, collection_name="raptor")
raptor_retriever = raptor_vectorstore.as_retriever(search_kwargs={"k": 5})

# คำถามภาพรวม → จะดึง summary จาก level สูงมาด้วย
# คำถามเจาะลึก → จะดึง raw chunks จาก level 0
```

---

## 10. Corrective RAG (CRAG)

### 10.1 แนวคิด

ตรวจสอบคุณภาพ retrieval ก่อนส่งให้ LLM:

```
Query → Retrieve → ✅ เกี่ยวข้อง? ─── Yes → Generate Answer
                         │
                         No
                         │
                    ▼ Fallback ▼
                  ┌──────────────┐
                  │ Web Search   │
                  │  หรือ        │
                  │ Rephrase     │
                  │  แล้วค้นใหม่ │
                  └──────────────┘
```

### 10.2 Implementation ด้วย LangGraph

```python
from langgraph.graph import StateGraph, START, END
from typing import Annotated, Literal
from typing_extensions import TypedDict
from langgraph.graph.message import add_messages

class CRAGState(TypedDict):
    messages: Annotated[list, add_messages]
    question: str
    documents: list
    generation: str
    relevance_score: str  # "relevant" | "not_relevant"

def retrieve(state: CRAGState):
    """ดึงเอกสาร"""
    docs = retriever.invoke(state["question"])
    return {"documents": [d.page_content for d in docs]}

def grade_documents(state: CRAGState):
    """ให้ LLM ตรวจว่าเอกสารเกี่ยวข้องไหม"""
    context = "\n".join(state["documents"])
    grading_response = model.invoke(
        f"""ตรวจสอบว่าข้อมูลต่อไปนี้เกี่ยวข้องกับคำถามหรือไม่:

คำถาม: {state['question']}
ข้อมูล: {context[:500]}

ตอบเพียง: relevant หรือ not_relevant"""
    )
    score = "relevant" if "relevant" in grading_response.content.lower() and "not" not in grading_response.content.lower() else "not_relevant"
    return {"relevance_score": score}

def generate(state: CRAGState):
    """สร้างคำตอบจากเอกสาร"""
    context = "\n".join(state["documents"])
    response = model.invoke(
        f"ตอบคำถามจากข้อมูลนี้:\n\n{context}\n\nคำถาม: {state['question']}"
    )
    return {"generation": response.content}

def web_search_fallback(state: CRAGState):
    """Fallback: ค้นหาจากแหล่งอื่น"""
    # ในงานจริงจะใช้ web search tool
    response = model.invoke(
        f"ตอบคำถามนี้จากความรู้ของคุณ (บอกด้วยว่าไม่มีในเอกสารภายใน): {state['question']}"
    )
    return {"generation": response.content}

def route_after_grading(state: CRAGState) -> Literal["generate", "web_search"]:
    if state["relevance_score"] == "relevant":
        return "generate"
    return "web_search"

# Build Graph
builder = StateGraph(CRAGState)
builder.add_node("retrieve", retrieve)
builder.add_node("grade", grade_documents)
builder.add_node("generate", generate)
builder.add_node("web_search", web_search_fallback)

builder.add_edge(START, "retrieve")
builder.add_edge("retrieve", "grade")
builder.add_conditional_edges("grade", route_after_grading)
builder.add_edge("generate", END)
builder.add_edge("web_search", END)

crag = builder.compile()

# ทดสอบ
# คำถามที่มีข้อมูล → retrieve → relevant → generate
r1 = crag.invoke({"question": "ลาพักร้อนได้กี่วัน", "messages": [], "documents": [], "generation": "", "relevance_score": ""})
print(f"✅ {r1['generation']}")

# คำถามที่ไม่มีข้อมูล → retrieve → not_relevant → web_search
r2 = crag.invoke({"question": "ราคา Bitcoin วันนี้เท่าไหร่", "messages": [], "documents": [], "generation": "", "relevance_score": ""})
print(f"🌐 {r2['generation']}")
```

---

## 11. Self-RAG

### 11.1 แนวคิด

AI ตรวจสอบตัวเองในทุกขั้นตอน:

```
Query → Retrieve → [ตรวจ: ต้อง retrieve ไหม?]
                         │
                    Yes  │  No → ตอบตรงเลย
                         ▼
                   [ตรวจ: docs เกี่ยวข้อง?]
                         │
                    Yes  │  No → ค้นใหม่ / ตอบว่าไม่มีข้อมูล
                         ▼
                    Generate
                         │
                   [ตรวจ: คำตอบมี hallucination?]
                         │
                    No   │  Yes → สร้างใหม่
                         ▼
                   [ตรวจ: คำตอบตอบคำถามได้?]
                         │
                    Yes  │  No → สร้างใหม่
                         ▼
                    Final Answer ✅
```

### 11.2 Implementation

```python
class SelfRAGState(TypedDict):
    question: str
    documents: list
    generation: str
    needs_retrieval: bool
    is_relevant: bool
    has_hallucination: bool
    answers_question: bool
    retry_count: int

def decide_retrieval(state: SelfRAGState):
    """ตรวจว่าต้อง retrieve ไหม"""
    response = model.invoke(
        f"""คำถามนี้ต้องค้นหาข้อมูลจากเอกสารไหม หรือตอบได้เลย?
คำถาม: {state['question']}
ตอบ: RETRIEVE หรือ DIRECT"""
    )
    needs = "RETRIEVE" in response.content.upper()
    return {"needs_retrieval": needs}

def check_hallucination(state: SelfRAGState):
    """ตรวจว่าคำตอบ hallucinate ไหม"""
    context = "\n".join(state["documents"][:3])
    response = model.invoke(
        f"""ตรวจว่าคำตอบต่อไปนี้มีข้อมูลที่ไม่ได้มาจาก context หรือไม่:

Context: {context[:500]}
คำตอบ: {state['generation']}

ตอบ: GROUNDED (ตอบจาก context) หรือ HALLUCINATION (แต่งเพิ่ม)"""
    )
    has_halluc = "HALLUCINATION" in response.content.upper()
    return {"has_hallucination": has_halluc}

def check_answers_question(state: SelfRAGState):
    """ตรวจว่าคำตอบตอบคำถามจริงไหม"""
    response = model.invoke(
        f"""คำตอบนี้ตอบคำถามได้จริงหรือไม่?
คำถาม: {state['question']}
คำตอบ: {state['generation']}
ตอบ: YES หรือ NO"""
    )
    answers = "YES" in response.content.upper()
    return {"answers_question": answers}

# สร้าง Graph ที่มีการตรวจสอบทุกขั้นตอน
builder = StateGraph(SelfRAGState)
builder.add_node("decide", decide_retrieval)
builder.add_node("retrieve", retrieve)
builder.add_node("generate", generate)
builder.add_node("check_hallucination", check_hallucination)
builder.add_node("check_answer", check_answers_question)
builder.add_node("regenerate", generate)  # สร้างใหม่

builder.add_edge(START, "decide")
builder.add_conditional_edges("decide", lambda s: "retrieve" if s["needs_retrieval"] else "generate")
builder.add_edge("retrieve", "generate")
builder.add_edge("generate", "check_hallucination")
builder.add_conditional_edges("check_hallucination",
    lambda s: "check_answer" if not s["has_hallucination"] else "regenerate")
builder.add_conditional_edges("check_answer",
    lambda s: END if s["answers_question"] else "regenerate")
builder.add_edge("regenerate", END)  # หยุดหลัง retry 1 ครั้ง

self_rag = builder.compile()
```

---

## 12. Adaptive RAG

### 12.1 แนวคิด

เลือก RAG strategy อัตโนมัติตามประเภทคำถาม:

```
Query → [Classify] → "Simple"     → Basic RAG
                   → "Complex"    → Multi-query + Re-ranking
                   → "Comparison" → Multi-retrieval + Parallel
                   → "No data"    → Direct LLM / Web Search
```

### 12.2 Implementation

```python
from typing import Literal

class AdaptiveRAGState(TypedDict):
    messages: Annotated[list, add_messages]
    question: str
    query_type: str  # simple, complex, comparison, out_of_scope
    documents: list
    generation: str

def classify_query(state: AdaptiveRAGState):
    """จำแนกประเภทคำถาม"""
    response = model.invoke(f"""จำแนกคำถามนี้:
"{state['question']}"

ประเภท:
- simple: คำถามตรงไปตรงมา ค้นหาได้ง่าย (เช่น "ลาพักร้อนกี่วัน")
- complex: คำถามซับซ้อน ต้องรวมข้อมูลหลายส่วน (เช่น "สรุปสวัสดิการทั้งหมด")
- comparison: เปรียบเทียบ (เช่น "ลาพักร้อนกับลากิจต่างกันยังไง")
- out_of_scope: ไม่เกี่ยวกับเอกสารบริษัท

ตอบเพียงคำเดียว: simple, complex, comparison, out_of_scope""")

    qtype = response.content.strip().lower()
    valid_types = {"simple", "complex", "comparison", "out_of_scope"}
    if qtype not in valid_types:
        qtype = "simple"

    print(f"🏷️  Query type: {qtype}")
    return {"query_type": qtype}

def simple_rag(state: AdaptiveRAGState):
    """Basic retrieval"""
    docs = retriever.invoke(state["question"])
    context = "\n".join([d.page_content for d in docs])
    response = model.invoke(f"ตอบจากข้อมูล:\n{context}\n\nคำถาม: {state['question']}")
    return {"generation": response.content, "documents": [d.page_content for d in docs]}

def complex_rag(state: AdaptiveRAGState):
    """Multi-query + Re-ranking"""
    # Multi-query
    all_docs = custom_multi_query_retrieve(state["question"], base_retriever, model)
    # Re-rank (ใช้ LLM rerank ถ้าไม่มี cross-encoder)
    reranked = llm_rerank(state["question"], all_docs, top_n=5)
    context = "\n".join([d.page_content for d in reranked])
    response = model.invoke(f"ตอบอย่างครอบคลุมจากข้อมูล:\n{context}\n\nคำถาม: {state['question']}")
    return {"generation": response.content, "documents": [d.page_content for d in reranked]}

def comparison_rag(state: AdaptiveRAGState):
    """ดึงข้อมูลทั้ง 2 ฝั่งมาเปรียบเทียบ"""
    # ให้ LLM แยกเป็น 2 sub-queries
    sub_response = model.invoke(
        f"แยกคำถามเปรียบเทียบนี้เป็น 2 คำถามย่อย (คั่นด้วย |): {state['question']}"
    )
    sub_queries = sub_response.content.split("|")

    all_docs = []
    for sq in sub_queries:
        docs = retriever.invoke(sq.strip())
        all_docs.extend(docs)

    context = "\n".join([d.page_content for d in all_docs])
    response = model.invoke(f"เปรียบเทียบจากข้อมูล:\n{context}\n\nคำถาม: {state['question']}")
    return {"generation": response.content, "documents": [d.page_content for d in all_docs]}

def out_of_scope_response(state: AdaptiveRAGState):
    return {"generation": "ขออภัยค่ะ คำถามนี้อยู่นอกขอบเขตข้อมูลที่มี กรุณาติดต่อ HR หรือ IT โดยตรง"}

def route_by_type(state: AdaptiveRAGState) -> Literal["simple_rag", "complex_rag", "comparison_rag", "out_of_scope"]:
    return {
        "simple": "simple_rag",
        "complex": "complex_rag",
        "comparison": "comparison_rag",
        "out_of_scope": "out_of_scope",
    }.get(state["query_type"], "simple_rag")

# Build
builder = StateGraph(AdaptiveRAGState)
builder.add_node("classify", classify_query)
builder.add_node("simple_rag", simple_rag)
builder.add_node("complex_rag", complex_rag)
builder.add_node("comparison_rag", comparison_rag)
builder.add_node("out_of_scope", out_of_scope_response)

builder.add_edge(START, "classify")
builder.add_conditional_edges("classify", route_by_type)
builder.add_edge("simple_rag", END)
builder.add_edge("complex_rag", END)
builder.add_edge("comparison_rag", END)
builder.add_edge("out_of_scope", END)

adaptive_rag = builder.compile()

# ทดสอบ
for q in [
    "ลาพักร้อนได้กี่วัน",                          # simple
    "สรุปสวัสดิการทั้งหมดของบริษัท",                   # complex
    "ลาพักร้อนกับลากิจต่างกันยังไง",                  # comparison
    "ร้านอาหารแถวทองหล่อมีร้านไหนอร่อย",              # out_of_scope
]:
    r = adaptive_rag.invoke({"question": q, "messages": [], "query_type": "", "documents": [], "generation": ""})
    print(f"\n❓ {q}")
    print(f"💬 {r['generation'][:150]}...")
```

---

## 13. Production Pipeline

### รวมทุกเทคนิคเข้าด้วยกัน

```
User Query
    │
    ▼
[1. Query Classification] ← Adaptive RAG
    │
    ├── Simple → [2a. Hybrid Search] → [3. Re-rank] → [4. Generate] → [5. Self-check] → Answer
    │
    ├── Complex → [2b. Multi-query + Hybrid] → [3. Re-rank] → [4. Generate] → [5. Self-check] → Answer
    │
    ├── Comparison → [2c. Multi-retrieval] → [3. Re-rank] → [4. Generate] → [5. Self-check] → Answer
    │
    └── Out of scope → "ไม่มีข้อมูล" → Answer
```

---

## 14. เปรียบเทียบเทคนิคทั้งหมด

| เทคนิค | แก้ปัญหา | ความซับซ้อน | ค่าใช้จ่ายเพิ่ม | เหมาะกับ |
|--------|---------|-----------|--------------|---------|
| **Hybrid Search** | Keyword miss | ต่ำ | ต่ำ | ทุกกรณี (แนะนำเสมอ) |
| **Re-ranking** | ลำดับผิด | ต่ำ | กลาง | ผลลัพธ์มีของไม่เกี่ยวปน |
| **Multi-query** | คำถามซับซ้อน | กลาง | กลาง (LLM calls เพิ่ม) | คำถามกว้างๆ |
| **HyDE** | Semantic gap | กลาง | กลาง | คำถามสั้นมากๆ |
| **Parent Document** | ขาดบริบท | กลาง | ต่ำ | เอกสารยาวที่มีโครงสร้าง |
| **Self-Query** | Filter ไม่แม่น | กลาง | กลาง | มี metadata หลากหลาย |
| **Compression** | Context ยาวเกิน | กลาง | กลาง | Chunks ใหญ่ |
| **RAPTOR** | Multi-level | สูง | สูง | เอกสารยาวมากๆ |
| **CRAG** | Retrieval ผิด | สูง | สูง | ต้องการ reliability สูง |
| **Self-RAG** | Hallucination | สูง | สูง | Production ที่ต้องเชื่อถือได้ |
| **Adaptive RAG** | One-size-fits-all | สูง | สูง | หลากหลายประเภทคำถาม |

### คำแนะนำสำหรับเริ่มต้น

```
Level 1 (เริ่มต้น):     Hybrid Search + Re-ranking
Level 2 (ปรับปรุง):     + Multi-query + Compression
Level 3 (Production):  + CRAG / Self-RAG + Adaptive RAG
```
