# agent-skills ✂️

**A collection of AI agent skills by [@darian-deng](https://github.com/darian-deng).**

[![skills.sh](https://img.shields.io/badge/install-skills.sh-blue)](https://skills.sh/darian-deng/agent-skills)
[![GitHub](https://img.shields.io/badge/github-darian--deng%2Fagent--skills-black)](https://github.com/darian-deng/agent-skills)

---

## Install / 安装

```bash
# Install all skills (Claude Code + Cursor, global)
npx skills add darian-deng/agent-skills -g --all

# Install a specific skill
npx skills add darian-deng/agent-skills --skill skill-surgeon -g -a claude-code -a cursor
```

---

## Skills

### ✂️ skill-surgeon

**Safe surgical updates to existing AI skill files.**

**AI Skill 文件的外科式安全更新工具。**

| | |
|---|---|
| **Trigger** | update skill, modify skill, edit skill file, fix skill, improve skill, patch skill |
| **Install** | `npx skills add darian-deng/agent-skills --skill skill-surgeon -g` |

#### The Problem / 问题背景

When you ask an AI to update a specific part of a skill file, it often **rewrites the entire file** — silently discarding carefully crafted rules, examples, and constraints.

当你让 AI 修改 skill 文件的某个部分时，它往往会**把整个文件重写一遍**，悄无声息地丢掉精心设计的规则和约束。

#### The Solution / 解决方案

skill-surgeon enforces a **Safe Edit Protocol** on every skill file update:

```
1. Read file     → snapshot before
2. Edit (only!)  → apply the specific change via str_replace
3. Read file     → snapshot after
4. Compare       → list every changed line, flag unintended ones
5. Report        → ✅ clean / ❌ restore + retry
```

**No git. No Python. No external dependencies.** Works everywhere.

#### vs Official skill-creator

| | skill-creator (official) | skill-surgeon (this) |
|---|---|---|
| Create new skills | ✅ Full workflow | ✅ Full workflow |
| Update existing skills | ⚠️ May rewrite entire file | ✅ Edit-only + verified |
| Unintended change detection | ❌ Not built-in | ✅ Built-in |
| Auto-restore on bad edit | ❌ Manual | ✅ Automatic |

> **Rule of thumb**: Use official `skill-creator` for new skills. Use `skill-surgeon` for updates.
>
> **使用原则**：新建 skill → 官方 `skill-creator`；修改已有 skill → `skill-surgeon`。

#### Background / 背景

This skill was born from a real incident: during a large-scale skill refactoring session, an AI rewrote the entire `ai-flow` skill file while only being asked to make 3 targeted changes — silently losing constraints and workflow logic that had taken weeks to build.

此 skill 源于一次真实事故：AI 在只被要求做 3 处改动时，将整个 skill 文件重写，静默丢失了数周构建的约束规则。修复花费的时间比原始工作还长。

---

## License

MIT — skill-surgeon is based on [anthropics/skills skill-creator](https://github.com/anthropics/skills/tree/main/skills/skill-creator), with modifications.
