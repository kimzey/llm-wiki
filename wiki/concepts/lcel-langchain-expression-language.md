---
title: "LCEL — LangChain Expression Language"
type: concept
tags: [langchain, lcel, chain, pipe-operator, runnable, composability]
sources: [langchain_full.md, langchain_basics.md, langchain-llamaindex-deep-dive.md]
related: [wiki/concepts/langchain-framework.md, wiki/concepts/rag-retrieval-augmented-generation.md]
created: 2026-04-14
updated: 2026-04-14
---

## สรุปสั้น

LCEL (LangChain Expression Language) คือ syntax ใน LangChain ที่ใช้สัญลักษณ์ `|` (pipe) เชื่อม components เข้าด้วยกันแบบ composable — เหมือน Unix pipe — output ของซ้ายกลายเป็น input ของขวา รองรับ `.invoke()`, `.stream()`, `.batch()`, `.ainvoke()` ทุก chain ที่สร้างด้วย LCEL

## อธิบาย

LCEL เป็น core pattern ของ LangChain สมัยใหม่ (แทน `LLMChain` รุ่นเก่า) ทุก component ใน LangChain ที่ใช้ LCEL จะ implement `Runnable` interface ซึ่งมี method มาตรฐาน: `.invoke()`, `.stream()`, `.batch()`, `.ainvoke()`

```python
# pattern พื้นฐาน
chain = prompt | llm | parser

# invoke
result = chain.invoke({"question": "สวัสดี"})

# stream
for chunk in chain.stream({"question": "สวัสดี"}):
    print(chunk, end="")

# batch — ส่งหลายอย่างพร้อมกัน
results = chain.batch([{"question": "A"}, {"question": "B"}])
```

## ประเด็นสำคัญ

- **`|` operator**: เชื่อม Runnable objects — `chain = A | B | C` หมายความว่า output ของ A ส่งเป็น input ของ B
- **`RunnablePassthrough`**: ส่ง input ผ่านไปโดยไม่เปลี่ยนแปลง — ใช้บ่อยใน RAG chain เพื่อส่ง question ผ่านไปพร้อมกับ context
- **`RunnableLambda`**: ห่อ Python function ธรรมดาให้กลายเป็น Runnable สามารถใส่ใน chain ได้
- **`RunnableParallel`**: รันหลาย chain พร้อมกัน รวม output เป็น dict
- **`.assign()`**: เพิ่ม key ใหม่เข้า dict โดยไม่ทำลาย key เดิม
- **`.bind()`**: กำหนด kwargs ให้ Runnable ล่วงหน้า เช่น `llm.bind(stop=["END"])` หรือ `llm.bind_tools([...])`
- **`itemgetter`**: ใช้ดึง specific key จาก dict input ใน chain

## ตัวอย่าง / กรณีศึกษา

### RAG Chain ด้วย LCEL

```python
rag_chain = (
    {
        "context": retriever | format_docs,   # ← RunnableParallel implied
        "question": RunnablePassthrough(),
    }
    | prompt
    | llm
    | StrOutputParser()
)

answer = rag_chain.invoke("รายได้ปี 2567 เป็นเท่าไร?")
```

### Parallel Chain

```python
from langchain_core.runnables import RunnableParallel

parallel = RunnableParallel({
    "summary": summary_chain,
    "keywords": keyword_chain,
})
result = parallel.invoke({"text": "..."})
# result["summary"] และ result["keywords"] คำนวณพร้อมกัน
```

### Routing ด้วย RunnableLambda

```python
def route(info):
    if "ลา" in info["question"]:
        return hr_chain
    elif "โค้ด" in info["question"]:
        return tech_chain
    else:
        return general_chain

routed_chain = RunnableLambda(route)
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/langchain-framework|LangChain Framework]] — LCEL เป็น core syntax ของ LangChain
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — LCEL ใช้สร้าง RAG chain แบบ composable

## แหล่งที่มา

- [[wiki/sources/langchain-full-reference|LangChain คู่มือสมบูรณ์]] — PART 4 อธิบาย LCEL ทุก Runnable type
- [[wiki/sources/langchain-basics-openrag|LangChain พื้นฐาน]] — บทที่ 3 Chain ด้วย LCEL
- [[wiki/sources/openrag-langchain-llamaindex-deepdive|OpenRAG LangChain & LlamaIndex Deep Dive]] — Block 4 LCEL
