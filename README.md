# agent-skills ✂️

**A collection of AI agent skills by [@darian-deng](https://github.com/darian-deng).**

[![skills.sh](https://img.shields.io/badge/install-skills.sh-blue)](https://skills.sh/darian-deng/agent-skills)
[![GitHub](https://img.shields.io/badge/github-darian--deng%2Fagent--skills-black)](https://github.com/darian-deng/agent-skills)

---

## Install / 安装

```bash
# Install all skills (Claude Code + Cursor, global)
npx skills add darian-deng/agent-skills -g --all -y

# Install a specific skill
npx skills add darian-deng/agent-skills --skill <skill-name> -g -y
```

---

## Skills

### 🧭 grounded-design

**把一句话需求拆解成一份接地的详细技术方案 —— 先查清能查的,只问开发者真正只有他知道的。**

**Turn a one-line requirement into a grounded technical design — investigate what's knowable first, ask only what only the developer knows.**

| | |
|---|---|
| **Trigger** | 需求拆解, 技术方案, 把需求变成设计, scope the work, grill my plan, turn requirement into design |
| **Install** | `npx skills add darian-deng/agent-skills --skill grounded-design -g -y` |

#### The Problem / 问题背景

通用的审讯式提问 skill(如 grill-me)会**凭训练数据印象编选项**:面对「CLI 事件 sessionId 怎么填」,它列出「读 last_session_id」这种选项——而代码里早已做好多 session 保护,这个选项压根不该存在。根因是「能查代码就查」只是一句可选的、被动的提示,模型跳过了。

A generic interrogation skill fabricates options from training-data impressions instead of reading the code first — surfacing choices that don't exist because the codebase already settled them.

#### The Solution / 解决方案

- **接地优先(硬规则)**:提第一个问题前,必须先输出一份带来源(`file:line`)的《现状理解》,没有它不得提问 —— 把「先探索」从训诫变成可见的事前 gate。
- **三分类**:每个待定点归为 `finding`(查清的事实,陈述不提问)/ `问开发者`(只有他知道的意图)/ `假设`(接不了地的空白,标默认值 + 猜错后果)。不用置信度数字。
- **铁律**:事实型 / 技术型的选项或推荐,凭印象就禁止呈现;纯产品意图问题用开放式提问、不编选项。
- **选择性呈现**:finding 只在决策相关 / 可争议 / 带风险时拿出,其余静默记录,不淹没开发者。

通用 skill,不绑定产物格式 —— 产物格式由调用方指定。

---

### 🎯 optimize-claude-context

**Manage your Claude Code context layer — CLAUDE.md, rules, and skills — so every token earns its place.**

**管理 Claude Code 的 context 层 — CLAUDE.md、规则、skills — 让每一个 token 都物有所值。**

| | |
|---|---|
| **Trigger** | set up Claude, initialize CLAUDE.md, audit rules, optimize context, add/remove/move rule, "my CLAUDE.md is too long" |
| **Install** | `npx skills add darian-deng/agent-skills --skill optimize-claude-context -g -y` |

Two workflows in one skill:

- **audit-refactor** — Subagent deep-explores the codebase → health card → per-item proposals → diff preview → apply. Works for both new projects and existing bloated configs.
- **add-rule** — Verifies against codebase (duplicates, conflicts, lint coverage, phantom deps) → pushes back boldly → proposes location + phrasing → diff preview → apply.

Core ideas: decision tree (CLAUDE.md → path-scoped rules → skills), mechanism selection (linter/hooks first, context layer last), 15 writing principles, 100–150 line budget.

→ [Full README](skills/optimize-claude-context/README.md)

---

### ✂️ skill-surgeon

**Safe surgical updates to existing AI skill files.**

**AI Skill 文件的外科式安全更新工具。**

| | |
|---|---|
| **Trigger** | update skill, modify skill, edit skill file, fix skill, improve skill, patch skill |
| **Install** | `npx skills add darian-deng/agent-skills --skill skill-surgeon -g -y` |

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

### 📋 adr-manage

**Manage Architecture Decision Records (ADR) for your project — create, supersede, index, list, search, and bootstrap from scratch.**

**管理项目的架构决策记录（ADR）— 新建、替代（supersede）、索引、列出、搜索、从零初始化。**

| | |
|---|---|
| **Trigger** | 新加 ADR, record a decision, supersede ADR, 初始化 ADR, bootstrap ADR, 重建索引, rebuild index, 列出 ADR, list ADR, 搜索 ADR, search ADR |
| **Install** | `npx skills add darian-deng/agent-skills --skill adr-manage -g -y` |

#### What it does / 功能

Six sub-capabilities behind a single intent router:

通过意图路由统一入口，包含六个子能力：

| Sub-capability | Description | 说明 |
|---|---|---|
| **new** | Create a new ADR with auto-numbering (4-digit zero-padded) and Nygard template | 自动编号（四位零填充）+ Nygard 模板创建新 ADR |
| **supersede** | Replace an old ADR with a new one — bidirectional links maintained | 用新 ADR 替代旧 ADR，自动维护双向链接 |
| **index** | Rebuild `docs/adr/README.md` index table from ADR files | 从 ADR 文件重建索引表 |
| **list** | List ADRs, optionally filtered by status | 按状态列出 ADR |
| **grep** | Search ADR content by keyword | 按关键词搜索 ADR 内容 |
| **bootstrap** | Initialize ADR system from scratch (directory + meta ADR + README) | 从零初始化 ADR 体系（目录 + meta ADR + README） |

#### Design principles / 设计理念

- **AI writes content, skill enforces structure** — numbering, templates, index, supersede links are all handled mechanically. AI focuses on the *why / what / consequences* of the decision.
- **AI 负责内容，skill 负责结构** — 编号、模板、索引、supersede 链接全部机械化处理。AI 专注于决策的 *why / what / consequences*。
- **Negative consequences cannot be empty** — a quality gate that forces thorough trade-off analysis.
- **Negative consequences 不允许为空** — 强制进行充分的 trade-off 分析。
- **ADRs are never deleted** — only superseded or deprecated, preserving full decision history.
- **ADR 不删除** — 只能 supersede 或 deprecate，保留完整决策历史。

---

## License

MIT — skill-surgeon is based on [anthropics/skills skill-creator](https://github.com/anthropics/skills/tree/main/skills/skill-creator), with modifications.
