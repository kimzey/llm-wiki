---
title: "Bun Runtime Guide"
type: source
source_file: raw/notes/javascript/bun-runtime.md
url: https://bun.sh/docs
published: 2026-04-14
tags: [bun, javascript, typescript, runtime, package-manager]
related: []
created: 2026-04-14
updated: 2026-04-14
---

> **Full source**: [[../../../raw/notes/javascript/bun-runtime.md|Original file]]

## สรุป

คู่มีบุนธรรม Bun Runtime ซึ่งเป็น JavaScript runtime ที่ทันสมัย ทำงานเร็ว และเป็น drop-in replacement สำหรับ Node.js เขียนด้วยภาษา Zig และรองรับ TypeScript โดยตรงโดยไม่ต้องมีการ compile

## ประเด็นสำคัญ

- **Native TypeScript Support** - รัน TypeScript ได้โดยตรงโดยไม่ต้อง compile step
- **Built-in Package Manager** - `bun install` เร็วกว่า `npm install` ถึง 100 เท่า
- **All-in-one Toolkit** - รวม runtime, package manager, test runner, bundler ไว้ในตัวเดียว
- **Built-in SQL Client** - สามารถ query ฐานข้อมูลโดยตรงโดยไม่ต้องใช้ library เพิ่ม
- **Performance** - Server startup เร็วกว่า Node.js 6 เท่า, package installation เร็ว 30 เท่า

## ข้อมูล / หลักฐาน ที่น่าสนใจ

### Performance Comparison

| Operation | Bun | Node.js | Speedup |
|-----------|-----|---------|---------|
| `npm install` | 1s | 30s | 30x faster |
| Server startup | 50ms | 300ms | 6x faster |
| TypeScript execution | Native | Transpiled | Instant |

### Key APIs

- `Bun.file()` - File I/O operations
- `Bun.env` - Environment variables access (ไม่ต้องใช้ dotenv)
- `Bun.sql` - Built-in SQL client
- `Bun.write()` - Write files
- `bun --watch` - Hot reload ใน development mode

### Migration Notes

- ไม่มี `__dirname` - ใช้ `import.meta.dir` แทน
- Node.js addons อาจจะไม่ทำงาน - ควรใช้ Bun-native alternatives
- APIs บางตัวเช่น `process.cwd()` ทำงานแตกต่างจาก Node.js

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อมูลที่ขัดแย้งกับ wiki ที่มีอยู่

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/typescript-programming-language|TypeScript]] (ควรสร้างใหม่)
- [[wiki/concepts/javascript-runtime|JavaScript Runtime]] (ควรสร้างใหม่)
- [[wiki/concepts/package-manager|Package Manager]] (ควรสร้างใหม่)
