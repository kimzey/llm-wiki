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
├── settings.json          ← ค่าตั้งต้นของโปรเจ็ค
├── commands/              ← slash commands (/ingest, /query, ฯลฯ)
│   ├── ingest.md
│   ├── query.md
│   ├── lint.md
│   ├── status.md
│   ├── research.md
│   ├── new-concept.md
│   ├── new-book.md
│   ├── synthesis.md
│   ├── note.md
│   └── update.md
└── skills/                ← พฤติกรรมที่ activate อัตโนมัติตาม context
    ├── wiki-ingest/
    ├── wiki-query/
    ├── wiki-lint/
    ├── wiki-status/
    └── wiki-research/
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

```
/ingest raw/clips/article.md
/ingest raw/notes/otel/
```

---

### `/query [คำถาม]`
ถามคำถาม research จาก wiki ที่สะสมไว้

```
/query cognitive load theory คืออะไร
/query เปรียบเทียบ System 1 กับ System 2 ของ Kahneman
```

---

### `/research [หัวข้อ]`
ค้น web เพื่อเติม gap ที่ยังขาดใน wiki

```
/research distributed tracing
/research              ← รัน /lint ก่อนเพื่อหา gaps แล้วเสนอหัวข้อ
```

---

### `/lint`
ตรวจสุขภาพ wiki ทั้งหมด — หาปัญหาและแนะนำสิ่งที่ควรทำต่อ

> แนะนำ: รันทุก 1–2 สัปดาห์

---

### `/status`
ดู dashboard สรุปสถานะ wiki ปัจจุบัน: จำนวนหน้า, ไฟล์ที่ยังไม่ ingest, กิจกรรมล่าสุด, next actions

> เริ่มต้น session ใหม่ด้วย `/status` เสมอ

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

```
/update wiki/concepts/cognitive-load.md เพิ่มตัวอย่างจากชีวิตประจำวัน
/update wiki/books/atomic-habits.md แก้สรุปบทที่ 3 ใหม่
```

---

## Skills ทั้งหมด

| Skill           | Activate เมื่อ                         | ทำอะไร                              |
|-----------------|----------------------------------------|-------------------------------------|
| `wiki-ingest`   | พูดถึง ingest / วาง file path          | รัน ingest workflow ครบวงจร         |
| `wiki-query`    | ถามคำถาม research ใดๆ                  | ค้น wiki → ตอบ → offer synthesis    |
| `wiki-research` | พูดถึง gap / ต้องการค้น web            | ค้น web → สรุป → อัปเดต wiki        |
| `wiki-lint`     | พูดว่า lint / ตรวจ / health check      | ตรวจ wiki 7 จุด → รายงาน checklist  |
| `wiki-status`   | พูดว่า status / wiki มีอะไรบ้าง        | แสดง dashboard ปัจจุบัน             |

---

## Workflow ที่แนะนำ

### เริ่มต้น session ใหม่
```
/status
```

### ทุกครั้งที่ clip บทความใหม่
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

---

## หมายเหตุทางเทคนิค

- **Commands** ถูก load จาก `commands/*.md` ทุกครั้งที่พิมพ์ `/command-name`
- **Skills** ถูก load อัตโนมัติจาก `skills/*/SKILL.md` ตาม context matching
- **settings.json** ตั้งค่า `effortLevel: high` — Claude จะใช้ความพยายามสูงสุดในทุก operation
- แก้ไข command/skill ได้ตลอดเวลา — เปิดไฟล์ `.md` แล้วแก้ prompt ได้เลย
- ถ้าอยากเพิ่ม command ใหม่ → สร้างไฟล์ `.md` ใน `commands/` ได้เลย
