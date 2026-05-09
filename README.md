# skill-surgeon ✂️

**Surgical, safe updates for AI skill files.**  
**AI Skill 文件的外科式安全更新工具。**

[![Install](https://img.shields.io/badge/install-skills.sh-blue)](https://skills.sh/darian-deng/skill-surgeon)
[![GitHub](https://img.shields.io/badge/github-darian--deng%2Fskill--surgeon-black)](https://github.com/darian-deng/skill-surgeon)

---

## The Problem / 问题背景

When you ask an AI to update a specific part of a skill file, it often **rewrites the entire file** — silently discarding carefully crafted rules, examples, and constraints you spent hours building.

当你让 AI 修改一个 skill 文件的某个具体部分时，它往往会**把整个文件重写一遍**——悄无声息地丢掉你精心设计的规则、示例和约束。

```
User: "Change line 42 from X to Y"
AI:   *rewrites all 300 lines, loses 80% of the original content*
```

This happens because:
- LLMs default to `Write` (full overwrite) when editing feels complex
- There's no verification that "only the intended change was made"
- `git diff` catches it — but only after the damage is done

原因在于：
- LLM 在编辑复杂文件时倾向于使用 `Write` 工具（全量覆写）
- 没有机制验证"是否只做了预期的改动"
- `git diff` 能发现问题——但往往是损失发生之后

---

## The Solution / 解决方案

**skill-surgeon** enforces a **Safe Edit Protocol** on every skill file update:

```
1. Read file     → snapshot before
2. Edit (only!)  → apply the specific change via str_replace
3. Read file     → snapshot after
4. Compare       → list every changed line, flag unintended ones
5. Report        → ✅ clean / ❌ restore + retry
```

**No git. No Python. No external dependencies.**  
Works in Claude Code, Claude.ai, CI environments, everywhere.

---

## Install / 安装

```bash
# Claude Code (global)
npx skills add darian-deng/skill-surgeon -g -a claude-code

# Or with all agents
npx skills add darian-deng/skill-surgeon -g --all
```

One-click via skills.sh: [skills.sh/darian-deng/skill-surgeon](https://skills.sh/darian-deng/skill-surgeon)

---

## Usage / 使用

**For updating an existing skill** (the primary use case):
```
/skill-surgeon
"Update the S3 section of my ai-flow skill to add a broadcast line"
```

**For creating a new skill** (same as official skill-creator):
```
/skill-surgeon
"Create a new skill for X"
```

---

## vs Official skill-creator / 与官方 skill-creator 的对比

| | skill-creator (official) | skill-surgeon (this) |
|---|---|---|
| Create new skills | ✅ Full workflow | ✅ Full workflow |
| Update existing skills | ⚠️ May rewrite entire file | ✅ Edit-only + verified |
| Unintended change detection | ❌ Not built-in | ✅ Built-in |
| Revert on bad edit | ❌ Manual | ✅ Automatic |
| External dependencies | Python (scripts) | None for safety protocol |

**Rule of thumb**: Use official `skill-creator` if you're starting from scratch. Use `skill-surgeon` whenever you're touching an existing skill file.

**使用原则**：从零创建新 skill → 用官方 `skill-creator`；修改已有 skill → 用 `skill-surgeon`。

---

## How it works / 原理

The Safe Edit Protocol is built into the skill's SKILL.md. When the AI reads the skill file before making any changes, it internalizes the constraint:

> "Read first → Edit only (never Write) → Read again → Compare → Report or Restore"

This is a **soft constraint** (text-based instruction), not a hard OS-level lock. The AI follows it because the instruction is prominent and explicit. Combined with the before/after comparison, any violation is immediately visible and reversible.

安全编辑协议内嵌在 SKILL.md 中。AI 在修改文件之前读取它，就会内化这个约束：先读取快照 → 只用 Edit 工具 → 再次读取 → 上下文比对 → 报告或还原。这是软约束（文本指令），不是系统级锁定，但结合前后快照对比，任何违规都立即可见且可还原。

---

## Background / 背景

This skill was born from a real incident: during a large-scale skill refactoring session, an AI assistant rewrote the entire `ai-flow` skill file while only being asked to make 3 targeted changes — silently losing carefully designed constraints, stage descriptions, and workflow logic that had taken weeks to build.

The fix took longer than the original work. That's when we decided: _the update workflow needs its own guardrails_.

这个 skill 源于一次真实事故：在一次大型 skill 重构过程中，AI 助手在只被要求做 3 处定向改动的情况下，将整个 `ai-flow` skill 文件重写了一遍——悄无声息地丢失了耗费数周构建的约束规则、阶段描述和工作流逻辑。修复花费的时间比原始工作还长。于是我们决定：更新工作流需要自己的护栏。

---

## Related / 相关

- [anthropics/skills](https://github.com/anthropics/skills) — Official Claude Code skills
- [skills.sh](https://skills.sh) — Skills directory and installer

---

## License

MIT — Based on [anthropics/skills skill-creator](https://github.com/anthropics/skills/tree/main/skill-creator), with modifications.
