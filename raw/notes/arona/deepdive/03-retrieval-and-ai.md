# Lesson 3: ระบบการค้นหาและปัญญาประดิษฐ์ (Retrieval & AI Engine)

ยินดีต้อนรับสู่บทที่ 3 ซึ่งเป็น "หัวใจและสมอง" ของ Arona ในบทนี้เราจะมาเจาะลึกว่าระบบหาคำตอบที่ถูกต้องเจอได้อย่างไรท่ามกลางข้อมูลมหาศาล และ AI ใช้กระบวนการคิดอย่างไรในการสรุปคำตอบให้พนักงาน

---

## 1. การค้นหาแบบ Hybrid Search: ทำไมต้องสองอย่าง?

ปัญหาที่ใหญ่ที่สุดของ RAG คือ "หาข้อมูลไม่เจอ" หรือ "หามาผิดอัน" Arona จึงใช้ระบบ **Hybrid Search** ซึ่งเป็นการรวมพลังของ:

1.  **BM25 (Keyword Search):** เก่งเรื่องการหาคำเฉพาะเป๊ะๆ เช่น "ESS", "HR-001", "รหัสพนักงาน"
2.  **Vector Search (Semantic Search):** เก่งเรื่องการหา "ความหมาย" เช่น เมื่อ User ถามว่า "ไม่สบายต้องทำไง" ระบบจะหาเอกสารที่เกี่ยวกับ "การลาป่วย" เจอ แม้ในเอกสารจะไม่มีคำว่า "ไม่สบาย" เลยก็ตาม

---

## 2. เจาะลึก SQL Hybrid Search (`src/modules/ai/service.ts`)

เรามาดูโค้ดจริงๆ ที่ทำหน้าที่ค้นหาข้อมูลครับ ฟังก์ชัน `search` คือส่วนที่รวบรวมข้อมูลมาให้ AI:

```typescript
export async function search(
	value: string,
	userContext: UserContext,
	abortSignal?: AbortSignal
) {
	// 1. ทำความสะอาดข้อความค้นหา
	value = value.toLowerCase().trim()
	if (!value) return []

	const departments = userContext.departments

	// 2. ค้นหาด้วย BM25 (Full-text search)
	// เราใช้ SQL.findByBM25 ซึ่งถูกปรับแต่งมาให้ค้นหาใน Title และ Content พร้อมกัน
	const references = await sql.unsafe<Reference[]>(
		SQL.findByBM25, 
		[value.replace(/\(\)/g, '\\$1'), `{${departments.join(',')}}`]
	).then((x) => [...x])

	if (abortSignal?.aborted) return references

	// 3. ค้นหาด้วย Vector (Semantic search)
	// ถ้า Keyword Search ได้ผลไม่พอ เราจะใช้ Vector เข้ามาช่วยเสริม
	const vectorResult = await sql.unsafe<Reference[]>(
		SQL.findByVector, 
		[`[${await getEmbedding(value).then((x) => x.join(','))}]`, 
		 6 - references.length, 
		 `{${departments.join(',')}}`]
	).then((x) => [...x])

	// 4. รวมผลลัพธ์และลบตัวซ้ำ (Deduplication)
	for (const ref of vectorResult) {
		if (!references.find((r) => r.source_name === ref.source_name))
			references.push(ref)
	}

	return references
}
```

---

## 3. วงจรความคิดของ AI (The AI Reasoning Loop)

ฟังก์ชัน `ask` คือจุดที่ "ความรู้" และ "คำถาม" มาเจอกัน โดยใช้ **Vercel AI SDK**:

```typescript
export function ask({ message, history, references, userContext, think }: AskParams) {
	// 1. สร้างเครื่องมือ (Tools) ให้ AI
	// AI สามารถตัดสินใจเรียก Search เพิ่มเติม หรืออ่านเอกสารทั้งฉบับได้เอง
	const searchTool = createSearchTool(references, userContext)
	const readDocTool = createDocumentTool(references, userContext)

	// 2. เรียกใช้ AI Model (streamText)
	return record('Gather Resources', () =>
		streamText({
			model, // Model จาก OpenRouter (เช่น GPT-4o)
			tools: {
				readDocument: readDocTool,
				search: searchTool,
				listDocuments: listDocumentsTool
			},
			// จำกัดจำนวนรอบที่ AI จะคิดวนลูป (Prevent Infinite Loops)
			stopWhen: [
				stepCountIs(think ? 12 : 8),
				() => references.length > 32
			],
			system: instruction, // System Prompt หลักที่สั่งให้ AI สุภาพและแม่นยำ
			messages: [
				{ role: 'user', content: message },
				// ... ส่งประวัติการคุยและข้อมูลอ้างอิงเบื้องต้นไปให้ AI
			],
			// ระบบ Streaming: ส่งคำตอบกลับหา User ทันทีที่คำแรกถูกสร้าง
		})
	).textStream
}
```

---

## 4. ระบบการจำ 3 ชั้น (Triple-Tier Caching)

เพื่อให้ระบบตอบสนองได้รวดเร็ว (Latency ต่ำ) และประหยัดค่าใช้จ่าย เรามีระบบ Cache ที่ซับซ้อน:

1.  **Tier 1: Redis Exact Match**
    - ใช้คีย์จาก `cyrb53(message + history)`
    - ถ้าถามเหมือนเดิมเป๊ะ ตอบกลับในหลักมิลลิวินาที
2.  **Tier 2: Semantic Cache**
    - ใช้ Vector ค้นหาคำถามที่ "ความหมายเหมือนกัน"
    - แม้ User จะพิมพ์ผิดหรือใช้คำต่างกัน ระบบก็หาคำตอบเดิมเจอ
3.  **Tier 3: Response Cache (In-Memory)**
    - อยู่ใน `AI.ResponseCache` (ใช้ `BurstCache` class)
    - เก็บคำตอบล่าสุดไว้ในแรมของ Server เป็นเวลา 30 วินาที เพื่อป้องกันการยิงคำถามเดิมรัวๆ จากหน้าเว็บ

---

## 5. การรักษาความปลอดภัยระดับแถว (Row-Level Security)

สังเกตในฟังก์ชัน `readDocument`:
```typescript
export async function readDocument(sourceName: string, userContext: UserContext) {
	return sql.unsafe<Reference[]>(
		`SELECT ... 
		 WHERE s.source_name = $1
		 AND (c.access_level = 'public' OR c.department = ANY($2::text[]) OR c.department = 'all')`,
		[sourceName, `{${departments.join(',')}}`]
	)
}
```
**หัวใจสำคัญ:** AI จะไม่มีทางเข้าถึงเอกสารที่ User คนนั้นไม่มีสิทธิ์เห็น เพราะเราบังคับเช็ค `department` ในระดับ SQL ทุกครั้ง ไม่ว่า AI จะพยายามเรียกเครื่องมือ `readDocument` กี่รอบก็ตาม

---

## 6. สรุปบทเรียนที่ 3

สมองของ Arona ทำงานด้วยการประสานงานของ:
- **Retrieval:** หาข้อมูลให้เจอด้วย Hybrid Search (BM25 + Vector)
- **Reasoning:** คิดวิเคราะห์ด้วย LLM พร้อมความสามารถในการใช้เครื่องมือ (Tool Calling)
- **Security:** บังคับใช้สิทธิ์เข้าถึงข้อมูลในทุกระดับการค้นหา
- **Performance:** ลดภาระ AI ด้วยระบบ Cache อัจฉริยะ 3 ชั้น

ในบทสุดท้าย เราจะมาดูเรื่องการดูแลระบบและการทำให้ Arona ปลอดภัยใน **"Lesson 4: ความปลอดภัย การตั้งค่า และการดูแลรักษา"** ครับ!

---

### ภาคผนวก: เทคนิคการบีบอัดประวัติ (History Compression)

เพื่อให้ AI ไม่สับสนเมื่อคุยกันยาวๆ Arona มีฟังก์ชัน `compressHistory`:
- มันจะเก็บข้อความล่าสุดไว้ครบถ้วน
- ข้อความเก่าๆ จะถูกตัดทอนให้เหลือแค่ใจความสำคัญ
- ช่วยให้ AI ยังจำบริบทได้โดยไม่ทำให้ Context Window เต็มและไม่เปลือง Token

ความฉลาดของ Arona ไม่ได้อยู่ที่ Model เพียงอย่างเดียว แต่อยู่ที่ "การเตรียมบริบท" ที่ดีที่สุดให้ Model นั่นเองครับ!
