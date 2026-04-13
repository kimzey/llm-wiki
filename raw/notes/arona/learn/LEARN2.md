# 🧪 เรียนรู้ Arona Ingest (ตอนที่ 2): เจาะลึกกระบวนการทำงาน (The Pipeline Walkthrough)

ในหน้านี้ เราจะมาดูโค้ดใน `src/modules/ingest/service.ts` ว่ามันทำงานอย่างไรทีละขั้นตอน เพื่อให้คุณเห็น "ทางเดิน" ของข้อมูลครับ

---

## 1. จุดเริ่มต้น: ฟังก์ชัน `ingest(req)`
เมื่อมี Request เข้ามาทาง API ฟังก์ชันนี้จะถูกเรียกใช้ โดยรับค่าพารามิเตอร์สำคัญคือ:
*   `source_name`: ชื่อเอกสาร (เช่น "ระเบียบการลา")
*   `source_type`: ประเภทไฟล์ (เช่น "markdown")
*   `paths`: ที่อยู่ของไฟล์ในเครื่อง (เช่น `data/test/hr-policy.md`)

---

## 2. ขั้นตอนที่ 1: การดึงข้อมูล (Extract)
```typescript
// ใน service.ts
if (req.source_type === 'markdown') {
    const docs = await extractMarkdown({ paths: req.paths })
    rawContent = docs.map((d) => d.content).join('\n\n')
    sourceContentHash = docs.map((d) => d.contentHash).join(':')
}
```
*   ระบบจะไปอ่านไฟล์ และนำเนื้อหาทั้งหมดมาต่อกันเป็น `rawContent` (String ยาวๆ)
*   มีการสร้าง `sourceContentHash` (รหัสลายนิ้วมือของไฟล์) เพื่อดูว่าไฟล์เปลี่ยนไปไหม

---

## 3. ขั้นตอนที่ 2: การตัดแบ่ง (Chunking)
```typescript
const newChunks = chunk(rawContent, req.source_name, chunkConfigs)
```
*   เราส่งเนื้อหายาวๆ เข้าไปให้ `chunk()` ตัดแบ่ง
*   **ทำไมต้องตัด?** เพราะ AI รับข้อมูลได้จำกัด และการค้นหาข้อมูลใน "ชิ้นเล็กๆ" ที่ตรงประเด็น จะให้ผลลัพธ์ที่ดีกว่าการเอาไฟล์ใหญ่ๆ ไปให้ AI อ่าน

---

## 4. ขั้นตอนที่ 3: การวางแผน (Planning)
```typescript
const plan = planIngestion(newChunks, existingChunks, sourceContentHash, existingSourceHash)
```
*   ขั้นตอนนี้จะเช็คว่าใน Database มี Chunk เหล่านี้อยู่แล้วหรือยัง? 
*   **`plan` จะบอกเราว่า:** อันไหนต้องเพิ่มใหม่ (Create), อันไหนต้องแก้ไข (Update), อันไหนลบออก (Delete) และอันไหนที่ใช้ของเดิมได้เลย (Reuse)
*   *ถ้าไฟล์เหมือนเดิมทุกประการ* `plan` จะบอกให้ `skip` (จบการทำงานทันที)

---

## 5. ขั้นตอนที่ 4: การสร้างพิกัดเวกเตอร์ (Embedding)
```typescript
const { embeddings } = await embedMany({
    model: openai.embeddingModel('text-embedding-3-small'),
    values: [
        ...toProcess.map(p => p.chunk.content), // เนื้อหา
        ...toProcess.map(p => p.chunk.title),   // หัวข้อ
        ...toProcess.map(_ => req.source_name)  // ชื่อไฟล์
    ]
})
```
*   เราใช้โมเดลของ OpenAI (`text-embedding-3-small`) เปลี่ยน "ข้อความ" ให้เป็น "ตัวเลขชุดยาวๆ" (Vector)
*   **ความเจ๋ง:** เราสร้าง 3 เวกเตอร์ต่อ 1 ชิ้นงาน เพื่อให้เวลาค้นหา ค้นได้ทั้งจาก *เนื้อหา*, *หัวข้อ* และ *ชื่อไฟล์*

---

## 6. ขั้นตอนที่ 5: บันทึกลงฐานข้อมูล (Database)
```typescript
await sql`INSERT INTO chunks (...) VALUES (...)`
```
*   บันทึกทั้งข้อความและตัวเลข (Vector) ลงตาราง `chunks`
*   ใช้คำสั่ง `ON CONFLICT` เพื่อให้ถ้ามีข้อมูลเดิมอยู่แล้ว ให้ทำการ Update แทนการ Insert ซ้ำ

---

**สรุป:** ข้อมูลเดินทางจาก **ไฟล์บนดิสก์** ➡️ **การตัดแบ่ง** ➡️ **การเปรียบเทียบ** ➡️ **การแปลงเป็นเวกเตอร์** ➡️ **การจัดเก็บ**

**ไปต่อกันที่ [LEARN3.MD](./LEARN3.MD) เพื่อดูตรรกะเบื้องหลังการประหยัดค่าใช้จ่าย!**
