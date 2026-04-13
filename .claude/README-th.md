# .claude — คู่มือการใช้งาน

โฟลเดอร์นี้คือ "สมองของ Claude Code" สำหรับ vault นี้
ประกอบด้วย commands (คำสั่ง slash), skills (พฤติกรรมอัตโนมัติ), และ settings

ดูเวอร์ชันภาษาอังกฤษได้ที่ [README.md](README.md)

---

## โครงสร้างไฟล์

```
.claude/
├── README.md              ← English version
├── README-th.md           ← ไฟล์นี้ (ภาษาไทย)
├── plan.md                ← แผนการ update skills
├── settings.json          ← ค่าตั้งต้นของโปรเจ็ค
├── commands/              ← slash commands (/ingest, /query, ฯลฯ)
│   ├── ingest.md
│   ├── query.md
│   ├── lint.md
│   ├── status.md
│   ├── research.md
│   ├── clip.md            ← ใหม่: clip URL → raw/clips/
│   ├── canvas.md          ← ใหม่: สร้าง visual knowledge map
│   ├── base.md            ← ใหม่: สร้าง Obsidian Bases view
│   ├── new-concept.md
│   ├── new-book.md
│   ├── synthesis.md
│   ├── note.md
│   └── update.md
└── skills/                ← พฤติกรรมที่ activate อัตโนมัติตาม context
    ├── wiki-ingest/       ← ingest workflow (ใช้ obsidian-markdown + obsidian-cli)
    ├── wiki-query/        ← query workflow (ใช้ obsidian-cli search)
    ├── wiki-lint/         ← lint workflow (ใช้ obsidian-cli backlinks)
    ├── wiki-status/       ← status dashboard (ใช้ obsidian-cli tags)
    ├── wiki-research/     ← research workflow (ใช้ defuddle)
    ├── defuddle/          ← ดึง clean markdown จาก URL
    ├── json-canvas/       ← สร้าง/แก้ไข .canvas files
    ├── obsidian-bases/    ← สร้าง/แก้ไข .base database views
    ├── obsidian-cli/      ← ควบคุม Obsidian ผ่าน CLI
    └── obsidian-markdown/ ← เขียน Obsidian Flavored Markdown ถูกต้อง
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
เพิ่ม source ใหม่เข้า wiki ครบวงจร รับได้ทั้งไฟล์เดี่ยวและ folder
ใช้ **obsidian-markdown** syntax เมื่อสร้างหน้า wiki ทุกหน้า
ใช้ **obsidian-cli** verify ไฟล์ได้ถ้าเปิด Obsidian อยู่

```
/ingest raw/clips/article.md
/ingest raw/notes/otel/
```

---

### `/query [คำถาม]`
ถามคำถาม research จาก wiki ที่สะสมไว้
ใช้ **obsidian-cli** search เพิ่มเติมจาก index.md ถ้าเปิด Obsidian อยู่

```
/query cognitive load theory คืออะไร
/query เปรียบเทียบ System 1 กับ System 2 ของ Kahneman
```

---

### `/research [หัวข้อ]`
ค้น web เพื่อเติม gap ที่ยังขาดใน wiki
ใช้ **defuddle** อ่านหน้าเว็บแทน WebFetch — ได้ Markdown ที่สะอาดกว่า

```
/research distributed tracing
/research              ← รัน /lint ก่อนเพื่อหา gaps แล้วเสนอหัวข้อ
```

---

### `/clip [url]`
Clip หน้าเว็บลง `raw/clips/` ด้วย defuddle แล้วถามว่าจะ ingest เลยไหม

```
/clip https://example.com/article
```

---

### `/lint`
ตรวจสุขภาพ wiki ทั้งหมด — หาปัญหาและแนะนำสิ่งที่ควรทำต่อ
ใช้ **obsidian-cli** backlinks ได้ถ้าเปิด Obsidian อยู่

> แนะนำ: รันทุก 1–2 สัปดาห์

---

### `/status`
ดู dashboard สรุปสถานะ wiki ปัจจุบัน: จำนวนหน้า, ไฟล์ที่ยังไม่ ingest, กิจกรรมล่าสุด, next actions
แสดง tag stats ด้วย **obsidian-cli** ได้ถ้าเปิด Obsidian อยู่

> เริ่มต้น session ใหม่ด้วย `/status` เสมอ

---

### `/canvas [หัวข้อ]`
สร้าง visual knowledge map (`.canvas` file) ของหัวข้อ โดยใช้ wiki pages เป็น nodes

```
/canvas RAG pipeline
/canvas machine learning fundamentals
```

---

### `/base [ชื่อ]`
สร้าง Obsidian Bases database view (`.base` file) สำหรับดู wiki content แบบ table/cards

```
/base sources     ← ตารางแสดง sources ทั้งหมด
/base concepts    ← card gallery ของ concepts
/base books       ← book library table
/base all         ← full wiki dashboard
```

---

### `/new-concept [ชื่อ concept]`
สร้างหน้า concept เปล่าๆ โดยไม่ต้อง ingest source

```
/new-concept cognitive load
/new-concept การเรียนรู้แบบ spaced repetition
```

---

### `/new-book [ชื่อหนังสือ]`
สร้างหน้าสรุปหนังสือ

```
/new-book Thinking Fast and Slow
/new-book Atomic Habits
```

---

### `/synthesis [หัวข้อหรือคำถาม]`
วิเคราะห์เชิงลึกจากหลาย sources แล้วบันทึกเป็นหน้าถาวร
ใช้ **obsidian-markdown** syntax เมื่อสร้างหน้า

```
/synthesis เปรียบเทียบทฤษฎีการเรียนรู้แบบต่างๆ
/synthesis ความสัมพันธ์ระหว่าง dopamine กับ motivation
```

ต่างจาก `/query` ตรงที่: `/synthesis` ตั้งใจสร้างหน้าถาวรเสมอ

---

### `/note [ข้อความ]`
จด quick note ไว้ใน `raw/notes/` โดยไม่ต้อง ingest ทันที

```
/note อ่าน paper เรื่อง attention แล้วรู้สึกว่าขัดกับที่เคยเรียนมา
```

---

### `/update [path] [สิ่งที่จะเปลี่ยน]`
อัปเดตหน้า wiki ที่มีอยู่แล้ว โดยไม่ต้อง ingest source ใหม่
ใช้ **obsidian-markdown** syntax เมื่อแก้ไข

```
/update wiki/concepts/cognitive-load.md เพิ่มตัวอย่างจากชีวิตประจำวัน
/update wiki/books/atomic-habits.md แก้สรุปบทที่ 3 ใหม่
```

---

## Skills ทั้งหมด

| Skill               | Activate เมื่อ                          | ทำอะไร                                    |
|---------------------|----------------------------------------|-------------------------------------------|
| `wiki-ingest`       | พูดถึง ingest / วาง file path          | ingest workflow + obsidian-markdown       |
| `wiki-query`        | ถามคำถาม research ใดๆ                  | ค้น wiki + cli search → ตอบ              |
| `wiki-research`     | พูดถึง gap / ต้องการค้น web            | defuddle URLs → สรุป → อัปเดต wiki       |
| `wiki-lint`         | พูดว่า lint / ตรวจ / health check      | ตรวจ 7 จุด + cli backlinks               |
| `wiki-status`       | พูดว่า status / wiki มีอะไรบ้าง        | dashboard + cli tag stats                |
| `defuddle`          | user ส่ง URL มาให้อ่าน                 | ดึง clean markdown จากเว็บ               |
| `obsidian-markdown` | สร้าง/แก้ไข .md ใดๆ ใน wiki            | ใช้ syntax Obsidian ที่ถูกต้อง           |
| `obsidian-cli`      | search vault / verify / backlinks      | ควบคุม Obsidian จาก terminal             |
| `json-canvas`       | ทำงานกับ .canvas files                 | สร้าง/แก้ไข visual canvas               |
| `obsidian-bases`    | ทำงานกับ .base files                   | สร้าง/แก้ไข database views              |

---

## Workflow ที่แนะนำ

### เริ่มต้น session ใหม่
```
/status
```

### Clip บทความจาก URL โดยตรง
```
/clip https://example.com/article
→ Claude clip ลง raw/clips/ และถามว่าจะ ingest เลยไหม
```

### ทุกครั้งที่ clip ผ่าน Obsidian Web Clipper
```
1. Obsidian Web Clipper → บันทึกลง raw/clips/
2. /ingest raw/clips/[ชื่อไฟล์].md
```

### ทุกครั้งที่อยากรู้อะไร
```
/query [คำถาม]
```

### เติม gap ใน wiki
```
/research [หัวข้อ]
```

### ตรวจสุขภาพ wiki (ทุก 1–2 สัปดาห์)
```
/lint
```

### วิเคราะห์เชิงลึก
```
/synthesis [หัวข้อ]
```

### ดู knowledge map แบบภาพ
```
/canvas [หัวข้อ]
```

### ดู wiki เป็น database view
```
/base sources     หรือ     /base all
```

---

## หมายเหตุทางเทคนิค

- **Commands** ถูก load จาก `commands/*.md` ทุกครั้งที่พิมพ์ `/command-name`
- **Skills** ถูก load อัตโนมัติจาก `skills/*/SKILL.md` ตาม context matching
- **settings.json** ตั้งค่า `effortLevel: high` — Claude จะใช้ความพยายามสูงสุดในทุก operation
- **obsidian-cli** steps เป็น optional เสมอ — ต้องเปิด Obsidian อยู่จึงจะใช้ได้ มี fallback ทุก step
- แก้ไข command/skill ได้ตลอดเวลา — เปิดไฟล์ `.md` แล้วแก้ prompt ได้เลย
- ถ้าอยากเพิ่ม command ใหม่ → สร้างไฟล์ `.md` ใน `commands/` ได้เลย
