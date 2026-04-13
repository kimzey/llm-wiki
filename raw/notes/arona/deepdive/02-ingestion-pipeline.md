# Lesson 2: ระบบการนำข้อมูลเข้า (The Ingestion Pipeline Deep Dive)

ในโลกของ AI และ RAG มีกฎเหล็กข้อหนึ่งที่ทุกคนต้องจำไว้คือ **"Garbage In, Garbage Out"** หากเราเตรียมข้อมูลไม่ดี หั่นข้อมูลไม่ฉลาด หรือดึงใจความสำคัญออกมาไม่ได้ AI ของเราก็จะตอบคำถามได้แย่ตามไปด้วย ในบทเรียนนี้เราจะมาเจาะลึกว่า Arona เปลี่ยนไฟล์เอกสารธรรมดาให้กลายเป็น "ความรู้" ของ AI ได้อย่างไร โดยเน้นการวิเคราะห์โค้ดและกลยุทธ์เบื้องหลังอย่างละเอียด

---

## 1. ปรัชญาการนำเข้าข้อมูล (Ingestion Philosophy)

การนำข้อมูลเข้าใน Arona ไม่ใช่แค่การ Copy & Paste ข้อความลงฐานข้อมูล แต่คือกระบวนการที่มีการวางแผนมาอย่างดีเพื่อให้ระบบมีความฉลาดและประหยัดทรัพยากรที่สุด:

1.  **Efficiency (ความมีประสิทธิภาพ):** ต้องประหยัดทรัพยากร ไม่ทำ Embedding ซ้ำถ้าข้อมูลไม่เปลี่ยน การเรียก AI API มีค่าใช้จ่ายสูง เราจึงต้องมีระบบ Checksum เพื่อลดการทำงานที่ไม่จำเป็น
2.  **Context Preservation (การรักษาบริบท):** เมื่อหั่นข้อมูลเป็นชิ้นเล็กๆ (Chunks) เราต้องมั่นใจว่าแต่ละชิ้นยังมีใจความที่สมบูรณ์ ไม่ถูกตัดขาดกลางประโยค ซึ่งจะทำให้ AI สับสน
3.  **Accuracy (ความแม่นยำ):** กระบวนการดึงข้อมูล (Extraction) ต้องแม่นยำ 100% รวมถึงการจัดการกับอักขระพิเศษและตารางข้อมูล

---

## 2. ขั้นตอนที่ 1: การดึงเนื้อหา (Text Extraction)

เราเริ่มจากการเปลี่ยนไฟล์ (.md, .pdf) ให้กลายเป็นโครงสร้างข้อมูลที่เราจัดการได้ใน `src/libs/extractors/`

### 2.1 ทำไมต้องเน้น Markdown?
Markdown คือหัวใจของ Arona เพราะ:
- **Semantic Structure:** มีโครงสร้างชัดเจนผ่านทาง Headers (#, ##, ###) ซึ่งบอกลำดับความสำคัญของข้อมูลได้ดีกว่าไฟล์ Word หรือ PDF
- **AI-Friendly:** โมเดล LLM ส่วนใหญ่ถูกฝึกมากับข้อมูล Markdown จำนวนมาก ทำให้มันเข้าใจสัญลักษณ์ต่างๆ ได้อย่างเป็นธรรมชาติ
- **Lightweight:** ไฟล์มีขนาดเล็กและประมวลผลได้รวดเร็วมาก

### 2.2 เจาะลึก Markdown Extractor (`src/libs/extractors/markdown.ts`)
```typescript
import { cyrb53 } from '../modules/ai/libs/utils'

export async function extractMarkdown({ paths }: ExtractOptions): Promise<RawDocument[]> {
  const docs: RawDocument[] = []
  
  for (const path of paths) {
    // ใช้ Bun.file() ซึ่งเป็น Native API ที่เร็วที่สุดในการอ่านไฟล์
    const file = Bun.file(path)
    const exists = await file.exists()
    
    if (!exists) {
      console.warn(`File not found: ${path}`)
      continue
    }

    const content = await file.text()
    
    // สร้าง Hash (Checksum) เพื่อตรวจสอบการเปลี่ยนแปลงในภายหลัง
    // เราใช้ cyrb53 เพราะเร็วและมีโอกาสชนกัน (Collision) ต่ำมากสำหรับข้อความทั่วไป
    const contentHash = cyrb53(content).toString()
    
    docs.push({
      content,
      contentHash,
      metadata: { 
        path, 
        source: 'local_markdown',
        lastModified: new Date().toISOString()
      }
    })
  }
  
  return docs
}
```

---

## 3. ขั้นตอนที่ 2: ศิลปะการหั่นข้อมูล (Smart Chunking)

นี่คือส่วนที่สำคัญที่สุดของระบบ RAG อยู่ใน `src/libs/chunker.ts` หากหั่นผิด AI จะตอบผิดทันที

### 3.1 ยุทธศาสตร์การหั่นตามหัวข้อ (Heading-Based Strategy)
ใน Arona เราเชื่อว่า "หัวข้อ" คือตัวแบ่งบริบทที่ดีที่สุด:
- ระบบจะใช้ Regex มองหาเครื่องหมาย `##` หรือ `###`
- เมื่อเจอหัวข้อใหม่ ระบบจะถือว่าเป็น "เรื่องใหม่" และเริ่ม Chunk ใหม่ทันที
- **ตัวอย่าง:** ถ้ามีหัวข้อ "ระเบียบการลาป่วย" และต่อด้วย "ระเบียบการลาพักร้อน" ระบบจะแยกสองเรื่องนี้ออกจากกันชัดเจน

### 3.2 การจัดการกับ Chunk ที่ใหญ่เกินไป
บางครั้งเนื้อหาภายใต้หัวข้อเดียวอาจยาวหลายหน้า เราจึงต้องมีระบบ **Sub-splitting**:
- เรากำหนด `maxChunkSize` (ปกติคือ 1,500 - 2,000 ตัวอักษร)
- หากยาวเกิน ระบบจะหั่นย่อยโดยมองหาจุดจบประโยค (Sentence Boundary) เพื่อไม่ให้เนื้อหาขาดกลางคัน

---

## 4. ขั้นตอนที่ 3: การวางแผนนำเข้า (The Ingestion Plan)

หัวใจของความฉลาดใน Arona อยู่ที่ `src/modules/ingest/service.ts` ซึ่งทำหน้าที่เป็น "ตัวคัดกรอง"

### 4.1 ปัญหาของระบบ RAG ทั่วไป
ระบบส่วนใหญ่เมื่อไฟล์เปลี่ยน จะใช้วิธี "ลบแล้วสร้างใหม่":
- **Costly:** ต้องเสียเงินเรียก Embedding API ใหม่ทั้งหมดแม้จะแก้แค่ตัวอักษรเดียว
- **Slow:** การเขียนฐานข้อมูลใหม่ทั้งหมดทำให้ระบบค้าง

---

## 5. เจาะลึกโค้ดระบบ Ingest แบบบรรทัดต่อบรรทัด (`src/modules/ingest/service.ts`)

เพื่อให้คุณเข้าใจกระบวนการทั้งหมด เรามาดูโค้ดจริงที่รันระบบนี้กันครับ:

```typescript
import { sql, log } from '@arona/libs'
import { embedMany } from 'ai'
import { openai } from '@arona/libs/ai'
import { extractMarkdown } from '@arona/libs/extractors/markdown'
import { chunk, chunkContentHash, type ChunkResult } from '@arona/libs/chunker'

// 1. ฟังก์ชันวางแผนการนำเข้า (Ingestion Planning)
// ฟังก์ชันนี้คือ "สมอง" ที่ตัดสินใจว่าอะไรควรสร้างใหม่ หรืออะไรควรลบ
export function planIngestion(
	newChunks: ChunkResult[],
	existingChunks: Array<{ id: string; content_hash: string; sequence: number }>,
	sourceContentHash: string,
	existingSourceHash: string | null
): IngestionPlan {
	// ถ้าเป็นเอกสารใหม่แกะกล่อง (ไม่มีใน DB)
	if (existingSourceHash === null) {
		return {
			sourceAction: 'create',
			chunksToCreate: [...newChunks],
			chunksToUpdate: [],
			chunksToReuse: [],
			chunkIdsToDelete: []
		}
	}

	// ถ้า Hash รวมของไฟล์เหมือนเดิมเป๊ะ ไม่ต้องทำอะไรเลย (ประหยัดทรัพยากรสุดๆ)
	if (sourceContentHash === existingSourceHash) {
		return { sourceAction: 'skip', ... }
	}

	// ... Logic การวนลูปเปรียบเทียบ Hash ราย Chunk เพื่อแยกกลุ่ม Create, Update, Reuse
}

// 2. ฟังก์ชันหลักในการรัน Pipeline
export async function ingest(req: IngestRequest): Promise<IngestResult> {
	const start = Date.now()
	log(`Starting ingestion for: ${req.source_name}`)

	// ขั้นตอนที่ 1: ดึงข้อความ (Extract)
	// ระบบรองรับทั้ง Markdown และ Text ธรรมดา
	let rawContent = ''
	if (req.source_type === 'markdown') {
		const docs = await extractMarkdown({ paths: req.paths })
		rawContent = docs.map((d) => d.content).join('\n\n')
	}

	// ขั้นตอนที่ 2: หั่นข้อมูล (Chunk)
	// เรียกใช้ Chunker ที่เราอธิบายไว้ในหัวข้อที่ 3
	const newChunks = chunk(rawContent, req.source_name, { strategy: 'heading' })

	// ขั้นตอนที่ 3: วางแผน (Plan)
	// เช็คกับ Database ว่ามีของเก่าอยู่ไหม
	const existingSource = await sql`SELECT id, content_hash FROM sources WHERE source_name = ${req.source_name}`.then(x => x[0])
	const plan = planIngestion(newChunks, existingChunks, sourceContentHash, existingSource?.content_hash)

	// ขั้นตอนที่ 4: ทำ Embedding (Vectorization)
	// เราใช้ embedMany เพื่อส่งข้อมูลไป OpenAI ทีเดียวเป็นชุด (Batch) เพื่อความรวดเร็ว
	if (toProcess.length > 0) {
		const { embeddings } = await embedMany({
			model: openai.embeddingModel('text-embedding-3-small'),
			values: [
				...toProcess.map(p => p.chunk.content), // เนื้อหาหลัก
				...toProcess.map(p => p.chunk.title),   // หัวข้อ (เพื่อความแม่นยำในการ Search)
				...toProcess.map(_ => req.source_name) // ชื่อไฟล์
			]
		})

		// ขั้นตอนที่ 5: บันทึกลงฐานข้อมูล (Save to ParadeDB)
		// เราใช้ระบบ Transaction หรือการวนลูป Insert/Update ตามที่วางแผนไว้
		// สังเกตการใช้ ::vector ใน SQL เพื่อบอก Postgres ว่านี่คือข้อมูล Vector
		for (let i = 0; i < chunkSize; i++) {
			await sql`INSERT INTO chunks (...) VALUES (..., ${`[${embedding.join(',')}]`}::vector)`
		}
	}

	log(`Ingestion completed in ${Date.now() - start}ms`)
}
```

---

## 6. วิเคราะห์ส่วนประกอบทางเทคนิคที่สำคัญ

### 6.1 ระบบ Checksum ราย Chunk
สังเกตในโค้ดมีการใช้ `chunkContentHash`. นี่คือสิ่งที่ทำให้ Arona แตกต่าง:
- เราไม่ได้แค่เช็คว่า "ไฟล์เปลี่ยนไหม"
- แต่เราเช็คว่า "ย่อหน้าไหนเปลี่ยนบ้าง"
- ถ้าคุณแก้แค่หน้า 10 จาก 100 หน้า ระบบจะจ่ายเงินทำ Embedding ใหม่แค่หน้าเดียว!

### 6.2 การใช้ Vector ใน SQL
ใน ParadeDB เราเก็บข้อมูล Vector เป็นอาร์เรย์ของตัวเลขทศนิยม 1,536 ตัว:
- เราต้องใส่ `::vector` เพื่อให้ Postgres เรียกใช้ Extension `pgvector` ในการประมวลผล
- ข้อมูลนี้จะถูกนำไปสร้าง Index แบบ HNSW เพื่อให้ค้นหาได้รวดเร็วในบทถัดไป

---

## 7. สรุปบทเรียนที่ 2

การเตรียมความรู้คือ 80% ของความสำเร็จในระบบ RAG:
- **Extract** ให้แม่นยำด้วย Markdown
- **Chunk** ให้ฉลาดตามหัวข้อเอกสาร
- **Plan** ให้ประหยัดด้วยระบบ Diffing
- **Embed** ให้คมชัดด้วยการส่งข้อมูลเป็นชุด (Batch)

ในบทถัดไป เราจะมาดูส่วนที่ตื่นเต้นที่สุด คือการนำความรู้เหล่านี้มาตอบคำถามใน **"Lesson 3: ระบบการค้นหาและปัญญาประดิษฐ์ (Retrieval & AI Engine)"** ครับ!

---

### ภาคผนวก: ตัวอย่างโครงสร้าง JSON ที่ใช้ในระบบ Ingest

เพื่อให้คุณเห็นภาพว่าข้อมูลที่ส่งเข้า API หน้าตาเป็นอย่างไร:

```json
{
  "source_name": "hr-policy-2026",
  "source_type": "markdown",
  "paths": ["./data/hr/policy.md"],
  "category": "HR",
  "department": "all",
  "access_level": "public"
}
```

เมื่อคุณส่ง JSON นี้ไปที่ `POST /ingest` ระบบจะเริ่มรัน Pipeline ทั้งหมดที่เราคุยกันมาในบทนี้ทันทีครับ!
