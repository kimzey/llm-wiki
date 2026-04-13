---
title: "Package Manager"
type: concept
tags: [npm, bun, dependency-management, javascript]
sources: [raw/notes/javascript/bun-runtime.md]
related: []
created: 2026-04-14
updated: 2026-04-14
---

## สรุปสั้น

Package Manager คือเครื่องมือสำหรับจัดการ dependencies ในโปรเจกต์ software ทำให้การติดตั้ง, update, และ manage libraries ต่างๆ เป็นไปอย่างมีระเบียบและ reproduce ได้

## อธิบาย

Package Manager ทำหน้าที่:
- ติดตั้ง packages และ dependencies จาก registry
- จัดการ version conflicts
- Maintain lock files สำหรับ consistency ระหว่าง environments
- Run scripts ต่างๆ (test, build, dev)
- Cache packages เพื่อเร็วขึ้นในการติดตั้งครั้งต่อไป

## ประเด็นสำคัญ

- **Dependency Resolution** - แก้ปัญหา conflicts ระหว่าง packages ที่ต้องการ version ต่างกัน
- **Lock Files** - `package-lock.json`, `bun.lockb`, `yarn.lock` รับประกันว่าทุกคนได้ dependencies เหมือนกัน
- **Registry** - Central repository สำหรับ packages (เช่น npmjs.org)
- **SemVer** - Semantic Versioning (MAJOR.MINOR.PATCH) สำหรับ version management
- **Workspaces** - จัดการ monorepo ที่มีหลาย packages ใน repo เดียว

## ตัวอย่าง / กรณีศึกษา

### npm (Node Package Manager)

```bash
npm install          # Install dependencies
npm add <package>    # Add a package
npm run dev          # Run script
```

- Package manager มาตรฐานสำหรับ Node.js
- ติดตั้งช้า (30 วินาที สำหรับ projects ขนาดกลาง)

### bun (Built-in with Bun Runtime)

จาก [[wiki/sources/javascript/bun-runtime]]:

```bash
bun install          # Install dependencies (100x faster than npm)
bun add <package>    # Add a package
bun run dev          # Run script
bun --watch run src/index.ts  # Hot reload
```

**ข้อดี:**
- **เร็วมาก** - 30x เร็วกว่า `npm install`
- **All-in-one** - Built-in ใน Bun, ไม่ต้องติดตั้งแยก
- **Compatible** - อ่าน `package.json` ได้, ใช้งานร่วมกับ npm packages ได้

### yarn

```bash
yarn install
yarn add <package>
yarn <command>
```

- พัฒนาโดย Facebook
- มี PnP (Plug'n'Play) สำหรับ performance ที่ดีขึ้น
- Workspaces support ดีมาก

### pnpm

```bash
pnpm install
pnpm add <package>
```

- ใช้ hard links และ symbolic links
- ประหยัด disk space มาก
- เหมาะกับ monorepos

## Performance Comparison

จาก [[wiki/sources/javascript/bun-runtime]]:

| Package Manager | Install Time | Relative Speed |
|----------------|--------------|----------------|
| **bun** | 1s | 30x faster |
| **npm** | 30s | 1x (baseline) |
| **yarn** | ~15s | 2x faster |
| **pnpm** | ~10s | 3x faster |

## Best Practices

1. **Use lock files** - Commit lock files ใน version control
2. **Pin versions** - ระบุ exact versions สำหรับ production
3. **Regular updates** - Run `npm outdated` เพื่อ check dependencies ที่ล้าสมัย
4. **Security audits** - Run `npm audit` เพื่อหา vulnerabilities
5. **Clean installs** - ใช้ `rm -rf node_modules && bun install` ถ้ามีปัญหา

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/javascript-runtime|JavaScript Runtime]] - แต่ละ runtime มี package manager เป็นของตัวเอง (Bun มี bun, Node.js ใช้ npm)
- [[wiki/concepts/typescript-programming-language|TypeScript]] - TypeScript packages ต้องถูกติดตั้งผ่าน package manager

## แหล่งที่มา

- [[wiki/sources/javascript/bun-runtime]] - Bun Runtime Guide
