---
title: "TypeScript"
type: concept
tags: [programming-language, javascript, type-system, microsoft]
sources: [raw/notes/javascript/bun-runtime.md]
related: []
created: 2026-04-14
updated: 2026-04-14
---

## สรุปสั้น

TypeScript เป็น superset ของ JavaScript ที่เพิ่ม static typing เข้าไป พัฒนาโดย Microsoft ช่วยให้ catch errors ได้ตั้งแต่ขั้นตอน development และทำให้ codebase ขนาดใหญ่จัดการได้ง่ายขึ้น

## อธิบาย

TypeScript คือ JavaScript ที่มี type system เพิ่มเข้ามา สามารถ compile เป็น JavaScript ธรรมดาได้ ทำให้ run บน browser หรือ Node.js ได้ทุกที่ ประโยชน์หลักคือการช่วย prevent bugs ที่เกิดจาก type errors และทำให้ IDE สามารถ provide better autocomplete และ refactoring tools ได้ดีขึ้น

## ประเด็นสำคัญ

- **Static Type Checking** - ตรวจสอบประเภทข้อมูลขณะ compile time ไม่ใช่ runtime
- **Type Inference** - สามารถ推断 types ได้อัตโนมัติโดยไม่ต้องประกาศทุกครั้ง
- **Interface & Type Aliases** - ช่วย define data structures ได้ชัดเจน
- **Generics** - Reusable components ที่รองรับหลายประเภทข้อมูล
- **Decorators** - Meta-programming feature สำหรับ class และ members
- **ES6+ Support** - รองรับ modern JavaScript features ทั้งหมด

## ตัวอย่าง / กรณีศึกษา

### Basic Type Annotation

```typescript
// Interface definition
interface User {
  id: number;
  name: string;
  email: string;
}

// Function with typed parameters and return type
function getUserById(id: number): User | null {
  // Implementation
  return null;
}
```

### Native TypeScript in Bun

จาก [[wiki/sources/javascript/bun-runtime]] - Bun รองรับ TypeScript โดยตรงโดยไม่ต้อง compile:

```typescript
// app.ts - No build step needed!
import { serve } from 'bun'

serve({
  port: 3000,
  fetch(req) {
    return new Response('Hello from Bun!')
  }
})
```

### Generics Example

```typescript
function identity<T>(arg: T): T {
  return arg;
}

const num = identity<number>(42);
const str = identity<string>("hello");
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/javascript-runtime|JavaScript Runtime]] - TypeScript ต้องถูก compile หรือรันโดย runtime ที่รองรับ
- [[wiki/concepts/package-manager|Package Manager]] - TypeScript ecosystem มี libraries และ tools มากมาย

## แหล่งที่มา

- [[wiki/sources/javascript/bun-runtime]] - Bun Runtime Guide
