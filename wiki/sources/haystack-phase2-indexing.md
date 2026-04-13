---
title: "Haystack Deep Dive — Phase 2: Document Processing & Indexing Pipeline"
type: source
source_file: raw/notes/Hetstack/haystack-phase2-document-processing.md
url: ""
published: 2026-04-01
tags: [haystack, document-processing, indexing, chunking, embedding, splitter]
related: [wiki/sources/haystack-phase1-overview.md, wiki/sources/haystack-phase3-retrieval.md, wiki/concepts/haystack-framework.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/Hetstack/haystack-phase2-document-processing.md|Original file]]
> *ดูเพิ่ม: [[../../raw/notes/Hetstack/haystack-phase2-indexing.md|Indexing alt file]]*

## สรุป

อธิบาย Indexing Pipeline ใน Haystack — การแปลงไฟล์ทุกประเภทเป็น Document, ทำความสะอาด, ตัดเป็น chunks, สร้าง embedding, และเขียนเข้า DocumentStore พร้อมตัวอย่าง complete pipeline สำหรับ BM25 และ Semantic Search

## ประเด็นสำคัญ

- **File Converters**: PyPDFToDocument, TextFileToDocument, HTMLToDocument, DOCXToDocument, JSONConverter, CSVToDocument — รองรับไฟล์หลากหลาย
- **FileTypeRouter + DocumentJoiner**: route ไฟล์ตาม MIME type แล้ว join กลับ — pattern สำหรับ multi-format indexing
- **DocumentCleaner**: ลบ empty lines, extra whitespace, repeated substrings (header/footer)
- **DocumentSplitter** — หัวใจ RAG: ตัดด้วย `word`/`sentence`/`passage`/`page`/`function` + overlap ป้องกันข้อมูลหายตรงรอยต่อ
- **DocumentWriter policy**: `overwrite`/`skip`/`fail` — สำหรับ idempotent indexing ใช้ `skip`
- **Metadata** ช่วยใน filtering ตอน retrieval — inject เพิ่มด้วย `meta={}` ตอน run

## ข้อมูล / หลักฐาน ที่น่าสนใจ

**Chunk Size Guidelines:**
```
เอกสารทั่วไป:     split_length=200-300 words, overlap=20-30
FAQ/Q&A:          split_length=100-150 words
Legal/Technical:  split_length=300-500 words
Code/Technical:   split_by="passage"
```

**Split Overlap ทำงานอย่างไร:**
```
ไม่มี overlap: [1-200] [201-400] ← ข้อมูลหายถ้าคำตอบอยู่ตรงรอยต่อ!
มี overlap=20: [1-200] [181-380] [361-560] ← overlap 20 คำกันหาย
```

**Embedding Models สำหรับภาษาไทย:**
- `BAAI/bge-m3` — ดีที่สุดสำหรับ multilingual รวมไทย
- `intfloat/multilingual-e5-large` — แม่นยำ แต่ช้า/หนัก
- `sentence-transformers/paraphrase-multilingual-mpnet-base-v2` — เบา เร็ว

**Indexing Pipeline flow (embedding-based):**
```
Converter → Cleaner → Splitter → Embedder → Writer → DocumentStore
```

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่มีข้อขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/haystack-framework.md|Haystack Framework]]
- [[wiki/concepts/rag-retrieval-augmented-generation.md|RAG — Retrieval-Augmented Generation]]
