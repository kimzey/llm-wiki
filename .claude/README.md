# .claude — คู่มือการใช้งาน

โฟลเดอร์นี้คือ "สมองของ Claude Code" สำหรับ vault นี้
ประกอบด้วย commands (คำสั่ง slash), skills (พฤติกรรมอัตโนมัติ), และ settings

---

## โครงสร้างไฟล์

```
.claude/
├── README.md              ← ไฟล์นี้
├── settings.json          ← ค่าตั้งต้นของโปรเจ็ค
├── commands/              ← slash commands (/ingest, /query, ฯลฯ)
│   ├── ingest.md
│   ├── query.md
│   ├── lint.md
│   ├── status.md
│   ├── new-concept.md
│   ├── new-book.md
│   ├── synthesis.md
│   ├── note.md
│   └── update.md
└── skills/                ← พฤติกรรมที่ activate อัตโนมัติตาม context
    ├── wiki-ingest/
    ├── wiki-query/
    ├── wiki-lint/
    └── wiki-status/
```

---

## Commands คืออะไร?

Commands คือ **คำสั่งที่พิมพ์ได้ทันที** โดยใช้ `/` นำหน้า
Claude Code จะโหลดไฟล์ใน `commands/` แล้วรันตาม prompt ที่เขียนไว้

```
พิมพ์ใน Claude Code:  /ingest raw/clips/article.md
Claude จะ:           อ่าน commands/ingest.md → ทำตาม steps ทั้งหมด
```

---

## Skills คืออะไร?

Skills คือ **ชุดพฤติกรรมที่ Claude จะ activate อัตโนมัติ** เมื่อ context ตรง
ไม่ต้องพิมพ์ slash — Claude อ่าน SKILL.md แล้วรู้เองว่าต้องทำอะไร

เช่น ถ้าพูดว่า "ingest ไฟล์นี้ให้หน่อย" → skill `wiki-ingest` จะเปิดใช้อัตโนมัติ

---

## Commands ทั้งหมด

### `/ingest [path]`

**ไฟล์:** `commands/ingest.md`

เพิ่ม source ใหม่เข้า wiki ครบวงจร

```
/ingest raw/clips/article.md
/ingest raw/books/chapter1.md
/ingest raw/notes/2026-04-13-idea.md
```

**Claude จะทำ:**

1. อ่าน source ทั้งหมด
2. คุยสรุป 3–5 ประเด็นหลัก + ถามว่า focus ด้านไหน
3. สร้าง `wiki/sources/[slug].md`
4. สร้าง/อัปเดต concept pages ทุกตัวที่เจอใน source
5. อัปเดต book page ถ้า source เป็นหนังสือ
6. อัปเดต `index.md` และ `log.md`
7. รายงานว่าแตะไฟล์ไหนบ้าง

> ingest 1 ครั้งอาจแตะ 5–15 หน้า — ปกติ

---

### `/query [คำถาม]`

**ไฟล์:** `commands/query.md`

ถามคำถาม research จาก wiki ที่สะสมไว้

```
/query cognitive load theory คืออะไร
/query เปรียบเทียบ System 1 กับ System 2 ของ Kahneman
/query งานวิจัยไหนพูดถึง spaced repetition บ้าง
```

**Claude จะทำ:**

1. อ่าน `index.md` หา pages ที่เกี่ยวข้อง
2. อ่านหน้าเหล่านั้น + follow links ที่เกี่ยวข้อง
3. ตอบเป็นภาษาไทย พร้อม cite `[[wiki/concepts/...]]`
4. ถามว่า "ต้องการบันทึกเป็น synthesis page ไหม?"

> ถ้า wiki ยังว่าง Claude จะบอกว่าต้อง ingest source ก่อน

---

### `/lint`

**ไฟล์:** `commands/lint.md`

ตรวจสุขภาพ wiki ทั้งหมด — หาปัญหาและแนะนำสิ่งที่ควรทำต่อ

```
/lint
```

**Claude จะตรวจ:**
| จุดตรวจ | คืออะไร |
|---------|---------|
| Orphan pages | หน้าที่ไม่มีใครลิงก์มา |
| Broken links | `[[link]]` ที่ชี้ไปหน้าที่ไม่มีอยู่ |
| Missing concepts | concept ถูก mention 2+ ครั้งแต่ไม่มีหน้า |
| Contradictions | ข้อมูลขัดแย้งกันระหว่างหน้า |
| Stale content | หน้าที่ไม่ได้อัปเดตนาน > 60 วัน |
| Empty sections | heading ที่ไม่มีเนื้อหา |
| Data gaps | concept ที่มี source น้อยเกินไป |

Output เป็น checklist — แล้วถามว่าจะแก้ไหนทันที

> แนะนำ: รัน `/lint` ทุก 1–2 สัปดาห์

---

### `/status`

**ไฟล์:** `commands/status.md`

ดู dashboard สรุปสถานะ wiki ปัจจุบัน

```
/status
```

**ได้ข้อมูล:**

- จำนวนหน้าแยกตามประเภท (concepts, books, sources, synthesis)
- จำนวน raw files แยกตามโฟลเดอร์
- ไฟล์ใน `raw/` ที่ยังไม่ได้ ingest
- 5 activities ล่าสุดจาก `log.md`
- คำแนะนำ next action

> เริ่มต้น session ใหม่ด้วย `/status` เสมอ เพื่อรู้ว่า wiki อยู่ตรงไหน

---

### `/new-concept [ชื่อ concept]`

**ไฟล์:** `commands/new-concept.md`

สร้างหน้า concept เปล่าๆ โดยไม่ต้อง ingest source

```
/new-concept cognitive load
/new-concept การเรียนรู้แบบ spaced repetition
/new-concept ทฤษฎีแรงจูงใจภายใน
```

**ใช้เมื่อ:** รู้อยู่แล้วว่า concept คืออะไร อยากสร้างหน้าไว้รอก่อน แล้วค่อย ingest sources ทีหลัง

**Claude จะทำ:**

1. เช็คว่ามีหน้านี้อยู่แล้วไหม
2. สร้าง `wiki/concepts/[slug].md` พร้อม frontmatter ครบ
3. อัปเดต `index.md`

---

### `/new-book [ชื่อหนังสือ]`

**ไฟล์:** `commands/new-book.md`

สร้างหน้าสรุปหนังสือ

```
/new-book Thinking Fast and Slow
/new-book ทำไมสมองถึงโง่
/new-book Atomic Habits
```

**Claude จะถาม:** ชื่อผู้แต่ง, ปีที่พิมพ์, โน้ตเบื้องต้น
แล้วสร้าง `wiki/books/[slug].md` พร้อม structure ครบ รอ ingest แต่ละบททีหลัง

---

### `/synthesis [หัวข้อหรือคำถาม]`

**ไฟล์:** `commands/synthesis.md`

วิเคราะห์เชิงลึกจากหลาย sources แล้วบันทึกเป็นหน้าถาวร

```
/synthesis เปรียบเทียบทฤษฎีการเรียนรู้แบบต่างๆ
/synthesis งานวิจัยบอกว่าวิธีอ่านหนังสือแบบไหนดีที่สุด
/synthesis ความสัมพันธ์ระหว่าง dopamine กับ motivation
```

**ต่างจาก `/query` ตรงที่:** `/synthesis` ตั้งใจสร้างหน้าถาวรเสมอ ไม่ใช่แค่ตอบในแชท

**Claude จะทำ:**

1. อ่านทุกหน้าที่เกี่ยวข้องใน wiki
2. ถามว่าต้องการวิเคราะห์แบบไหน (เปรียบเทียบ / หา pattern / วิจารณ์ ฯลฯ)
3. เขียน synthesis เชิงลึก
4. บันทึกเป็น `wiki/synthesis/[slug].md`

---

### `/note [ข้อความ]`

**ไฟล์:** `commands/note.md`

จด quick note ไว้ใน `raw/notes/` โดยไม่ต้อง ingest ทันที

```
/note อ่าน paper เรื่อง attention นี้แล้วรู้สึกว่าขัดกับที่เคยเรียนมา ต้องหาข้อมูลเพิ่ม
/note บทความของ Cal Newport พูดถึง deep work ว่าต้องการ 4 ชั่วโมงต่อวัน
/note ไอเดีย: ลอง link concept นี้กับ habit loop ดู
```

**ใช้เมื่อ:** คิดอะไรขึ้นมาแต่ไม่อยากหยุด ingest ตอนนี้ — บันทึกไว้ก่อน ค่อย ingest ทีหลัง

บันทึกเป็น `raw/notes/YYYY-MM-DD-[slug].md` พร้อมวันที่

---

### `/update [path] [สิ่งที่จะเปลี่ยน]`

**ไฟล์:** `commands/update.md`

อัปเดตหน้า wiki ที่มีอยู่แล้ว โดยไม่ต้อง ingest source ใหม่

```
/update wiki/concepts/cognitive-load.md เพิ่มตัวอย่างจากชีวิตประจำวัน
/update wiki/books/atomic-habits.md แก้สรุปบทที่ 3 ใหม่
/update wiki/concepts/dopamine.md เพิ่ม link ไปที่ motivation
```

**Claude จะ:** อ่านหน้านั้นก่อน → merge ข้อมูลใหม่เข้าไป → ไม่ลบของเดิมทิ้ง

---

## Skills ทั้งหมด

Skills ทำงาน **อัตโนมัติ** ตาม context ไม่ต้องพิมพ์คำสั่งพิเศษ

| Skill         | Activate เมื่อ                                     | ทำอะไร                             |
| ------------- | -------------------------------------------------- | ---------------------------------- |
| `wiki-ingest` | พูดถึง ingest / วาง file path / "เพิ่ม source นี้" | รัน ingest workflow ครบวงจร        |
| `wiki-query`  | ถามคำถาม research ใดๆ                              | ค้น wiki → ตอบ → offer synthesis   |
| `wiki-lint`   | พูดว่า lint / ตรวจ / health check                  | ตรวจ wiki 7 จุด → รายงาน checklist |
| `wiki-status` | พูดว่า status / สถานะ / wiki มีอะไรบ้าง            | แสดง dashboard ปัจจุบัน            |

---

## Workflow ที่แนะนำ

### เริ่มต้น session ใหม่

```
/status
```

ดูว่า wiki อยู่ตรงไหน มีไฟล์ที่ยังไม่ ingest ไหม

### ทุกครั้งที่ clip บทความใหม่

```
1. Obsidian Web Clipper → บันทึกลง raw/clips/
2. /ingest raw/clips/[ชื่อไฟล์].md
```

### ทุกครั้งที่อยากรู้อะไร

```
/query [คำถาม]
```

ถ้า wiki ตอบได้ → ได้คำตอบทันที
ถ้าไม่มีข้อมูล → Claude จะแนะนำว่าต้อง ingest source อะไร

### ตรวจสุขภาพ wiki (ทุก 1–2 สัปดาห์)

```
/lint
```

### อยากวิเคราะห์เชิงลึก

```
/synthesis [หัวข้อ]
```

---

## หมายเหตุทางเทคนิค

- **Commands** ถูก load จาก `commands/*.md` ทุกครั้งที่พิมพ์ `/command-name`
- **Skills** ถูก load อัตโนมัติจาก `skills/*/SKILL.md` ตาม context matching
- **settings.json** ตั้งค่า `effortLevel: high` — Claude จะใช้ความพยายามสูงสุดในทุก operation
- แก้ไข command/skill ได้ตลอดเวลา — เปิดไฟล์ `.md` แล้วแก้ prompt ได้เลย
- ถ้าอยากเพิ่ม command ใหม่ → สร้างไฟล์ `.md` ใน `commands/` ได้เลย
