# Step 5: Advanced RAG
## Hybrid Search, Re-ranking, Multi-query — ทำ RAG ให้แม่นยำขึ้น

> Step 3 ทำ RAG พื้นฐานได้แล้ว Step นี้จะทำให้ค้นหาแม่นยำขึ้น ลด hallucination

---

## 5.1 ปัญหาของ RAG พื้นฐาน

```
❌ ปัญหาที่เจอ:

1. ค้นหาไม่เจอ
   คำถาม: "WFH policy 2025"
   Semantic search → ไม่เจอ (เพราะไม่รู้จักคำย่อ "WFH")

2. ได้ chunks ซ้ำๆ
   ดึงมา 5 chunks → 3 อันพูดเรื่องเดียวกัน = เสีย token

3. chunks ที่ดึงมาไม่ตรงประเด็น
   ถามเรื่อง OPD → ได้ chunk เรื่องค่าอาหารมาด้วย
```

---

## 5.2 ติดตั้ง

```bash
pip install rank_bm25 sentence-transformers
# rank_bm25            = Keyword search (BM25)
# sentence-transformers = Cross-encoder สำหรับ Re-ranking
```

---

## 5.3 Hybrid Search — รวม Keyword + Semantic

```python
# file: 27_hybrid_search.py

from dotenv import load_dotenv
from langchain_community.retrievers import BM25Retriever
from langchain.retrievers import EnsembleRetriever

load_dotenv()

# สมมติว่ามี chunks จาก Step 3 แล้ว

# ==== 1. สร้าง Keyword Retriever (BM25) ====
bm25_retriever = BM25Retriever.from_documents(chunks)
# BM25 คืออะไร?
# = อัลกอริทึมค้นหาแบบ "นับคำ" (keyword matching)
# เก่งเรื่อง: คำเฉพาะ (ชื่อคน, รหัส, ตัวย่อ เช่น WFH, OPD, VPN)
# ไม่เก่งเรื่อง: ความหมาย ("ทำงานจากบ้าน" ≠ "WFH" ในมุม BM25)

bm25_retriever.k = 4  # ดึง 4 อัน

# ==== 2. สร้าง Semantic Retriever (จาก Step 3) ====
semantic_retriever = vectorstore.as_retriever(search_kwargs={"k": 4})
# Semantic คืออะไร?
# = ค้นหาด้วย "ความหมาย" (vector similarity)
# เก่งเรื่อง: "ทำงานจากบ้าน" = "WFH" = "remote work" (เข้าใจความหมาย)
# ไม่เก่งเรื่อง: คำเฉพาะที่ต้อง match ตรงๆ

# ==== 3. รวมกัน! ====
hybrid_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, semantic_retriever],
    weights=[0.4, 0.6],
    # weights คืออะไร?
    # = น้ำหนักของแต่ละ retriever
    # [0.4, 0.6] = ให้ semantic มากกว่า keyword นิดหน่อย
    # [0.5, 0.5] = เท่ากัน
    # [0.7, 0.3] = ให้ keyword มากกว่า (เหมาะกับข้อมูลที่มีศัพท์เทคนิคเยอะ)
)

# ==== 4. ทดสอบเปรียบเทียบ ====
query = "ทำงานจากบ้านได้ไหม"  # ไม่มีคำว่า "WFH" ตรงๆ

print("📝 BM25 (Keyword):")
for doc in bm25_retriever.invoke(query)[:2]:
    print(f"   {doc.page_content[:60]}...")
    # ❌ อาจไม่เจอ WFH เพราะไม่มีคำตรง

print("\n🧠 Semantic (Vector):")
for doc in semantic_retriever.invoke(query)[:2]:
    print(f"   {doc.page_content[:60]}...")
    # ✅ เจอ WFH เพราะเข้าใจความหมาย

print("\n⚡ Hybrid (รวมกัน):")
for doc in hybrid_retriever.invoke(query)[:2]:
    print(f"   {doc.page_content[:60]}...")
    # ✅ ได้ทั้งสองอย่าง
```

---

## 5.4 Re-ranking — จัดอันดับใหม่ให้แม่นยำ

```python
# file: 28_reranking.py

# แนวคิด:
# 1. ดึง chunks มาเยอะๆ (เช่น 15 อัน) → เร็ว แต่หยาบ
# 2. จัดอันดับใหม่ด้วย model ที่แม่นกว่า → ช้าหน่อย แต่แม่น
# 3. เลือก top 3 → ส่งให้ AI

from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import CrossEncoderReranker
from langchain_community.cross_encoders import HuggingFaceCrossEncoder

# ==== 1. สร้าง Cross-Encoder ====
cross_encoder = HuggingFaceCrossEncoder(
    model_name="cross-encoder/ms-marco-MiniLM-L-6-v2"
    # Cross-Encoder คืออะไร?
    # = model พิเศษที่เทียบ "คำถาม" กับ "เอกสาร" ทีละคู่
    # แม่นกว่า embedding similarity มาก
    # แต่ช้ากว่า → เลยใช้ตอน rerank (เอกสารน้อยแล้ว)
)

compressor = CrossEncoderReranker(
    model=cross_encoder,
    top_n=3,  # เลือก top 3 หลัง rerank
)

# ==== 2. สร้าง Base Retriever ที่ดึงมาเยอะ ====
base_retriever = vectorstore.as_retriever(search_kwargs={"k": 10})
# ดึงมา 10 อันก่อน (หยาบๆ)

# ==== 3. รวมกัน ====
reranking_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,      # ใช้ cross-encoder จัดอันดับ
    base_retriever=base_retriever,   # ดึงมาจากตัวนี้
)
# Flow: ดึง 10 → rerank → เลือก top 3

# ==== 4. ทดสอบ ====
query = "สวัสดิการด้านสุขภาพ"

print("📥 Before reranking (10 อัน):")
raw = base_retriever.invoke(query)
for i, doc in enumerate(raw[:5], 1):
    print(f"  {i}. {doc.page_content[:50]}...")

print(f"\n📊 After reranking (top 3):")
reranked = reranking_retriever.invoke(query)
for i, doc in enumerate(reranked, 1):
    print(f"  {i}. {doc.page_content[:50]}...")
    # → ได้แค่อันที่เกี่ยวข้องจริงๆ!
```

### Re-ranking ด้วย LLM (ถ้าไม่อยากลง model)

```python
# file: 29_llm_rerank.py

def llm_rerank(query: str, docs: list, top_n: int = 3) -> list:
    """ใช้ LLM จัดอันดับแทน Cross-Encoder"""

    doc_list = "\n".join([f"[{i}] {d.page_content[:150]}" for i, d in enumerate(docs)])

    response = model.invoke(f"""จัดอันดับเอกสารตามความเกี่ยวข้องกับคำถาม:
คำถาม: "{query}"

เอกสาร:
{doc_list}

ตอบเฉพาะหมายเลข คั่นด้วย comma เช่น: 2,0,5""")

    try:
        indices = [int(x.strip()) for x in response.content.split(",")]
        return [docs[i] for i in indices[:top_n] if i < len(docs)]
    except:
        return docs[:top_n]

# ใช้งาน
raw_docs = base_retriever.invoke("สวัสดิการ")
reranked = llm_rerank("สวัสดิการด้านสุขภาพ", raw_docs, top_n=3)
```

---

## 5.5 Multi-Query — ค้นหาหลายมุมมอง

```python
# file: 30_multi_query.py

from langchain.retrievers.multi_query import MultiQueryRetriever

# แนวคิด:
# คำถามเดียวอาจไม่ครอบคลุม
# ให้ LLM สร้างคำถามหลายเวอร์ชัน → ค้นหาทุกเวอร์ชัน → รวมผลลัพธ์
#
# ตัวอย่าง:
# คำถามเดิม: "สวัสดิการด้านสุขภาพมีอะไรบ้าง"
# LLM สร้าง:
#   1. "ประกันสุขภาพบริษัทครอบคลุมอะไร"
#   2. "วงเงินค่ารักษาพยาบาลเท่าไหร่"
#   3. "มีสวัสดิการทันตกรรมไหม"
# → ค้นหาทั้ง 3 → ได้ผลลัพธ์ครอบคลุมมากขึ้น

multi_retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(search_kwargs={"k": 3}),
    llm=model,
    # LLM จะสร้าง 3 คำถาม variation อัตโนมัติ
)

# เปิด log เพื่อดูว่า LLM สร้างคำถามอะไร
import logging
logging.basicConfig()
logging.getLogger("langchain.retrievers.multi_query").setLevel(logging.INFO)

results = multi_retriever.invoke("สวัสดิการด้านสุขภาพมีอะไรบ้าง")
print(f"📊 ดึงได้ {len(results)} chunks (deduplicated)")
```

**Log Output:**
```
INFO: Generated queries:
['ประกันสุขภาพของบริษัทครอบคลุมอะไรบ้าง',
 'วงเงินค่ารักษาพยาบาล OPD IPD เท่าไหร่',
 'สวัสดิการด้านทันตกรรมและตรวจสุขภาพ']
📊 ดึงได้ 6 chunks (deduplicated)
```

---

## 5.6 Hybrid + Re-ranking (Best Combo)

```python
# file: 31_best_combo.py

# รวมทุกเทคนิคเข้าด้วยกัน:
# Hybrid Search (keyword + semantic) → ดึงมาเยอะ
# Re-ranking → จัดอันดับใหม่ เลือก top 3

# Step 1: Hybrid base (ดึงเยอะ)
hybrid_base = EnsembleRetriever(
    retrievers=[
        BM25Retriever.from_documents(chunks, k=10),
        vectorstore.as_retriever(search_kwargs={"k": 10}),
    ],
    weights=[0.4, 0.6],
)

# Step 2: Re-rank (เลือก top 3)
best_retriever = ContextualCompressionRetriever(
    base_compressor=CrossEncoderReranker(model=cross_encoder, top_n=3),
    base_retriever=hybrid_base,
)

# ใช้กับ RAG Chain
def ask_advanced(question: str) -> str:
    docs = best_retriever.invoke(question)
    context = "\n\n".join([d.page_content for d in docs])
    sources = list(set([d.metadata.get("source", "?") for d in docs]))

    formatted = rag_prompt.invoke({"context": context, "question": question})
    response = model.invoke(formatted)

    return f"{response.content}\n📎 อ้างอิง: {', '.join(sources)}"

print(ask_advanced("WFH policy 2025"))
print(ask_advanced("ทำงานจากบ้านได้ไหม"))
print(ask_advanced("OPD วงเงินเท่าไหร่"))
```

---

## 5.7 Conversational RAG — RAG ที่จำบทสนทนา

```python
# file: 32_conversational_rag.py

# ปัญหา:
# User: "ลาพักร้อนได้กี่วัน?"     → AI: "15 วัน"
# User: "แล้วสะสมได้ไหม?"         → AI: "???" ← ไม่รู้ว่า "สะสม" หมายถึงอะไร
#                                    เพราะค้นหาแค่ "สะสมได้ไหม" → ไม่เจอ!
#
# วิธีแก้:
# แปลง "สะสมได้ไหม" → "สะสมวันลาพักร้อนได้ไหม" ก่อนค้นหา

from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

# Prompt สำหรับแปลงคำถาม follow-up
condense_prompt = ChatPromptTemplate.from_messages([
    ("system", """ดูประวัติบทสนทนาและคำถามล่าสุด
แปลงคำถามล่าสุดให้เป็นคำถามที่สมบูรณ์ในตัวเอง (ไม่ต้องดู history ก็เข้าใจ)
ตอบแค่คำถามที่แปลงแล้ว ไม่ต้องอธิบาย"""),
    MessagesPlaceholder("chat_history"),
    ("human", "{question}"),
])

class ConversationalRAG:
    def __init__(self, retriever, model):
        self.retriever = retriever
        self.model = model
        self.history = []

    def ask(self, question: str) -> str:
        # ถ้ามี history → แปลงคำถามก่อน
        if self.history:
            condensed = self.model.invoke(
                condense_prompt.invoke({
                    "chat_history": self.history,
                    "question": question,
                })
            )
            search_query = condensed.content
            print(f"  🔄 แปลงเป็น: '{search_query}'")
        else:
            search_query = question

        # ค้นหา + ตอบ
        docs = self.retriever.invoke(search_query)
        context = "\n".join([d.page_content for d in docs])

        formatted = rag_prompt.invoke({"context": context, "question": question})
        response = self.model.invoke(formatted)

        # เก็บ history
        self.history.append({"role": "user", "content": question})
        self.history.append({"role": "assistant", "content": response.content})

        return response.content

# ใช้งาน
rag = ConversationalRAG(best_retriever, model)

print(rag.ask("ลาพักร้อนได้กี่วัน?"))
# → "15 วันต่อปี"

print(rag.ask("แล้วสะสมได้ไหม?"))
# 🔄 แปลงเป็น: 'สะสมวันลาพักร้อนได้ไหม'
# → "สะสมได้ไม่เกิน 5 วัน"

print(rag.ask("ต้องแจ้งล่วงหน้ากี่วัน?"))
# 🔄 แปลงเป็น: 'ลาพักร้อนต้องแจ้งล่วงหน้ากี่วัน'
# → "แจ้งล่วงหน้าอย่างน้อย 3 วันทำการ"
```

---

## สรุป Step 5

```
✅ Hybrid Search (BM25 + Semantic) → ค้นหาแม่นทั้งคำเฉพาะและความหมาย
✅ Re-ranking (Cross-Encoder / LLM) → จัดอันดับผลลัพธ์ให้แม่นยำขึ้น
✅ Multi-Query → ค้นหาหลายมุมมอง ครอบคลุมมากขึ้น
✅ Hybrid + Re-ranking combo → ดีที่สุดสำหรับ production
✅ Conversational RAG → จำบทสนทนา แปลงคำถาม follow-up
```

### ไป Step 6 → Multi-Agent Systems
