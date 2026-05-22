---
name: optimize-claude-context
description: >
  Create, audit, optimize, or rebuild a project's Claude Code context layer
  (CLAUDE.md, .claude/rules/, .claude/skills/) following lean context principles.
  Use when: initializing Claude Code for a project ("set up Claude", "init",
  "initialize", no CLAUDE.md exists), auditing existing context ("health check",
  "review my rules", "what's wrong with my CLAUDE.md"), optimizing context
  ("clean this up", "apply improvements", "my CLAUDE.md is too long"), or
  updating a specific rule ("add a rule", "change this rule", "remove this").
  Also use proactively when noticing bloat, wrong-layer content, or
  linter-enforceable rules in a project's CLAUDE.md.
---

# CRITICAL — Read first

- **Decision tree.** CLAUDE.md for every-session facts, path-scoped rules for
  file-triggered context, skills for semantically-triggered knowledge. No
  subdirectory CLAUDE.md. No `@import`.
- **Mechanism selection.** Linter → formatter → hooks → config → only then
  CLAUDE.md/rules/skills. "Could enforce" = technically feasible, not just
  currently configured.
- **Ruthless curation over completeness.** Every line competes for a finite
  context budget. When in doubt, cut.

# optimize-claude-context

Manage a project's Claude Code context layer — `CLAUDE.md`, path-scoped project
rules (`.claude/rules/`), and project-level skills (`.claude/skills/`) — so every
token in the context window earns its place.

## Scope

Three artifacts only:

- `./CLAUDE.md` (root, single file)
- `./.claude/rules/*.md` (path-scoped project rules)
- `./.claude/skills/*/SKILL.md` (semantically triggered project skills)

Do not modify Claude auto memory (`~/.claude/projects/*/memory/`). Do not import
content from `AGENTS.md`, `.cursorrules`, or other non-Claude agent configs — note
their existence but keep them out of scope.

## Dependencies

Check installation via filesystem before starting any workflow — more reliable
than `available_skills` context:

```bash
ls ~/.claude/skills/skill-creator/ 2>/dev/null || ls ~/.agents/skills/skill-creator/ 2>/dev/null
ls ~/.claude/skills/skill-surgeon/ 2>/dev/null || ls ~/.agents/skills/skill-surgeon/ 2>/dev/null
```

| Skill | When needed | Install |
|---|---|---|
| `skill-creator` | Creating a new project-level skill | `npx skills add anthropics/skills --skill skill-creator -g -y` |
| `skill-surgeon` | Updating an existing skill | `npx skills add darian-deng/agent-skills --skill skill-surgeon -g -y` |

If a required skill is missing when you reach the point of needing it: stop,
explain why it's needed, give the install command, wait for confirmation.

## Intent routing

Detect the user's intent before acting. **If ambiguous, ask — never guess.**

| Intent | Signals | Workflow |
|---|---|---|
| **init** | No CLAUDE.md / "初始化" / "从零建立" / "set up Claude" | [init-workflow.md](init-workflow.md) |
| **audit-report** | "出报告" / "帮我看看" / "诊断" / wants diagnosis only, no changes | [audit-report-workflow.md](audit-report-workflow.md) |
| **optimize** | "优化" / "帮我改" / "太长了" / "清理" / wants changes applied | [optimize-workflow.md](optimize-workflow.md) |
| **update-rule** | User brings a specific rule to add, change, or remove | [update-rule-workflow.md](update-rule-workflow.md) |

**Ambiguity examples:** "fix my CLAUDE.md" (audit-report or optimize?), "set up
Claude" with existing CLAUDE.md (init or optimize?). Ask one targeted question
before proceeding.

**Proactive detection:** If you notice bloat, wrong-layer content, or
linter-enforceable rules while working on an unrelated task, briefly flag the
issue and ask whether to run audit-report. Never modify files without
confirmation.

## Shared principles

These apply across all four workflows.

### Decision tree — where content belongs

```
Could linter / formatter / hooks / config enforce this?
├─ Yes → Graduate to toolchain. See Mechanism selection protocol below.
└─ No  → Does Claude need this in EVERY session?
         ├─ Yes → ./CLAUDE.md (root, single file, no subdirectory CLAUDE.md)
         └─ No  → Can a file-path glob trigger it, with low collateral damage?
                  ├─ Yes → ./.claude/rules/<name>.md (with paths: frontmatter)
                  └─ No  → ./.claude/skills/<name>/SKILL.md (semantic trigger)
```

Hard constraints:

- **One root CLAUDE.md only.** Never create CLAUDE.md in subdirectories.
- **No `@import`.** Every `@path` expands into context at launch, defeating lazy
  loading. Flag existing `@import` as improvement candidates.
- **Monorepo isolation via path-scoped rules.** Different sub-packages get
  separate rule files scoped by `paths:`, all inside `./.claude/rules/`.
- **Low collateral damage for path rules.** A path rule fires whenever any
  matching file is touched — regardless of whether the task is relevant to the
  rule. When the glob match is semantically too broad, the content belongs in a
  skill, not a rule.

For detailed layer descriptions, frontmatter format, known `paths:` issues, and
monorepo patterns, read [decision-tree.md](decision-tree.md).

### Mechanism selection protocol

Before writing any rule:

```
Step 1 — Could the project's linter, formatter, hooks, or config enforce it?
         "Could" = technically feasible — regardless of whether the rule exists
         yet. If adding one config line would enforce it, that counts.
         → Yes → Propose "graduation." See optimize-workflow.md § Graduation.
Step 2 — If I remove this line, will Claude make a mistake it wouldn't
         otherwise make?
         → No  → Do not add it. Explain why.
         → Yes → Route through the decision tree above.
```

Read linter/formatter config files **in full — never truncate** before evaluating
Step 1. If no linter/formatter exists, Step 1 is N/A.

Exception: explicitly naming a tool has outsized agent-behavior leverage (e.g.,
"Use `pnpm`, never npm or yarn"). These one-liners earn their place.

### Writing principles

Apply to every line written into CLAUDE.md, rules, or skills. See
[writing-principles.md](writing-principles.md) for the full list with examples.

Quick reference (14 principles):

1. Imperative form.
2. Positive phrasing. Cuts violation rate roughly in half. (Specific
   failure-mode prohibitions with rationale are allowed — see principle 3.)
3. Every prohibition carries rationale + alternative.
4. MUST / IMPORTANT ≤ 3–5 per file.
5. Never write what a linter, formatter, hook, or config file can enforce.
6. Never write what Claude already does correctly unprompted.
7. Domain concepts over file paths.
8. No `@import`.
9. One instruction per bullet.
10. Commands in code fences with exact flags.
11. Use H1/H2/H3 + bold as parsing landmarks. Never H4+.
12. Target 100–150 total lines per CLAUDE.md. 200 is the red line.
13. Litmus test: "If I remove this line, will Claude make a mistake?"
14. Failure-driven rules default to keep during audits.

### Output language

All output follows the user's language. Quotes from existing files stay in their
original language; rationale and commentary use the user's language.
