---
name: optimize-claude-context
description: >
  Create, audit, or restructure CLAUDE.md, .claude/rules/, and .claude/skills/
  following lean context-optimization principles. Use this skill whenever the
  user asks to set up Claude Code for a project, initialize CLAUDE.md, audit or
  clean up existing CLAUDE.md or rules, add/remove/move a rule, says "my
  CLAUDE.md is too long", "optimize my context", "set this up for Claude",
  "review my rules", or mentions CLAUDE.md maintenance. Also use proactively
  when you notice an existing CLAUDE.md is bloated, repetitive, contains
  linter-enforceable content, or has rules in the wrong layer.
---

# CRITICAL — Read first

- **Decision tree.** CLAUDE.md for every-session facts, path-scoped rules for
  file-triggered context, skills for semantically-triggered knowledge. No
  subdirectory CLAUDE.md. No `@import`.
- **Mechanism selection.** Linter → formatter → hooks → config → only then
  CLAUDE.md/rules/skills. Litmus test: "If I remove this line, will Claude make
  a mistake it wouldn't otherwise make?"
- **Ruthless curation over completeness.** Every line competes for a finite
  context budget. When in doubt, cut.

# optimize-claude-context

Manage a project's Claude Code context layer — `CLAUDE.md`, path-scoped project
rules (`.claude/rules/`), and skills (`.claude/skills/`) — so every token in the
context window earns its place.

## Scope

This skill manages three artifacts only:

- `./CLAUDE.md` (root, single file)
- `./.claude/rules/*.md` (path-scoped project rules)
- `./.claude/skills/*/SKILL.md` (semantically triggered skills)

Do not read, modify, or propose changes to Claude auto memory
(`~/.claude/projects/*/memory/`). Do not import or merge content from
`AGENTS.md`, `.cursorrules`, or other non-Claude agent configs — note their
existence during audit but keep them out of scope.

## Dependencies

This skill delegates to two external skills at specific points. Check whether
they appear in your `available_skills` before starting either workflow:

| Skill | Purpose | Install |
|---|---|---|
| `skill-creator` | Full creation workflow for new skills (eval loop, description optimization) | `npx skills add anthropics/skills --skill skill-creator -g -y` |
| `skill-surgeon` | Safe surgical updates to existing skills (Edit-only, before/after verified) | `npx skills add darian-deng/agent-skills --skill skill-surgeon -g -y` |

**If a required skill is missing:** stop, explain what the skill does and why
it is needed at this point, give the install command, and wait for the user to
confirm installation before continuing. Do not proceed without it — skipping
the dependency produces lower-quality output without the user realizing it.

## Core philosophy

Context is the scarcest resource in an AI coding session. Every line loaded into
the context window competes with your conversation, file reads, and command
outputs. A focused 100-line file Claude actually follows beats a 300-line file it
mostly ignores.

**The litmus test for every line:** _"If I remove this line, will Claude make a
mistake it wouldn't otherwise make?"_ If no, cut it.

## Where content belongs — the decision tree

```
Does Claude need this in EVERY session?
├─ Yes → ./CLAUDE.md (root, single file, no subdirectory CLAUDE.md)
└─ No  → Can a file-path glob trigger it, with low collateral damage?
         ├─ Yes → ./.claude/rules/<name>.md (with paths: frontmatter)
         └─ No  → ./.claude/skills/<name>/SKILL.md (semantic trigger)
```

Hard constraints:

- **One root CLAUDE.md only.** Never create CLAUDE.md in subdirectories.
- **No `@import`.** Every `@path` expands into context at launch, defeating lazy
  loading. Flag existing `@import` as improvement candidates.
- **Monorepo isolation via paths-scoped rules.** Different sub-packages get
  separate rule files scoped by `paths:`, all inside `./.claude/rules/`.
- **Low collateral damage for path rules.** A path rule fires whenever any
  matching file is touched — regardless of whether the task is actually relevant
  to the rule. When the glob match is semantically too broad (e.g., a rule about
  "add i18n keys when adding user-facing text" scoped to `**/*.tsx` fires on
  every tsx change, even unrelated refactors), the content belongs in a skill,
  not a rule.

For detailed layer descriptions, frontmatter format, known `paths:` issues, and
monorepo patterns, read
[references/decision-tree.md](references/decision-tree.md).

## Mechanism selection protocol

Before writing any rule, run this filter chain:

```
Step 1 — Can the project's linter, formatter, hooks, or config files enforce it?
         → Yes → Propose "graduation." See graduation checklist in
                 references/audit-refactor-workflow.md § Graduation.
Step 2 — If I remove this line, will Claude make a mistake it wouldn't
         otherwise make?
         → No  → Do not add it. Explain why to the user.
         → Yes → Route through the decision tree above.
```

This protocol is language-agnostic. Inspect the project's actual config to
determine coverage. If no linter/formatter exists, Step 1 is N/A — skip to
Step 2.

Exception: explicitly naming a tool has outsized agent-behavior leverage (e.g.,
"Use `pnpm`, never npm or yarn" — even when `pnpm-lock.yaml` exists). These
one-liners earn their place.

## Two workflows

Detect which workflow the user needs from their prompt.

**Dependency check (run once before starting either workflow):** Check whether
`skill-creator` and `skill-surgeon` appear in your `available_skills`. If
either is missing, follow the guidance in the Dependencies section above before
proceeding.

**Proactive detection:** If you notice bloat, wrong-layer content, or
linter-enforceable rules in a project's CLAUDE.md while working on an unrelated
task, briefly flag the issue and ask whether to run audit-refactor. Do not modify
files without user confirmation.

### Workflow 1 — audit-refactor

Triggered by: initializing a new project, optimizing existing context, "my
CLAUDE.md is too long", or proactive detection of bloat.

Read [references/audit-refactor-workflow.md](references/audit-refactor-workflow.md)
for the full step-by-step procedure (7 phases).

### Workflow 2 — add-rule (also covers move and remove)

Triggered by: user brings a specific rule to add, move, or remove.

Read [references/add-rule-workflow.md](references/add-rule-workflow.md) for the
full step-by-step procedure (7 steps).

## Writing principles

Apply these to every line written into CLAUDE.md, rules, or skills. See
[references/writing-principles.md](references/writing-principles.md) for the
full list with examples.

Quick reference (15 principles):

1. Imperative form. `Use named exports.` not `It's preferred that...`
2. Positive phrasing. Cuts violation rate roughly in half. (Specific
   failure-mode prohibitions with rationale are allowed — see principle 3.)
3. Every prohibition carries rationale + alternative.
4. MUST / IMPORTANT ≤ 3–5 per file.
5. Primacy–recency anchoring: mirror the 3 most critical rules at top and
   bottom of every CLAUDE.md.
6. Never write what a linter, formatter, hook, or config file can enforce.
7. Never write what Claude already does correctly unprompted.
8. Domain concepts over file paths.
9. No `@import`.
10. One instruction per bullet.
11. Commands in code fences with exact flags.
12. Use H1/H2/H3 + bold as parsing landmarks. Never H4+.
13. Target 100–150 total lines per CLAUDE.md. 200 is the red line.
14. Litmus test: "If I remove this line, will Claude make a mistake?"
15. Failure-driven rules default to keep during audits.

## Reference navigation

| When you need to… | Read |
|---|---|
| Run a full audit | audit-refactor-workflow.md + health-card.md + writing-principles.md |
| Add, move, or remove a rule | add-rule-workflow.md + decision-tree.md + writing-principles.md |
| Decide which layer (CLAUDE.md / rule / skill) | decision-tree.md |
| Graduate a rule to linter/hook | audit-refactor-workflow.md § Graduation |
| Check or produce a health card | health-card.md |

## Output language

All output — health cards, proposals, diff previews, explanations, and the
content written into CLAUDE.md / rules / skills — follows the user's language.
Quotes from existing files stay in their original language; rationale and
commentary use the user's language.

# CRITICAL — Read last

- **Decision tree.** CLAUDE.md for every-session facts, path-scoped rules for
  file-triggered context, skills for semantically-triggered knowledge. No
  subdirectory CLAUDE.md. No `@import`.
- **Mechanism selection.** Linter → formatter → hooks → config → only then
  CLAUDE.md/rules/skills. Litmus test: "If I remove this line, will Claude make
  a mistake it wouldn't otherwise make?"
- **Ruthless curation over completeness.** Every line competes for a finite
  context budget. When in doubt, cut.
