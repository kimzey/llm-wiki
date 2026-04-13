---
title: "AI Hooks System — ระบบ Trigger อัตโนมัติ"
type: concept
tags: [claude-code, hooks, security, automation, pre-tool-use, post-tool-use]
sources:
  - ai-context-phase1.md
  - ai-context-phase5.md
related:
  - wiki/concepts/claude-code-ai-cli.md
  - wiki/concepts/ai-agents-system.md
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

Hook system คือกลไกที่รัน script อัตโนมัติก่อน/หลัง tool call ของ AI ใช้สำหรับ security (บล็อก .env), quality control (auto-format, type check), และ workflow automation (audit, logging) Hook ทำงานกับ**ทุก context** ไม่ว่าจะเป็น main session, agent, skill, หรือ command

## อธิบาย

**Hook Types:**

| Type | ทำงานเมื่อ | ใช้ทำอะไร |
|---|---|---|
| **SessionStart** | เปิด session ใหม่ | Initialize state |
| **PreToolUse** | ก่อน tool ทำงาน | Validate, block, modify |
| **PostToolUse** | หลัง tool ทำงาน | Format, check, log |
| **PreCompact** | ก่อน context compact | Save important context |
| **Stop** | Session จบ (agent หยุด) | Final audit |
| **SessionEnd** | Session ปิดสมบูรณ์ | Cleanup, evaluate |

**Hook Protocol (stdin/stdout):**
```
Claude Code → stdin: JSON tool input → Hook script
Hook script → stdout: modified JSON (exit 0) หรือ error message (exit 1/2)

exit 0 = อนุญาต (stdout = modified JSON)
exit 1 = error แต่ไม่บล็อก (stderr = warning แสดงให้ user)
exit 2 = บล็อก! (stderr = reason) — tool call ถูกยกเลิก
```

**Matcher Syntax:**
```javascript
"matcher": "Read"                          // simple: match tool name
"matcher": "tool == \"Bash\" && tool_input.command matches \"git push\""
"matcher": "*"                             // wildcard: ทุก tool
"matcher": "tool == \"Edit\" && tool_input.file_path matches \"\\\\.(ts|tsx)$\""
```

## ประเด็นสำคัญ

**env-protection.sh (hook หลักในโปรเจกต์):**
```bash
# รับ JSON จาก stdin
file_path=$(echo "$input" | python3 -c "import sys,json; ...")
if [[ "$basename" == ".env" || "$basename" == .env.* ]]; then
    echo "Blocked: AI is not allowed to read .env files" >&2
    exit 2  # บล็อก
fi
```

**Global hooks ที่มีใน ~/.claude/settings.json (ตัวอย่าง):**
- `tmux-reminder` — PreToolUse: แนะนำ tmux สำหรับ long-running commands
- `git-push-review` — PreToolUse: เตือนก่อน git push
- `prettier` — PostToolUse: auto-format .js/.ts/.tsx หลัง Edit
- `tsc-check` — PostToolUse: TypeScript check หลัง Edit .ts/.tsx
- `console-log-warning` — PostToolUse: เตือนถ้ามี console.log

**Hook เป็น Global Security Layer:**
- ทำงานกับ**ทุก context** — main session, agent subprocess, skill, command
- ไม่มีทางหลีกเลี่ยงผ่าน agent หรือ skill

## ตัวอย่าง / กรณีศึกษา

**เพิ่ม hook บล็อก generated files (วิธีขยายระบบ):**
```bash
# .claude/hooks/protect-generated.sh
file_path=$(echo "$input" | python3 -c "...")
if [[ "$file_path" == *"spec.gen.go"* ]] || [[ "$file_path" == *".pb.go"* ]]; then
    echo "Blocked: Cannot edit generated file" >&2
    exit 2
fi
```
```json
// .claude/settings.json — เพิ่ม PreToolUse สำหรับ Edit
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit",
        "hooks": [{ "type": "command", "command": "bash .claude/hooks/protect-generated.sh" }]
      }
    ]
  }
}
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/claude-code-ai-cli|Claude Code AI CLI]] — Hook configs อยู่ใน settings.json

## แหล่งที่มา

- [[wiki/sources/ai-context/ai-context-phase1|Phase 1: โครงสร้าง .claude]]
- [[wiki/sources/ai-context/ai-context-phase5|Phase 5: Deep Dive — Hook Lifecycle]]
