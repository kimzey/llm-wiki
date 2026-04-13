---
title: "JavaScript Runtime"
type: concept
tags: [javascript, runtime, nodejs, bun, deno]
sources: [raw/notes/javascript/bun-runtime.md]
related: []
created: 2026-04-14
updated: 2026-04-14
---

## สรุปสั้น

JavaScript Runtime คือสภาพแวดล้อมที่ execute JavaScript code นอก browser ประกอบด้วย JavaScript engine, APIs สำหรับ system-level operations, และ package manager

## อธิบาย

Runtime ทำให้ JavaScript สามารถ run บน server, desktop, หรือ command-line ได้ ไม่จำกัดอยู่แค่ใน browser แต่ละ runtime มีจุดเด่นและ trade-offs ที่แตกต่างกัน เช่น performance, ecosystem compatibility, และ built-in features

## ประเด็นสำคัญ

- **V8 Engine** - JavaScript engine หลักที่ใช้ใน Node.js, Bun, Deno พัฒนาโดย Google สำหรับ Chrome
- **Event Loop** - Non-blocking I/O model ที่ทำให้ JavaScript เหมาะกับ I/O intensive operations
- **Package Management** - แต่ละ runtime มี package manager เป็นของตัวเอง (npm, pnpm, bun, yarn)
- **TypeScript Support** - บาง runtime เช่น Bun รองรับ TypeScript โดยตรงโดยไม่ต้อง compile
- **Cross-platform** - Runtime ส่วนใหญ่ทำงานได้บน Windows, macOS, Linux

## ตัวอย่าง / กรณีศึกษา

### Node.js

- Runtime ที่นิยมที่สุด
- มี ecosystem ขนาดใหญ่ที่สุด (npm registry)
- ต้อง compile TypeScript ก่อนรัน
- ใช้ CommonJS หรือ ES modules

### Bun Runtime

จาก [[wiki/sources/javascript/bun-runtime]]:

- **เร็วกว่า Node.js 6 เท่า** ใน server startup
- **Package installation เร็ว 30 เท่า**
- **Native TypeScript support** - ไม่ต้อง compile
- **All-in-one** - รวม runtime, bundler, test runner, package manager
- **Built-in APIs**:
  - `Bun.file()` - File I/O
  - `Bun.env` - Environment variables
  - `Bun.sql` - SQL client
  - `Bun.write()` - Write files

### Deno

- สร้างโดย Ryan Dahl (creator ของ Node.js)
- Secure by default (explicit permission)
- Built-in TypeScript support
- Uses ES modules เป็น default
- Standard library ที่ strong

## Performance Comparison

จาก [[wiki/sources/javascript/bun-runtime]]:

| Runtime | Startup Speed | Package Install | TypeScript |
|---------|--------------|----------------|------------|
| **Bun** | 50ms | 1s (30x faster) | Native |
| **Node.js** | 300ms | 30s | Transpiled |
| **Deno** | ~150ms | ~10s | Native |

## Migration Considerations

เมื่อ migrate จาก Node.js เป็น Bun:

- ไม่มี `__dirname` - ใช้ `import.meta.dir` แทน
- Node.js native addons อาจไม่ทำงาน
- APIs บางตัวทำงานแตกต่าง (เช่น `process.cwd()`)

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/typescript-programming-language|TypeScript]] - Runtime บางตัวรองรับ TypeScript โดยตรง
- [[wiki/concepts/package-manager|Package Manager]] - แต่ละ runtime มี package manager เป็นของตัวเอง

## แหล่งที่มา

- [[wiki/sources/javascript/bun-runtime]] - Bun Runtime Guide
