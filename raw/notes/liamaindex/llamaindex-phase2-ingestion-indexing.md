# LlamaIndex Deep Dive — Phase 2: Data Ingestion & Indexing

> **ซีรีส์นี้แบ่งเป็น 4 Phase:**
> - Phase 1 → Introduction & Core Concepts
> - **Phase 2** → Data Ingestion & Indexing *(ไฟล์นี้)*
> - Phase 3 → Querying, Retrieval & RAG Pipeline
> - Phase 4 → Advanced Features, Agents & Production

---

## 1. Data Loaders (Readers)

LlamaIndex มี **LlamaHub** — repository ของ loaders กว่า 100+ ตัว สำหรับอ่านข้อมูลจากแหล่งต่างๆ

### 1.1 SimpleDirectoryReader (พื้นฐาน)

```python
from llama_index.core import SimpleDirectoryReader

# โหลดทุกไฟล์ในโฟลเดอร์
documents = SimpleDirectoryReader("./data").load_data()

# โหลด recursive (รวม subfolder)
documents = SimpleDirectoryReader(
    "./data",
    recursive=True
).load_data()

# โหลดเฉพาะ file extension
documents = SimpleDirectoryReader(
    "./data",
    required_exts=[".pdf", ".txt", ".md"]
).load_data()

# โหลดไฟล์เดียว
documents = SimpleDirectoryReader(
    input_files=["./data/report.pdf", "./data/notes.txt"]
).load_data()

# exclude ไฟล์บางชื่อ
documents = SimpleDirectoryReader(
    "./data",
    exclude=["*.tmp", "secret.txt"]
).load_data()
```

### 1.2 PDF Reader

```python
# วิธีที่ 1: ผ่าน SimpleDirectoryReader (auto-detect)
documents = SimpleDirectoryReader("./pdfs").load_data()

# วิธีที่ 2: ใช้ PDFReader โดยตรง
pip install llama-index-readers-file
```

```python
from llama_index.readers.file import PDFReader

reader = PDFReader()
documents = reader.load_data(file="./document.pdf")

# แต่ละ page จะเป็น 1 Document
for doc in documents:
    print(f"Page: {doc.metadata['page_label']}")
    print(doc.text[:200])
```

### 1.3 Web/URL Reader

```python
pip install llama-index-readers-web
```

```python
from llama_index.readers.web import SimpleWebPageReader

# โหลดจาก URL
loader = SimpleWebPageReader(html_to_text=True)
documents = loader.load_data(
    urls=["https://docs.llamaindex.ai/en/stable/"]
)

# Sitemap Reader (crawl ทั้ง website)
from llama_index.readers.web import SitemapReader
loader = SitemapReader()
documents = loader.load_data(sitemap_url="https://example.com/sitemap.xml")
```

### 1.4 Database Reader

```python
pip install llama-index-readers-database
```

```python
from llama_index.readers.database import DatabaseReader

reader = DatabaseReader(
    scheme="postgresql",    # mysql, sqlite, mssql
    host="localhost",
    port=5432,
    user="admin",
    password="secret",
    dbname="mydb"
)

# Query → Documents
documents = reader.load_data(
    query="SELECT title, content, created_at FROM articles"
)

# แต่ละ row จะเป็น 1 Document
print(documents[0].text)  # "title: ...\ncontent: ..."
```

### 1.5 Notion Reader

```python
pip install llama-index-readers-notion
```

```python
from llama_index.readers.notion import NotionPageReader

reader = NotionPageReader(integration_token="secret_...")

# โหลดจาก page IDs
documents = reader.load_data(
    page_ids=["abc123", "def456"]
)

# โหลดทั้ง database
documents = reader.load_data(
    database_id="xyz789"
)
```

### 1.6 Google Docs / Drive Reader

```python
pip install llama-index-readers-google
```

```python
from llama_index.readers.google import GoogleDocsReader, GoogleDriveReader

# Google Docs
loader = GoogleDocsReader()
documents = loader.load_data(document_ids=["GOOGLE_DOC_ID"])

# Google Drive (folder)
loader = GoogleDriveReader(folder_id="FOLDER_ID")
documents = loader.load_data()
```

### 1.7 JSONReader

```python
from llama_index.readers.file import JSONReader

reader = JSONReader(
    levels_back=0,        # จำนวน level ที่ flatten
    clean_json=True       # ทำ JSON ให้อ่านง่ายขึ้น
)
documents = reader.load_data(input_file="data.json")
```

### 1.8 CSV / Excel Reader

```python
from llama_index.readers.file import CSVReader, PandasExcelReader

# CSV
csv_reader = CSVReader(concat_rows=False)  # แต่ละ row = 1 doc
documents = csv_reader.load_data(file="data.csv")

# Excel
excel_reader = PandasExcelReader(pandas_config={"header": 0})
documents = excel_reader.load_data(file="data.xlsx")
```

### 1.9 สร้าง Custom Reader

```python
from llama_index.core.readers.base import BaseReader
from llama_index.core import Document
from typing import List

class MyCustomReader(BaseReader):
    def load_data(self, file_path: str, **kwargs) -> List[Document]:
        # อ่านข้อมูลตามต้องการ
        with open(file_path, "r") as f:
            content = f.read()
        
        # parse, transform ตามต้องการ
        sections = content.split("---")
        
        documents = []
        for i, section in enumerate(sections):
            doc = Document(
                text=section.strip(),
                metadata={"section": i, "source": file_path}
            )
            documents.append(doc)
        
        return documents

# ใช้งาน
reader = MyCustomReader()
docs = reader.load_data("myfile.txt")
```

---

## 2. Node Parsers (Text Splitters)

Node Parsers แบ่ง Document ออกเป็น Nodes (chunks) ก่อน indexing

### 2.1 SentenceSplitter (แนะนำสำหรับงานทั่วไป)

```python
from llama_index.core.node_parser import SentenceSplitter

parser = SentenceSplitter(
    chunk_size=1024,      # max tokens per chunk
    chunk_overlap=20,     # overlap tokens ระหว่าง chunks
    separator=" "         # แบ่งตรงไหน
)

nodes = parser.get_nodes_from_documents(documents)
print(f"จำนวน Nodes: {len(nodes)}")
print(f"ตัวอย่าง Node: {nodes[0].text[:200]}")
```

### 2.2 TokenTextSplitter

```python
from llama_index.core.node_parser import TokenTextSplitter

parser = TokenTextSplitter(
    chunk_size=512,
    chunk_overlap=50,
    separator="\n"
)
nodes = parser.get_nodes_from_documents(documents)
```

### 2.3 SemanticSplitterNodeParser (ฉลาดที่สุด)

แบ่ง chunk ตาม semantic similarity แทนที่จะแบ่งตาม token count

```python
pip install llama-index-node-parser-semantic-splitter
```

```python
from llama_index.node_parser.semantic_splitter import SemanticSplitterNodeParser
from llama_index.embeddings.openai import OpenAIEmbedding

parser = SemanticSplitterNodeParser(
    buffer_size=1,              # จำนวน sentences ที่รวมกัน
    breakpoint_percentile_threshold=95,  # threshold สำหรับแบ่ง
    embed_model=OpenAIEmbedding()
)

nodes = parser.get_nodes_from_documents(documents)
```

### 2.4 HierarchicalNodeParser

สร้าง hierarchy ของ nodes (ดีสำหรับ parent-child retrieval)

```python
from llama_index.core.node_parser import HierarchicalNodeParser, get_leaf_nodes

parser = HierarchicalNodeParser.from_defaults(
    chunk_sizes=[2048, 512, 128]  # จากใหญ่ไปเล็ก
)

nodes = parser.get_nodes_from_documents(documents)
leaf_nodes = get_leaf_nodes(nodes)  # nodes ที่เล็กที่สุด

print(f"Total nodes: {len(nodes)}")
print(f"Leaf nodes: {len(leaf_nodes)}")
```

### 2.5 MarkdownNodeParser

```python
from llama_index.core.node_parser import MarkdownNodeParser

parser = MarkdownNodeParser()
nodes = parser.get_nodes_from_documents(documents)

# แต่ละ section ของ Markdown จะเป็น Node แยก
```

### 2.6 SimpleFileNodeParser

```python
from llama_index.core.node_parser import SimpleFileNodeParser

# เลือก parser อัตโนมัติตาม file type
parser = SimpleFileNodeParser()
nodes = parser.get_nodes_from_documents(documents)
```

### 2.7 Metadata Extraction ระหว่าง Parsing

```python
from llama_index.core.extractors import (
    TitleExtractor,
    QuestionsAnsweredExtractor,
    SummaryExtractor,
    KeywordExtractor
)
from llama_index.core.node_parser import SentenceSplitter
from llama_index.core.ingestion import IngestionPipeline

pipeline = IngestionPipeline(
    transformations=[
        SentenceSplitter(chunk_size=512),
        TitleExtractor(nodes=5),              # extract title
        QuestionsAnsweredExtractor(questions=3),  # extract Q&A
        KeywordExtractor(keywords=5),         # extract keywords
    ]
)

nodes = pipeline.run(documents=documents)
print(nodes[0].metadata)
# {'document_title': '...', 'questions_this_excerpt_can_answer': '...', 'excerpt_keywords': '...'}
```

---

## 3. IngestionPipeline

`IngestionPipeline` คือ pipeline สำหรับ transform documents → nodes อย่างเป็นระบบ

```python
from llama_index.core.ingestion import IngestionPipeline
from llama_index.core.node_parser import SentenceSplitter
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.core.storage.docstore import SimpleDocumentStore

pipeline = IngestionPipeline(
    transformations=[
        SentenceSplitter(chunk_size=1024, chunk_overlap=20),
        OpenAIEmbedding(),  # embed nodes ด้วย
    ],
    docstore=SimpleDocumentStore(),  # track documents ที่ process แล้ว
)

nodes = pipeline.run(documents=documents)
```

### Cache และ Incremental Processing

```python
# Cache pipeline (ไม่ต้อง re-process ถ้า doc ไม่เปลี่ยน)
pipeline = IngestionPipeline(
    transformations=[SentenceSplitter(), OpenAIEmbedding()],
    cache=IngestionCache(collection="my_cache")
)

# ครั้งแรก: process ทั้งหมด
nodes = pipeline.run(documents=documents)

# ครั้งต่อไป: skip docs ที่ cache แล้ว (เร็วกว่ามาก!)
nodes = pipeline.run(documents=new_documents)
```

---

## 4. Index Types

### 4.1 VectorStoreIndex (ใช้บ่อยที่สุด)

เก็บ embeddings ใน vector store ค้นหาด้วย similarity search

```python
from llama_index.core import VectorStoreIndex

# สร้างจาก documents
index = VectorStoreIndex.from_documents(documents)

# สร้างจาก nodes (control มากกว่า)
index = VectorStoreIndex(nodes)

# เพิ่ม documents ทีหลัง
index.insert(new_document)
index.insert_nodes(new_nodes)

# ลบ document
index.delete_ref_doc("doc_id", delete_from_docstore=True)
```

### 4.2 SummaryIndex (ListIndex)

เก็บ nodes เรียงต่อกัน เหมาะสำหรับ summarization tasks

```python
from llama_index.core import SummaryIndex

index = SummaryIndex.from_documents(documents)
query_engine = index.as_query_engine()

# ดีสำหรับ: "สรุปทั้งเอกสาร"
response = query_engine.query("สรุปเนื้อหาทั้งหมด")
```

### 4.3 TreeIndex

สร้าง tree structure ของ summaries (bottom-up)

```python
from llama_index.core import TreeIndex

index = TreeIndex.from_documents(documents)
query_engine = index.as_query_engine()
```

### 4.4 KeywordTableIndex

Index ตาม keywords สำหรับ keyword-based retrieval

```python
from llama_index.core import KeywordTableIndex

index = KeywordTableIndex.from_documents(documents)
query_engine = index.as_query_engine()

response = query_engine.query("LlamaIndex embedding")
```

### 4.5 KnowledgeGraphIndex

สร้าง Knowledge Graph จาก documents (เชื่อมโยง entities)

```python
from llama_index.core import KnowledgeGraphIndex

index = KnowledgeGraphIndex.from_documents(
    documents,
    max_triplets_per_chunk=10,  # จำนวน triplets per chunk
    include_embeddings=True
)

query_engine = index.as_query_engine(
    include_text=True,
    retriever_mode="keyword",  # keyword, embedding, hybrid
    response_mode="tree_summarize"
)
```

---

## 5. Vector Stores

By default LlamaIndex ใช้ in-memory vector store แต่สำหรับ production ควรใช้ external vector store

### 5.1 Chroma (แนะนำสำหรับ development)

```python
pip install llama-index-vector-stores-chroma chromadb
```

```python
import chromadb
from llama_index.vector_stores.chroma import ChromaVectorStore
from llama_index.core import StorageContext, VectorStoreIndex

# สร้าง Chroma client
chroma_client = chromadb.PersistentClient(path="./chroma_db")
chroma_collection = chroma_client.get_or_create_collection("my_docs")

# สร้าง vector store
vector_store = ChromaVectorStore(chroma_collection=chroma_collection)
storage_context = StorageContext.from_defaults(vector_store=vector_store)

# สร้าง index (บันทึกลง Chroma)
index = VectorStoreIndex.from_documents(
    documents,
    storage_context=storage_context
)

# โหลด index ที่มีอยู่แล้ว (ไม่ต้อง embed ใหม่)
index = VectorStoreIndex.from_vector_store(
    vector_store=vector_store
)
```

### 5.2 Pinecone

```python
pip install llama-index-vector-stores-pinecone pinecone-client
```

```python
from pinecone import Pinecone, ServerlessSpec
from llama_index.vector_stores.pinecone import PineconeVectorStore

pc = Pinecone(api_key="YOUR_PINECONE_KEY")
pc.create_index(
    name="llamaindex-demo",
    dimension=1536,  # text-embedding-3-small dimension
    metric="cosine",
    spec=ServerlessSpec(cloud="aws", region="us-east-1")
)

pinecone_index = pc.Index("llamaindex-demo")
vector_store = PineconeVectorStore(pinecone_index=pinecone_index)
storage_context = StorageContext.from_defaults(vector_store=vector_store)

index = VectorStoreIndex.from_documents(
    documents,
    storage_context=storage_context
)
```

### 5.3 Weaviate

```python
pip install llama-index-vector-stores-weaviate weaviate-client
```

```python
import weaviate
from llama_index.vector_stores.weaviate import WeaviateVectorStore

client = weaviate.Client("http://localhost:8080")

vector_store = WeaviateVectorStore(
    weaviate_client=client,
    index_name="LlamaIndex"
)
storage_context = StorageContext.from_defaults(vector_store=vector_store)

index = VectorStoreIndex.from_documents(
    documents,
    storage_context=storage_context
)
```

### 5.4 Qdrant

```python
pip install llama-index-vector-stores-qdrant qdrant-client
```

```python
import qdrant_client
from llama_index.vector_stores.qdrant import QdrantVectorStore

client = qdrant_client.QdrantClient(
    host="localhost",
    port=6333
)

vector_store = QdrantVectorStore(
    client=client,
    collection_name="my_collection"
)
storage_context = StorageContext.from_defaults(vector_store=vector_store)

index = VectorStoreIndex.from_documents(
    documents,
    storage_context=storage_context
)
```

---

## 6. Persist & Load Index

### บันทึก Index ลง disk

```python
from llama_index.core import StorageContext

# บันทึก
index.storage_context.persist(persist_dir="./storage")

# โหลดกลับ
from llama_index.core import load_index_from_storage

storage_context = StorageContext.from_defaults(persist_dir="./storage")
index = load_index_from_storage(storage_context)
```

### StorageContext ประกอบด้วย

```python
storage_context = StorageContext.from_defaults(
    vector_store=vector_store,   # เก็บ embeddings
    docstore=docstore,           # เก็บ documents
    index_store=index_store,     # เก็บ index metadata
)
```

---

## 7. Workshop: PDF Q&A System

```python
import os
from llama_index.core import (
    VectorStoreIndex, 
    SimpleDirectoryReader,
    Settings,
    StorageContext
)
from llama_index.core.node_parser import SentenceSplitter
from llama_index.llms.openai import OpenAI
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.core.extractors import KeywordExtractor, TitleExtractor
from llama_index.core.ingestion import IngestionPipeline
import chromadb
from llama_index.vector_stores.chroma import ChromaVectorStore

# Setup
os.environ["OPENAI_API_KEY"] = "sk-..."
Settings.llm = OpenAI(model="gpt-4o-mini", temperature=0)
Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")

# 1. โหลด PDF
print("Loading PDFs...")
documents = SimpleDirectoryReader(
    "./pdfs",
    required_exts=[".pdf"]
).load_data()
print(f"Loaded {len(documents)} documents")

# 2. Setup Pipeline
pipeline = IngestionPipeline(
    transformations=[
        SentenceSplitter(chunk_size=1024, chunk_overlap=50),
        TitleExtractor(nodes=3),
        KeywordExtractor(keywords=5),
    ]
)

print("Processing nodes...")
nodes = pipeline.run(documents=documents)
print(f"Created {len(nodes)} nodes")

# 3. Setup ChromaDB
chroma_client = chromadb.PersistentClient(path="./chroma_db")
collection = chroma_client.get_or_create_collection("pdf_qa")
vector_store = ChromaVectorStore(chroma_collection=collection)
storage_context = StorageContext.from_defaults(vector_store=vector_store)

# 4. Create Index
print("Indexing...")
index = VectorStoreIndex(
    nodes,
    storage_context=storage_context
)

# 5. Query
query_engine = index.as_query_engine(similarity_top_k=5)

# ถามคำถาม
while True:
    question = input("\nถามคำถาม (หรือ 'quit' เพื่อออก): ")
    if question.lower() == "quit":
        break
    
    response = query_engine.query(question)
    print(f"\n📝 คำตอบ: {response}")
    print(f"\n📚 Sources ({len(response.source_nodes)} nodes):")
    for i, node in enumerate(response.source_nodes):
        print(f"  [{i+1}] Score: {node.score:.3f} | {node.metadata.get('file_name', 'unknown')}")
```

---

## สรุป Phase 2

| Component | ใช้ทำอะไร |
|---|---|
| SimpleDirectoryReader | โหลดไฟล์จาก folder |
| PDFReader, WebReader | โหลดจาก source เฉพาะ |
| SentenceSplitter | แบ่ง document เป็น chunks |
| SemanticSplitterNodeParser | แบ่งตาม semantic |
| IngestionPipeline | orchestrate การ transform |
| VectorStoreIndex | index หลักสำหรับ RAG |
| ChromaDB / Pinecone | persistent vector stores |
| StorageContext | จัดการ storage ทั้งหมด |

---

## ขั้นตอนถัดไป

👉 **Phase 3: Querying, Retrieval & RAG Pipeline** — เจาะลึก retrievers, response synthesizers, chat engines, และ advanced RAG techniques

---

*Phase 2/4 | LlamaIndex Deep Dive Series*
