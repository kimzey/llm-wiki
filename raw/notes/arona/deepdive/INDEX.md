# Arona Deep Dive Learning Series (ชุดบทเรียนเจาะลึก Arona RAG)

ยินดีต้อนรับสู่ชุดบทเรียนเจาะลึกสถาปัตยกรรมและซอร์สโค้ดของระบบ **Arona RAG** บทเรียนเหล่านี้ถูกออกแบบมาเพื่อให้คุณเข้าใจระบบอย่างถ่องแท้ ตั้งแต่บรรทัดแรกของโค้ดไปจนถึงการทำงานของ AI และการค้นหาในฐานข้อมูล

## สารบัญบทเรียน (Course Syllabus)

### 📘 [Lesson 1: เจาะลึกสถาปัตยกรรมและวงจรการทำงาน (Architecture & Lifecycle)](./01-architecture-and-lifecycle.md)
- ปรัชญาการออกแบบและ Technical Stack (Bun, Elysia, ParadeDB)
- วิเคราะห์ `src/index.ts` และ `src/server.ts` ทีละบรรทัด
- วงจรการเดินทางของคำถาม (Query Trace) ตั้งแต่ต้นจนจบ
- ระบบ Clustering และการขยายตัวของระบบ

### 📗 [Lesson 2: ระบบการนำข้อมูลเข้า (The Ingestion Pipeline)](./02-ingestion-pipeline.md)
- การดึงเนื้อหาจาก Markdown และไฟล์ประเภทต่างๆ
- ศิลปะการหั่นข้อมูล (Smart Chunking) ตามหัวข้อ
- ระบบวางแผนนำเข้า (Ingestion Planning) เพื่อประหยัด Embedding
- การเตรียมข้อมูลก่อนทำ Vectorization (Data Cleaning)

### 📙 [Lesson 3: ระบบการค้นหาและปัญญาประดิษฐ์ (Retrieval & AI Engine)](./03-retrieval-and-ai.md)
- การค้นหาแบบ Hybrid Search (BM25 + Vector) ใน SQL เดียว
- การทำงานของ AI Tools และระบบ Agentic Loop
- การจัดการ Context และประวัติการสนทนา (Conversation History)
- ระบบ Cache 3 ชั้นเพื่อความเร็วสูงสุด

### 📕 [Lesson 4: ความปลอดภัย การตั้งค่า และการดูแลรักษา (Security & Maintenance)](./04-security-and-maintenance.md)
- ระบบตรวจสอบสิทธิ์ (API Key Auth) และสิทธิ์ระดับแผนก
- การจัดการคอนฟิกใน `.env` และระบบ Monitoring (Axiom)
- การดูแลรักษาฐานความรู้และการ Re-indexing
- การติดตั้งด้วย Docker และแนวทางการพัฒนาต่อในอนาคต

---

## วิธีการเรียนรู้ที่แนะนำ
1. **อ่านคู่ไปกับ Code:** เปิดไฟล์ที่ระบุในบทเรียนใน Code Editor ของคุณเพื่อดูตัวอย่างจริง
2. **ลองแก้ไข:** ในแต่ละบทเรียนจะมีคำแนะนำในการพัฒนาต่อ ลองนำไปทำจริงเพื่อเสริมสร้างความเข้าใจ
3. **รัน Tests:** ใช้คำสั่ง `bun test` เพื่อดูว่าการทำงานของฟังก์ชันต่างๆ สอดคล้องกับที่อธิบายในบทเรียนไหม

ขอให้สนุกกับการเดินทางเข้าสู่โลกของ RAG และ AI Engineer ครับ!
