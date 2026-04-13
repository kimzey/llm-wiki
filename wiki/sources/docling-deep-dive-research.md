---
title: "Docling Deep Dive — Granite-Docling & PDF Parsing Benchmarks"
type: source
source_file: raw/notes/docling-research-2026.md
url: "https://github.com/docling-project/docling"
published: 2026-04-14
tags: [docling, ibm, document-parser, ocr, pdf, granite-docling, llm-ingestion]
related: [wiki/concepts/docling-document-parser, wiki/concepts/openrag-platform, wiki/concepts/rag-chunking-strategies]
created: 2026-04-14
updated: 2026-04-14
---

> **Full source**: [[../../raw/notes/docling-research-2026.md|Research Notes]]
>
> Sources: [Docling GitHub](https://github.com/docling-project/docling) · [Docling Docs](https://docling-project.github.io/docling/) · [Docling Paper (arXiv)](https://arxiv.org/html/2501.17887v1) · [Granite-Docling IBM](https://www.ibm.com/new/announcements/granite-docling-end-to-end-document-conversion) · [PDF Extraction Benchmark](https://procycons.com/en/blogs/pdf-data-extraction-benchmark/)

## สรุป

Docling คือ open-source document parser จาก IBM Research ที่ใช้ AI model เฉพาะทาง (DocLayNet + TableFormer) แปลง PDF/DOCX/PPTX ที่ซับซ้อนให้เป็น structured Markdown — เหมาะกับ RAG ingestion pipeline ที่ต้องการคุณภาพสูง

## ประเด็นสำคัญ

### Format Support ครบครัน

รองรับ: PDF, DOCX, PPTX, XLSX, HTML, WAV, MP3, WebVTT, PNG/TIFF/JPEG, LaTeX, plain text

### Granite-Docling-258M (March 2025)

- รุ่นใหม่ล่าสุด: single model รวม vision + language ขนาด ~258M parameters
- ใช้ **DocTags** markup language: แยก text content จาก document structure ออกจากกัน
- DocTags ระบุตำแหน่งแต่ละ element บนหน้า, reading order, hierarchy
- Output: DocTags → Markdown / JSON / HTML

### AI Models ที่ใช้ภายใน

| Model | หน้าที่ |
|-------|---------|
| **DocLayNet** | Layout analysis: regions, headings, figures, tables, reading order |
| **TableFormer** | Table structure recognition: merge cells, row/col headers |
| **EasyOCR / Tesseract** | OCR สำหรับ scanned PDFs และ images ฝังในหน้า |

### Benchmark vs Competitors (2025)

| Tool | Strengths |
|------|-----------|
| **Docling** | ดีที่สุดสำหรับ structured PDFs (tables, multi-column layouts) |
| **LlamaParse** | ดีกว่าสำหรับ scanned docs คุณภาพต่ำ |
| **Unstructured.io** | Format support หลากหลายที่สุด |

### Integration Ecosystem

```python
# LangChain
from langchain_docling import DoclingLoader
loader = DoclingLoader(file_path="document.pdf")
docs = loader.load()

# LlamaIndex  
from llama_index.readers.docling import DoclingReader
reader = DoclingReader()
docs = reader.load_data("document.pdf")
```

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- Granite-Docling เป็น product-ready evolution ของ SmolDocling-256M-preview (released March 2025)
- IBM Research Zurich เป็นทีมสร้าง — เป็น DS4SD (Document Store for Scientific Documents) project
- Docling ใช้งานอยู่ใน OpenRAG Platform เป็น default document parser ใน ingestion pipeline

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/docling-document-parser|Docling Document Parser]]
- [[wiki/concepts/openrag-platform|OpenRAG Platform]]
- [[wiki/concepts/rag-chunking-strategies|RAG Chunking Strategies]]
