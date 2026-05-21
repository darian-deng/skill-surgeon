# optimize-claude-context

**Manage your Claude Code context layer — CLAUDE.md, path-scoped rules, and skills — so every token earns its place.**

**管理 Claude Code 的 context 层 — CLAUDE.md、路径作用域规则、skills — 让每一个 token 都物有所值。**

| | |
|---|---|
| **Trigger** | set up Claude, initialize CLAUDE.md, audit rules, optimize context, add/remove/move rule, "my CLAUDE.md is too long" |
| **Install** | `npx skills add darian-deng/agent-skills --skill optimize-claude-context -g -y` |

## What it does / 功能

Two workflows, one skill:

| Workflow | When | What happens |
|---|---|---|
| **audit-refactor** | Init a new project, or optimize an existing one | Subagent deep-explores the codebase → health card → per-item proposals with rationale → diff preview → apply |
| **add-rule** | Add, move, or remove a specific rule | Verifies against codebase (duplicates, conflicts, lint coverage, phantom deps) → pushes back boldly → proposes location + phrasing → diff preview → apply |

## Core principles / 核心原则

- **Decision tree:** every rule goes to exactly one of three layers — `CLAUDE.md` (every session) → `.claude/rules/` (path-triggered) → `.claude/skills/` (semantic-triggered).
- **Mechanism selection:** linter → formatter → hooks → config → only then context layer. If a tool can enforce it, don't write it in CLAUDE.md.
- **Litmus test:** "If I remove this line, will Claude make a mistake it wouldn't otherwise make?" If no, cut it.
- **Line budget:** 100–150 lines target, 200 red line.

## 15 writing principles / 写作原则

1. Imperative form
2. Positive phrasing (failure-mode prohibitions with rationale allowed)
3. Every prohibition carries rationale + alternative
4. MUST/IMPORTANT ≤ 3–5 per file
5. Primacy–recency anchoring (top & bottom `# CRITICAL` sections)
6. Never write what linter/formatter/hooks/config can enforce
7. Never write what Claude already does correctly
8. Domain concepts over file paths
9. No `@import`
10. One instruction per bullet
11. Commands in code fences
12. H1/H2/H3 + bold as landmarks, no H4+
13. 100–150 line target, 200 red line
14. Litmus test on every line
15. Failure-driven rules default to keep

## File structure / 文件结构

```
optimize-claude-context/
├── SKILL.md                              # Main skill file (~130 lines)
└── references/
    ├── writing-principles.md             # 15 principles with examples
    ├── audit-refactor-workflow.md         # 7-phase audit + graduation checklist
    ├── add-rule-workflow.md              # 7-step add/move/remove + graduation
    ├── health-card.md                    # Health card template + thresholds
    └── decision-tree.md                  # 3-layer decision tree + known issues
```

## Sources / 参考来源

Built from synthesis of:

- [Claude Code official best practices](https://code.claude.com/docs/en/best-practices)
- [Claude Code official memory docs](https://code.claude.com/docs/en/memory)
- [claude-md-management plugin](https://claude.com/plugins/claude-md-management) by Anthropic
- [venables/optimize-agents-md](https://github.com/venables/dotfiles/blob/main/agents/.agents/skills/optimize-agents-md/SKILL.md) — lean writing philosophy, compliance research, token budget framework
