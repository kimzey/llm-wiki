---
title: "Step 5: Advanced RAG — Hybrid Search, Re-ranking, Multi-query"
type: source
source_file: raw/notes/LangChain/step5-advanced-rag-tutorial.md
tags: [langchain, rag, advanced, hybrid-search, reranking]
related: []
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/langchain/step5-advanced-rag-tutorial.md|Original file]]

## สรุป

บทช่วยสอน Advanced RAG สอนเทคนิคทำ RAG ให้แม่นยำขึ้น ลด hallucination เนื้อหาครอบคลุม Hybrid Search (BM25 + Semantic), Re-ranking (Cross-Encoder / LLM), Multi-Query, และ Conversational RAG

## ประเด็นสำคัญ

### ปัญหาของ RAG พื้นฐาน
1. ค้นหาไม่เจอ (เช่น "WFH policy 2025" → Semantic search ไม่รู้จักคำย่อ)
2. ได้ chunks ซ้ำๆ (ดึงมา 5 chunks → 3 อันพูดเรื่องเดียวกัน)
3. chunks ที่ดึงมาไม่ตรงประเด็น

### Hybrid Search (BM25 + Semantic)
- **BM25 (Keyword Retriever)**:
  - อัลกอริทึมค้นหาแบบ "นับคำ" (keyword matching)
  - เก่งเรื่อง: คำเฉพาะ (ชื่อคน, รหัส, ตัวย่อ เช่น WFH, OPD, VPN)
  - ไม่เก่งเรื่อง: ความหมาย ("ทำงานจากบ้าน" ≠ "WFH")

- **Semantic Retriever**:
  - ค้นหาด้วย "ความหมาย" (vector similarity)
  - เก่งเรื่อง: "ทำงานจากบ้าน" = "WFH" = "remote work"
  - ไม่เก่งเรื่อง: คำเฉพาะที่ต้อง match ตรงๆ

- **EnsembleRetriever**:
  ```python
  EnsembleRetriever(
      retrievers=[bm25_retriever, semantic_retriever],
      weights=[0.4, 0.6]  # น้ำหนักของแต่ละ retriever
  )
  ```
  - `[0.4, 0.6]` = semantic มากกว่า keyword นิดหน่อย
  - `[0.5, 0.5]` = เท่ากัน
  - `[0.7, 0.3]` = keyword มากกว่า (เหมาะกับข้อมูลที่มีศัพท์เทคนิคเยอะ)

### Re-ranking
- **แนวคิด**:
  1. ดึง chunks มาเยอะๆ (เช่น 15 อัน) → เร็ว แต่หยาบ
  2. จัดอันดับใหม่ด้วย model ที่แม่นกว่า → ช้าหน่อย แต่แม่น
  3. เลือก top 3 → ส่งให้ AI

- **Cross-Encoder**:
  - Model พิเศษที่เทียบ "คำถาม" กับ "เอกสาร" ทีละคู่
  - แม่นกว่า embedding similarity มาก
  - แต่ช้ากว่า → เลยใช้ตอน rerank (เอกสารน้อยแล้ว)
  - Model: `cross-encoder/ms-marco-MiniLM-L-6-v2`

- **ContextualCompressionRetriever**:
  ```python
  ContextualCompressionRetriever(
      base_compressor=CrossEncoderReranker(model=cross_encoder, top_n=3),
      base_retriever=base_retriever  # ดึงมา 10 อันก่อน
  )
  ```
  - Flow: ดึง 10 → rerank → เลือก top 3

- **Re-ranking ด้วย LLM** (ถ้าไม่อยากลง model):
  - ใช้ LLM จัดอันดับแทน Cross-Encoder
  - Prompt: "จัดอันดับเอกสารตามความเกี่ยวข้องกับคำถาม"

### Multi-Query
- **แนวคิด**: คำถามเดียวอาจไม่ครอบคลุม
- ให้ LLM สร้างคำถามหลายเวอร์ชัน → ค้นหาทุกเวอร์ชัน → รวมผลลัพธ์
- ตัวอย่าง:
  - คำถามเดิม: "สวัสดิการด้านสุขภาพมีอะไรบ้าง"
  - LLM สร้าง:
    1. "ประกันสุขภาพบริษัทครอบคลุมอะไร"
    2. "วงเงินค่ารักษาพยาบาลเท่าไหร่"
    3. "มีสวัสดิการทันตกรรมไหม"
  - ค้นหาทั้ง 3 → ได้ผลลัพธ์ครอบคลุมมากขึ้น

```python
MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(search_kwargs={"k": 3}),
    llm=model  # LLM จะสร้าง 3 คำถาม variation อัตโนมัติ
)
```

### Best Combo: Hybrid + Re-ranking
```python
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
```

### Conversational RAG (จำบทสนทนา)
- **ปัญหา**:
  - User: "ลาพักร้อนได้กี่วัน?" → AI: "15 วัน"
  - User: "แล้วสะสมได้ไหม?" → AI: "???" ← ไม่รู้ว่า "สะสม" หมายถึงอะไร
- **วิธีแก้**: แปลง "สะสมได้ไหม" → "สะสมวันลาพักร้อนได้ไหม" ก่อนค้นหา
- ใช้ `condense_prompt` ให้ LLM แปลงคำถาม follow-up ให้เป็นคำถามที่สมบูรณ์ในตัวเอง

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- ค้นหา "ทำงานจากบ้านได้ไหม" (ไม่มีคำว่า "WFH"):
  - BM25: อาจไม่เจอ WFH เพราะไม่มีคำตรง
  - Semantic: เจอ WFH เพราะเข้าใจความหมาย
  - Hybrid: ได้ทั้งสองอย่าง
- ดึง 10 chunks → After reranking → ได้แค่อันที่เกี่ยวข้องจริงๆ (top 3)

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อมูลที่ขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/agentic-rag|Agentic RAG]]
