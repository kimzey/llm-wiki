---
title: "Learn Phase 7: Global Agents, Auto Memory, Plan Mode + คู่มือการใช้งานจริง"
type: source
source_file: raw/notes/ai-context/LearnPhase7-Final-Manual สิ่งที่ยังไม่ได้ครอบคลุม + คู่มือการใช้งานจริง.md
url: ""
published: 2026-04-13
tags: [claude-code, global-agents, memory, plan-mode, context-window, practical-guide]
related:
  - wiki/concepts/ai-agents-system.md
  - wiki/concepts/claude-code-ai-cli.md
  - wiki/concepts/ai-dispatch-system.md
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/ai-context/LearnPhase7-Final-Manual สิ่งที่ยังไม่ได้ครอบคลุม + คู่มือการใช้งานจริง.md|Original file]]

## สรุป

Phase 7 ครอบคลุมสิ่งที่ Phase 1-6 ยังไม่ได้พูดถึง: Global Agents 10 ตัว, Auto Memory System, Plan Mode, Context Window Management, Model Selection และส่วนสำคัญคือ **คู่มือการใช้งานจริง 10 recipes** พร้อม Troubleshooting Guide

## ประเด็นสำคัญ

**Global Agents 10 ตัว** (ใช้ได้ทุกโปรเจกต์ ไม่มี MCP tools):
| Agent | ใช้เมื่อ |
|---|---|
| planner | feature ซับซ้อน, refactoring |
| architect | architectural decisions |
| code-reviewer | หลังเขียนโค้ดเสร็จ |
| tdd-guide | feature ใหม่, bug fix |
| security-reviewer | user input, auth, API |
| build-error-resolver | build fail, type errors |
| database-reviewer | SQL, migration, schema |
| e2e-runner | critical user flows |
| refactor-cleaner | dead code cleanup |
| doc-updater | documentation outdated |

**Auto Memory System:**
- เก็บที่ `~/.claude/projects/<hash>/memory/MEMORY.md`
- โหลดทุก session เหมือน CLAUDE.md
- ใช้จำ patterns, decisions, user preferences ข้าม sessions

**Plan Mode:**
1. Claude เข้า Plan Mode → สำรวจ codebase
2. เขียนแผนลงไฟล์ → user approve/แก้/reject
3. Claude implement ตามแผน

**Context Window Management:**
- Auto-compact เมื่อใกล้เต็ม (~200k tokens)
- PreCompact hook บันทึก context ก่อน compact
- Best practice: แบ่งงานเป็น phases, compact ระหว่าง phase

**Model Selection:**
- default: opus (ความสามารถสูง)
- Agent สามารถ override model ได้ใน YAML front matter

## ข้อมูลที่น่าสนใจ

**10 Practical Recipes:**
1. สร้าง Domain ใหม่ (3 วิธี: command, agent, plan first)
2. เพิ่ม Use Case Method
3. แก้ TypeSpec แล้ว Regenerate
4. ตรวจสอบคุณภาพโค้ด (parallel agents)
5. Debug Build Fail (build-error-resolver agent)
6. สร้าง Jira Tickets (PO agent)
7. เขียน Tests จาก Acceptance Criteria (QA agent)
8. Architecture Review ก่อน Merge
9. จัดการ Context งานยาว (compact ตาม phase)
10. Onboarding สมาชิกใหม่

**Troubleshooting สำคัญ:**
- Agent ไม่เห็น context → ใส่ context ครบใน Task prompt
- AI ไม่เรียก Agent ที่ควรเรียก → ระบุชื่อ agent ตรงๆ
- Command ไม่ทำตาม steps → /compact แล้วสั่งใหม่ หรือทำทีละ step

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/ai-agents-system|AI Agents System]]
- [[wiki/concepts/claude-code-ai-cli|Claude Code AI CLI]]
- [[wiki/concepts/ai-dispatch-system|AI Dispatch System]]
