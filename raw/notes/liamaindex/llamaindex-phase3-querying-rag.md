# LlamaIndex Deep Dive — Phase 3: Querying, Retrieval & RAG Pipeline

> **ซีรีส์นี้แบ่งเป็น 4 Phase:**
> - Phase 1 → Introduction & Core Concepts
> - Phase 2 → Data Ingestion & Indexing
> - **Phase 3** → Querying, Retrieval & RAG Pipeline *(ไฟล์นี้)*
> - Phase 4 → Advanced Features, Agents & Production

---

## 1. Query Engine vs Chat Engine vs Retriever

| | Query Engine | Chat Engine | Retriever |
|---|---|---|---|
| **Memory** | ไม่มี (single-turn) | มี (multi-turn) | ไม่มี |
| **Output** | Response object | Response object | List of Nodes |
| **ใช้เมื่อ** | Q&A ธรรมดา | Chatbot | custom pipeline |

---

## 2. Retrievers

Retriever คือ component ที่ **ค้นหา relevant nodes** จาก index

### 2.1 VectorIndexRetriever (default)

```python
from llama_index.core import VectorStoreIndex
from llama_index.core.retrievers import VectorIndexRetriever

index = VectorStoreIndex.from_documents(documents)

retriever = VectorIndexRetriever(
    index=index,
    similarity_top_k=5,   # ดึงมา 5 nodes
)

# retrieve
nodes = retriever.retrieve("LlamaIndex คืออะไร?")
for node in nodes:
    print(f"Score: {node.score:.4f}")
    print(f"Text: {node.text[:200]}")
    print("---")
```

### 2.2 BM25Retriever (keyword-based)

```python
pip install llama-index-retrievers-bm25
```

```python
from llama_index.retrievers.bm25 import BM25Retriever

# สร้างจาก nodes โดยตรง
bm25_retriever = BM25Retriever.from_defaults(
    nodes=nodes,          # list of nodes
    similarity_top_k=5
)

nodes = bm25_retriever.retrieve("keyword search query")
```

### 2.3 Hybrid Retriever (Vector + BM25)

ดีที่สุดสำหรับงานทั่วไป รวม semantic search กับ keyword search

```python
from llama_index.core.retrievers import QueryFusionRetriever
from llama_index.retrievers.bm25 import BM25Retriever

vector_retriever = index.as_retriever(similarity_top_k=5)
bm25_retriever = BM25Retriever.from_defaults(nodes=nodes, similarity_top_k=5)

hybrid_retriever = QueryFusionRetriever(
    retrievers=[vector_retriever, bm25_retriever],
    similarity_top_k=5,
    num_queries=1,        # จำนวน query variations ที่ generate
    mode="reciprocal_rerank",   # วิธี combine: reciprocal_rerank, relative_score, dist_based_score, simple
    use_async=True
)

nodes = hybrid_retriever.retrieve("คำถาม")
```

### 2.4 AutoMergingRetriever (Parent-Child)

ดึง leaf nodes แต่ return parent node ถ้า children ครบ threshold

```python
from llama_index.core.retrievers import AutoMergingRetriever
from llama_index.core.node_parser import HierarchicalNodeParser, get_leaf_nodes
from llama_index.core import StorageContext

# Setup hierarchical nodes
parser = HierarchicalNodeParser.from_defaults(chunk_sizes=[2048, 512, 128])
nodes = parser.get_nodes_from_documents(documents)
leaf_nodes = get_leaf_nodes(nodes)

# สร้าง docstore ที่มีทุก nodes (ทั้ง parent + leaf)
docstore = SimpleDocumentStore()
docstore.add_documents(nodes)

storage_context = StorageContext.from_defaults(docstore=docstore)

# Index เฉพาะ leaf nodes
index = VectorStoreIndex(leaf_nodes, storage_context=storage_context)

# AutoMerging retriever
base_retriever = index.as_retriever(similarity_top_k=6)
retriever = AutoMergingRetriever(
    simple_retriever=base_retriever,
    storage_context=storage_context,
    verbose=True
)

nodes = retriever.retrieve("คำถาม")
```

### 2.5 RecursiveRetriever

ดึง nodes แล้ว follow references ไปยัง related nodes

```python
from llama_index.core.retrievers import RecursiveRetriever

retriever = RecursiveRetriever(
    "vector",
    retriever_dict={"vector": vector_retriever},
    node_dict=all_nodes_dict,  # dict[node_id → node]
    verbose=True
)

nodes = retriever.retrieve("คำถาม")
```

---

## 3. Node Postprocessors (Re-ranking & Filtering)

หลังจาก retrieve nodes แล้ว สามารถ filter/rerank เพิ่มเติมได้

### 3.1 SimilarityPostprocessor

```python
from llama_index.core.postprocessor import SimilarityPostprocessor

# filter nodes ที่ score < threshold
postprocessor = SimilarityPostprocessor(similarity_cutoff=0.7)
```

### 3.2 KeywordNodePostprocessor

```python
from llama_index.core.postprocessor import KeywordNodePostprocessor

postprocessor = KeywordNodePostprocessor(
    required_keywords=["LlamaIndex"],    # ต้องมี keywords นี้
    exclude_keywords=["deprecated"]      # ห้ามมี keywords นี้
)
```

### 3.3 Cohere Reranker (ดีที่สุด)

```python
pip install llama-index-postprocessor-cohere-rerank cohere
```

```python
from llama_index.postprocessor.cohere_rerank import CohereRerank

reranker = CohereRerank(
    api_key="COHERE_API_KEY",
    top_n=3   # เลือก top 3 หลัง rerank
)

query_engine = index.as_query_engine(
    similarity_top_k=10,       # ดึง 10 มาก่อน
    node_postprocessors=[reranker]  # แล้ว rerank เหลือ 3
)
```

### 3.4 LLM Reranker

```python
from llama_index.core.postprocessor import LLMRerank

reranker = LLMRerank(
    choice_batch_size=5,
    top_n=3
)

query_engine = index.as_query_engine(
    similarity_top_k=10,
    node_postprocessors=[reranker]
)
```

### 3.5 ใช้หลาย Postprocessors พร้อมกัน

```python
query_engine = index.as_query_engine(
    similarity_top_k=20,
    node_postprocessors=[
        SimilarityPostprocessor(similarity_cutoff=0.5),  # ขั้นที่ 1
        KeywordNodePostprocessor(required_keywords=["AI"]),  # ขั้นที่ 2
        CohereRerank(top_n=5)  # ขั้นที่ 3
    ]
)
```

---

## 4. Response Synthesizers

หลังจาก retrieve nodes แล้ว synthesizer จะ **สังเคราะห์คำตอบ**

### Response Modes

```python
query_engine = index.as_query_engine(
    response_mode="compact"  # เลือก mode
)
```

| Mode | คำอธิบาย | เมื่อไรใช้ |
|---|---|---|
| `refine` | LLM call ทีละ node แล้ว refine | เนื้อหาซับซ้อน |
| `compact` | รวม nodes แล้ว call ครั้งเดียว (default) | งานทั่วไป |
| `tree_summarize` | สรุปแบบ tree (bottom-up) | summarization |
| `simple_summarize` | ตัด nodes ให้ fit context แล้ว summarize | เร็ว |
| `accumulate` | รวม responses จากทุก node | เปรียบเทียบ |
| `compact_accumulate` | compact + accumulate | |
| `no_text` | return nodes โดยไม่ LLM | debug |
| `generation` | ignore context, LLM ตอบเอง | |

```python
from llama_index.core.response_synthesizers import TreeSummarize, Refine

# ใช้ TreeSummarize โดยตรง
summarizer = TreeSummarize(verbose=True)

# ใช้กับ query engine
from llama_index.core import get_response_synthesizer

synthesizer = get_response_synthesizer(
    response_mode="tree_summarize",
    use_async=True
)

query_engine = RetrieverQueryEngine(
    retriever=retriever,
    response_synthesizer=synthesizer
)
```

---

## 5. Query Engine ขั้นสูง

### 5.1 RetrieverQueryEngine (Manual Assembly)

```python
from llama_index.core.query_engine import RetrieverQueryEngine
from llama_index.core import get_response_synthesizer
from llama_index.core.postprocessor import SimilarityPostprocessor

retriever = index.as_retriever(similarity_top_k=5)
synthesizer = get_response_synthesizer(response_mode="compact")

query_engine = RetrieverQueryEngine(
    retriever=retriever,
    response_synthesizer=synthesizer,
    node_postprocessors=[SimilarityPostprocessor(similarity_cutoff=0.6)]
)

response = query_engine.query("คำถาม")
```

### 5.2 SubQuestionQueryEngine

แยกคำถามซับซ้อนออกเป็นคำถามย่อย แล้วรวมคำตอบ

```python
from llama_index.core.query_engine import SubQuestionQueryEngine
from llama_index.core.tools import QueryEngineTool

# ต้องมีหลาย engines (หรือ engine เดียว)
query_engine_tools = [
    QueryEngineTool.from_defaults(
        query_engine=index.as_query_engine(),
        name="company_docs",
        description="ข้อมูลเอกสารบริษัท"
    )
]

sub_query_engine = SubQuestionQueryEngine.from_defaults(
    query_engine_tools=query_engine_tools,
    verbose=True
)

response = sub_query_engine.query(
    "เปรียบเทียบรายได้ปี 2023 กับ 2022 และอธิบายสาเหตุที่เปลี่ยนแปลง"
)
# LLM จะแยกเป็น: 1) รายได้ปี 2023, 2) รายได้ปี 2022, 3) สาเหตุ
```

### 5.3 RouterQueryEngine

เลือก query engine ที่เหมาะสมโดยอัตโนมัติ

```python
from llama_index.core.query_engine import RouterQueryEngine
from llama_index.core.selectors import LLMSingleSelector, LLMMultiSelector
from llama_index.core.tools import QueryEngineTool

# สร้างหลาย engines
vector_engine = vector_index.as_query_engine()
summary_engine = summary_index.as_query_engine(response_mode="tree_summarize")

tools = [
    QueryEngineTool.from_defaults(
        query_engine=vector_engine,
        description="ใช้สำหรับคำถามเฉพาะเจาะจง ค้นหาข้อมูล"
    ),
    QueryEngineTool.from_defaults(
        query_engine=summary_engine,
        description="ใช้สำหรับสรุปเนื้อหาทั้งหมด"
    )
]

router_engine = RouterQueryEngine(
    selector=LLMSingleSelector.from_defaults(),
    query_engine_tools=tools,
    verbose=True
)

response = router_engine.query("สรุปทั้งเอกสาร")  # → เลือก summary_engine
response = router_engine.query("ราคาของ product A คือเท่าไร?")  # → เลือก vector_engine
```

### 5.4 MultiStepQueryEngine

ถามคำถามซ้ำๆ จนได้คำตอบที่ต้องการ (iterative)

```python
from llama_index.core.query_engine import MultiStepQueryEngine
from llama_index.core.indices.query.query_transform import StepDecomposeQueryTransform

step_decompose_transform = StepDecomposeQueryTransform(verbose=True)

query_engine = MultiStepQueryEngine(
    query_engine=index.as_query_engine(similarity_top_k=3),
    query_transform=step_decompose_transform,
    index_summary="เอกสารเกี่ยวกับประวัติบริษัท"
)

response = query_engine.query(
    "ใครเป็น CEO คนปัจจุบัน และเขาเข้าร่วมบริษัทเมื่อปีไหน?"
)
```

---

## 6. Chat Engine

Chat Engine เพิ่ม **memory** ให้กับ query engine (multi-turn conversation)

### 6.1 SimpleChatEngine (ไม่มี context จาก index)

```python
from llama_index.core.chat_engine import SimpleChatEngine

chat_engine = SimpleChatEngine.from_defaults()

response = chat_engine.chat("สวัสดี ฉันชื่อ John")
print(response)

response = chat_engine.chat("ฉันชื่ออะไร?")
print(response)  # จำชื่อ John ได้
```

### 6.2 ContextChatEngine (มี index context)

```python
chat_engine = index.as_chat_engine(
    chat_mode="context",
    memory_token_limit=4096,
    context_template=(
        "ข้อมูลที่เกี่ยวข้อง:\n"
        "{context_str}"
        "\n\nใช้ข้อมูลข้างต้นเพื่อตอบคำถาม"
    )
)

response = chat_engine.chat("LlamaIndex คืออะไร?")
print(response)

response = chat_engine.chat("มีกี่ประเภทของ Index?")  # จำ context เดิม
print(response)

# Reset memory
chat_engine.reset()
```

### 6.3 CondensePlusContextChatEngine (แนะนำ)

Condense คำถามก่อน (รวม history) แล้วค้นหา context

```python
chat_engine = index.as_chat_engine(
    chat_mode="condense_plus_context",  # แนะนำ
    similarity_top_k=5
)
```

### 6.4 OpenAIAgent Chat Engine

```python
chat_engine = index.as_chat_engine(
    chat_mode="openai",
    verbose=True
)
```

### 6.5 Custom Memory

```python
from llama_index.core.memory import ChatMemoryBuffer

memory = ChatMemoryBuffer.from_defaults(
    token_limit=4096,
)

chat_engine = index.as_chat_engine(
    chat_mode="context",
    memory=memory,
    verbose=True
)

# ดู chat history
print(chat_engine.chat_history)

# Save/load memory
messages = memory.get()
# restore
memory.set(messages)
```

---

## 7. Advanced RAG Techniques

### 7.1 Query Transformation: HyDE

HyDE (Hypothetical Document Embeddings) — generate สมมติคำตอบก่อน แล้วค้นหาจาก embedding ของสมมติคำตอบนั้น

```python
from llama_index.core.indices.query.query_transform import HyDEQueryTransform
from llama_index.core.query_engine import TransformQueryEngine

hyde = HyDEQueryTransform(include_original=True)

base_query_engine = index.as_query_engine()
hyde_query_engine = TransformQueryEngine(base_query_engine, hyde)

response = hyde_query_engine.query(
    "LlamaIndex ใช้อัลกอริทึมอะไรในการ retrieve?"
)
```

### 7.2 Query Rewriting

```python
from llama_index.core.indices.query.query_transform import DecomposeQueryTransform

decompose_transform = DecomposeQueryTransform(verbose=True)

query_engine = TransformQueryEngine(
    base_query_engine=index.as_query_engine(),
    query_transform=decompose_transform
)
```

### 7.3 Sentence Window Retrieval

ดึง node ที่ match แต่ return window ของ sentences รอบๆ ด้วย

```python
from llama_index.core.node_parser import SentenceWindowNodeParser
from llama_index.core.postprocessor import MetadataReplacementPostProcessor

# 1. Parse ด้วย SentenceWindowNodeParser
parser = SentenceWindowNodeParser.from_defaults(
    window_size=3,           # จำนวน sentences รอบๆ
    window_metadata_key="window",
    original_text_metadata_key="original_text"
)
nodes = parser.get_nodes_from_documents(documents)

# 2. สร้าง index
index = VectorStoreIndex(nodes)

# 3. Query Engine พร้อม MetadataReplacement
query_engine = index.as_query_engine(
    similarity_top_k=6,
    node_postprocessors=[
        MetadataReplacementPostProcessor(target_metadata_key="window")
        # แทนที่ node.text ด้วย window (ข้อความรอบๆ)
    ]
)

response = query_engine.query("คำถาม")
```

### 7.4 RAG Fusion (Multiple Query Generation)

Generate หลาย query variations แล้วรวม results

```python
from llama_index.core.retrievers import QueryFusionRetriever

retriever = QueryFusionRetriever(
    retrievers=[index.as_retriever(similarity_top_k=5)],
    num_queries=4,           # generate 4 query variations
    mode="reciprocal_rerank",
    use_async=True,
    verbose=True
)

# LlamaIndex จะ generate:
# Original: "LlamaIndex คืออะไร?"
# Query 2: "อธิบาย LlamaIndex framework"
# Query 3: "LlamaIndex ทำงานยังไง?"
# Query 4: "การใช้งาน LlamaIndex"
nodes = retriever.retrieve("LlamaIndex คืออะไร?")
```

---

## 8. Evaluation

วัดประสิทธิภาพของ RAG pipeline

### 8.1 Faithfulness (คำตอบ hallucinate หรือเปล่า?)

```python
from llama_index.core.evaluation import FaithfulnessEvaluator

evaluator = FaithfulnessEvaluator()

response = query_engine.query("LlamaIndex คืออะไร?")

result = evaluator.evaluate_response(response=response)
print(f"Passing: {result.passing}")   # True/False
print(f"Score: {result.score}")       # 0-1
print(f"Feedback: {result.feedback}")
```

### 8.2 Relevancy (context ที่ retrieve มา relevant กับ query ไหม?)

```python
from llama_index.core.evaluation import RelevancyEvaluator

evaluator = RelevancyEvaluator()

result = evaluator.evaluate_response(
    query="LlamaIndex คืออะไร?",
    response=response
)
print(f"Passing: {result.passing}")
```

### 8.3 Correctness (คำตอบถูกต้องไหม? ต้องมี reference answer)

```python
from llama_index.core.evaluation import CorrectnessEvaluator

evaluator = CorrectnessEvaluator()

result = evaluator.evaluate(
    query="LlamaIndex คืออะไร?",
    response="LlamaIndex คือ data framework...",
    reference="LlamaIndex เป็น framework สำหรับ LLM applications"
)

print(f"Score: {result.score}")     # 1-5
print(f"Feedback: {result.feedback}")
```

### 8.4 Batch Evaluation

```python
from llama_index.core.evaluation import BatchEvalRunner

# สร้าง eval dataset
eval_questions = [
    "LlamaIndex คืออะไร?",
    "VectorStoreIndex ทำงานยังไง?",
    "RAG คืออะไร?"
]

runner = BatchEvalRunner(
    evaluators={
        "faithfulness": FaithfulnessEvaluator(),
        "relevancy": RelevancyEvaluator()
    },
    workers=4
)

eval_results = await runner.aevaluate_queries(
    query_engine=query_engine,
    queries=eval_questions
)

# สรุปผล
import pandas as pd
results_df = pd.DataFrame([
    {
        "query": q,
        "faithfulness": eval_results["faithfulness"][i].score,
        "relevancy": eval_results["relevancy"][i].score
    }
    for i, q in enumerate(eval_questions)
])
print(results_df)
```

---

## 9. Observability & Debugging

### 9.1 ดู Prompts ที่ใช้

```python
query_engine = index.as_query_engine()

# ดู prompts ทั้งหมด
prompts_dict = query_engine.get_prompts()
for key, prompt in prompts_dict.items():
    print(f"=== {key} ===")
    print(prompt.get_template())
    print()
```

### 9.2 Verbose Mode

```python
query_engine = index.as_query_engine(verbose=True)
response = query_engine.query("คำถาม")
# จะแสดง intermediate steps ทั้งหมด
```

### 9.3 LlamaTrace / Arize Phoenix

```python
pip install llama-index-callbacks-arize-phoenix arize-phoenix
```

```python
import phoenix as px
from llama_index.core import set_global_handler

px.launch_app()  # เปิด web UI ที่ localhost:6006
set_global_handler("arize_phoenix")

# ทุก query จะถูก trace โดยอัตโนมัติ
response = query_engine.query("คำถาม")
```

### 9.4 Callbacks

```python
from llama_index.core.callbacks import CallbackManager, LlamaDebugHandler
from llama_index.core import Settings

debug_handler = LlamaDebugHandler(print_trace_on_end=True)
Settings.callback_manager = CallbackManager([debug_handler])

# ทุก operation จะถูก log
response = query_engine.query("คำถาม")

# ดู events
event_pairs = debug_handler.get_llm_inputs_outputs()
for pair in event_pairs:
    print(f"Input: {pair[0].payload}")
    print(f"Output: {pair[1].payload}")
```

---

## 10. Workshop: Production-Ready RAG

```python
"""
Production RAG Pipeline พร้อม:
- HierarchicalNodeParser
- Hybrid Retrieval (Vector + BM25)
- CohereRerank
- Thai language prompt
- Streaming response
- Evaluation
"""
from llama_index.core import (
    VectorStoreIndex, SimpleDirectoryReader, Settings, StorageContext
)
from llama_index.core.node_parser import HierarchicalNodeParser, get_leaf_nodes
from llama_index.core.storage.docstore import SimpleDocumentStore
from llama_index.core.retrievers import QueryFusionRetriever, AutoMergingRetriever
from llama_index.core.query_engine import RetrieverQueryEngine
from llama_index.core import get_response_synthesizer
from llama_index.core.postprocessor import SimilarityPostprocessor
from llama_index.postprocessor.cohere_rerank import CohereRerank
from llama_index.core import PromptTemplate
from llama_index.llms.openai import OpenAI
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.retrievers.bm25 import BM25Retriever
import os

# Config
os.environ["OPENAI_API_KEY"] = "sk-..."
os.environ["COHERE_API_KEY"] = "..."

Settings.llm = OpenAI(model="gpt-4o", temperature=0)
Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")

# 1. Load & Parse
documents = SimpleDirectoryReader("./data").load_data()

parser = HierarchicalNodeParser.from_defaults(chunk_sizes=[2048, 512, 128])
all_nodes = parser.get_nodes_from_documents(documents)
leaf_nodes = get_leaf_nodes(all_nodes)

# 2. Storage
docstore = SimpleDocumentStore()
docstore.add_documents(all_nodes)
storage_context = StorageContext.from_defaults(docstore=docstore)

# 3. Index
index = VectorStoreIndex(leaf_nodes, storage_context=storage_context)

# 4. Hybrid Retrieval
vector_retriever = index.as_retriever(similarity_top_k=10)
bm25_retriever = BM25Retriever.from_defaults(nodes=leaf_nodes, similarity_top_k=10)

hybrid_retriever = QueryFusionRetriever(
    retrievers=[vector_retriever, bm25_retriever],
    similarity_top_k=10,
    num_queries=3,
    mode="reciprocal_rerank",
    use_async=True
)

# 5. AutoMerging
auto_merging_retriever = AutoMergingRetriever(
    simple_retriever=hybrid_retriever,
    storage_context=storage_context,
    verbose=True
)

# 6. Custom Thai Prompt
qa_prompt = PromptTemplate("""
คุณเป็นผู้ช่วย AI ที่ตอบคำถามจากเอกสารที่ให้มา

ข้อมูลที่เกี่ยวข้อง:
{context_str}

คำถาม: {query_str}

กรุณาตอบ:
1. ตอบตรงประเด็น
2. อ้างอิงจากข้อมูลที่ให้เท่านั้น
3. ถ้าไม่มีข้อมูล ให้บอกตรงๆ ว่า "ไม่มีข้อมูลในเอกสาร"

คำตอบ:
""")

# 7. Synthesizer + Query Engine
synthesizer = get_response_synthesizer(
    response_mode="compact",
    text_qa_template=qa_prompt
)

query_engine = RetrieverQueryEngine(
    retriever=auto_merging_retriever,
    response_synthesizer=synthesizer,
    node_postprocessors=[
        SimilarityPostprocessor(similarity_cutoff=0.4),
        CohereRerank(top_n=5)
    ]
)

# 8. Query
response = query_engine.query("คำถามของคุณ")
print(response)
for node in response.source_nodes:
    print(f"- Score: {node.score:.3f} | {node.text[:100]}")
```

---

## สรุป Phase 3

| Component | ใช้ทำอะไร |
|---|---|
| VectorIndexRetriever | ค้นหาด้วย semantic similarity |
| BM25Retriever | ค้นหาด้วย keyword |
| QueryFusionRetriever | Hybrid search |
| AutoMergingRetriever | Parent-child retrieval |
| CohereRerank | Re-rank results |
| Response Modes | วิธีสังเคราะห์คำตอบ |
| Chat Engine | Multi-turn conversation |
| HyDE | Query transformation |
| Evaluation | วัดคุณภาพ RAG |

---

## ขั้นตอนถัดไป

👉 **Phase 4: Advanced Features, Agents & Production** — Agents, Tools, Multi-modal, Fine-tuning, และการ deploy

---

*Phase 3/4 | LlamaIndex Deep Dive Series*
