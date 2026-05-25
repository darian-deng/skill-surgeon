# optimize-claude-context — Implementation Plan

## Dependency Graph

```
Phase 1 (no dependencies — write in parallel):
  directive.md
  linter-capabilities.md
  writing-formats.md   ← merges content from writing-principles.md + health-card.md
                          plus adds format specs per layer

Phase 2 (depends on Phase 1):
  decision-tree.md  ← directive.md + linter-capabilities.md

Phase 3 (depends on Phase 1 + 2):
  handle-one-directive.md  ← decision-tree.md + writing-formats.md + linter-capabilities.md

Phase 4 (depends on Phase 3 — write in parallel):
  rebuild-workflow.md  ← handle-one-directive.md
  audit-workflow.md    ← handle-one-directive.md + (health-card.md scoring logic)

Phase 5 (depends on Phase 4):
  SKILL.md rewrite  ← all reference files finalized

Phase 6:
  Delete old files
  Update README.md

Phase 7:
  Consistency verification pass
  Commit + push
```

---

## Phase 1 — Foundation (3 files, independent, write in parallel)

### 1a. `references/directive.md` (NEW)

Content:
- Directive definition + atomic test example (✅/❌ examples)
- Scope definition and scope-determination rules for monorepo
- Four layers table with role and loading behavior
- Layer selection rules (multi-step procedure wins over all)
- Mutual exclusivity constraint with examples

Acceptance: an agent reading only this file can correctly identify whether a piece
of text is one or multiple directives, assign scope, and know which layer it belongs to.

---

### 1b. `references/linter-capabilities.md` (NEW)

Content:
- The correct question format: "write exact config + cite official docs URL"
- For rules lacking individual URLs: parent page URL is acceptable
- ESLint mechanisms with examples:
  - `no-restricted-syntax` + AST selectors (TryStatement, ForInStatement, etc.)
  - `no-restricted-imports` (enforce import alternatives)
  - `no-restricted-globals` (ban global access)
  - `no-restricted-properties` (ban property access)
  - `@typescript-eslint/*` rules
- TypeScript compiler options (noImplicitAny, strict, strictNullChecks, etc.)
  - Docs: https://www.typescriptlang.org/tsconfig — parent page is acceptable citation
- Ruff rules (Python) with example patterns
- Pre-commit hook capabilities (file naming, pattern bans via grep hooks)
- Verification: run linter against code containing the pattern; confirm error is produced

Acceptance: an agent reading this file cannot miss common enforcement opportunities
(try-catch ban, import restrictions, global bans). The "write config + cite docs"
methodology prevents false "not enforceable" judgments.

---

### 1c. `references/writing-formats.md` (NEW — replaces writing-principles.md + health-card.md)

Content (four sections):

**CLAUDE.md format:**
- 14 writing principles (full list with examples, migrated from writing-principles.md)
- Line budget: target 100-150, hard limit 200
- H1/H2/H3 only, commands in code fences
- One instruction per bullet

**Path rule format:**
```yaml
---
paths:
  - "src/api/**/*.ts"
---
```
- Content: lookup/reference, not workflow instructions
- One file per domain (auth.md, api.md, etc.)
- Known `paths:` issue: YAML list may only match first pattern in some versions;
  use CSV format as fallback: `paths: "glob1,glob2"`

**Skill format:**
```yaml
---
name: <kebab-case>
description: <≤15 words>
---
```
- description hard limit: 15 words; verified via skill-creator eval loop
- Body: step-by-step, imperative form
- handle-one-directive writes stub only (frontmatter); skill-creator develops body

**ADR format (Nygard template):**
```yaml
---
adr: NNNN
title: <short title>
status: Accepted
date: YYYY-MM-DD
---
## Context
## Decision
## Consequences
## Alternatives Considered
```
- Consequences must not be empty (quality gate)
- Location: `<project-root>/docs/adrs/` always

**Health card section** (migrated from health-card.md):
- Used by audit-workflow.md for scoring per-directive
- Template for CLAUDE.md health: volume metrics, separation of concerns,
  structure checks, content quality checks

Acceptance: an agent can format any directive correctly for any target layer.

---

## Phase 2 — Decision tree

### 2a. `references/decision-tree.md` (REWRITE)

Replace all existing content. New content:
- Step 0: parse (atomic check, scope determination)
- Step 1: linter check (reference linter-capabilities.md for exact question format)
- Step 2: existing context layer check (scan `.claude/rules/*.md`,
  `.claude/skills/*/SKILL.md`, `./CLAUDE.md` — project-level only)
- Step 3: layer routing with explicit tiebreaker (multi-step procedure wins over all)
- Confidence calibration: H/M/L with concrete criteria
- Scope + monorepo routing rules
- Reverse-layer detection table (for audit use)

Acceptance: an agent reading only decision-tree.md and linter-capabilities.md can
correctly route any directive with consistent confidence levels.

---

## Phase 3 — Core workflow

### 3a. `references/handle-one-directive.md` (NEW — replaces update-rule-workflow.md)

Content:
- Three modes: manual, feat-flow, rebuild-execute — and where each starts in the flow
- feat-flow contract: directive is pre-parsed, scope is known, starts at Step 1
- rebuild-execute contract: decision is pre-made (from table), starts at Step 4
- Split handling: recursively call self; caller receives all results before continuing
- Linter graduation: modify linter config directly or create new config file
- Conflict surface: what to show user and in what format
- Content enrichment: when to use code-explorer (project-related), when to research
  (external knowledge)
- Skill stub writing: only frontmatter, notify user to use skill-creator
- Mode-specific write behavior: manual/feat-flow → file; rebuild-execute → file + table update
- Verification checklist (post-write)
- Global CLAUDE.md note: never modified by handle-one-directive

Acceptance: an agent can execute the complete flow for any directive in any mode
without additional guidance.

---

## Phase 4 — Higher-level workflows (parallel)

### 4a. `references/rebuild-workflow.md` (REWRITE — replaces init-workflow.md)

Content (8 phases):
- Phase 0: dependency check (filesystem + available_skills fallback)
- Phase 1: backup (exact paths, idempotency — abort if backup exists)
- Phase 2: collection (Subagent A non-ADR sequential, Subagent B ADR)
  - Global CLAUDE.md handling: collect project-specific, never modify global
  - Dynamic file batching
- Phase 3: domain grouping + stale check + conflict marking
  - Domain definition and split-vs-merge criteria
  - Stale grep methodology
- Phase 4: decision tree evaluation (NOT handle-one-directive; subagents apply decision
  tree directly and write results to table; 3-4 per domain batch with code-explorer reuse)
- Phase 5: user alignment (CONFLICT rows required; L rows recommended)
- Phase 6: execution (handle-one-directive rebuild-execute mode; domain-grouped batches)
- Phase 7: ADR processing (after Phase 6; 4 checks)
- Phase 8: final verification + cleanup (global CLAUDE.md migration note to user)

Full rebuild-progress.md table format with all column specs.
Status state machine: pending → in-progress → done / deprecated / conflict-blocked.

Acceptance: rebuild can be executed end-to-end by following this document alone.

---

### 4b. `references/audit-workflow.md` (NEW — replaces audit-report-workflow.md)

Content:
- Clarification: audit writes `.claude/audit-report.md` only; no context layer files modified
- Scan scope: project-level `.claude/` only
- Per-directive evaluation approach
- Linter graduation check (linter-capabilities.md methodology)
- Stale grep (same methodology as rebuild)
- ADR validation checks
- Scoring table (from design doc Section 6.1)
- Category score calculation
- Output format: `.claude/audit-report.md` template
- How to act on findings: call handle-one-directive for each chosen item

Acceptance: audit produces a scored, prioritized report with clearly actionable items.

---

## Phase 5 — SKILL.md rewrite

### 5a. `SKILL.md` (REWRITE)

New content:
- description: updated to cover handle-one-directive / rebuild / audit
  with all trigger signals (see Section 7.2 of design doc for signal list)
- CRITICAL Read first: 3-line summary of decision tree
- Scope: 3 artifacts only (CLAUDE.md, rules, skills)
- Dependencies: filesystem check for skill-creator + skill-surgeon
- Intent routing table (3 commands with all signals from design doc 7.2)
- Shared principles: pointers to reference files
- Output language note

Target: ≤120 lines. All detail in references/.

---

## Phase 6 — Cleanup

### Files to delete:
- `references/init-workflow.md`
- `references/audit-report-workflow.md`
- `references/optimize-workflow.md`
- `references/update-rule-workflow.md`
- `references/writing-principles.md` (content merged into writing-formats.md)
- `references/health-card.md` (content merged into writing-formats.md + audit-workflow.md)

### `README.md` update (skills/optimize-claude-context/README.md):
Update skill description and command list.

### Repo root `README.md`:
Update the optimize-claude-context table entry.

---

## Phase 7 — Consistency verification + commit

### Verification gate (before commit):

Read all 7 new reference files. Verify:
1. directive.md definitions match decision-tree.md routing criteria
2. linter-capabilities.md question format matches handle-one-directive.md Step 1
3. writing-formats.md constraints match handle-one-directive.md Step 5 spec
4. rebuild-workflow.md Phase 4 explicitly states it does NOT call handle-one-directive
5. rebuild-workflow.md Phase 6 explicitly states it calls handle-one-directive
   in rebuild-execute mode
6. audit-workflow.md references writing-formats.md health card section for metrics
7. SKILL.md signals cover all three commands without gaps

### Commit message:
```
refactor(optimize-claude-context): rebuild around directive model

Core concept: directive (atomic, scoped knowledge point)
Decision tree: linter-first → multi-step→skill → CLAUDE.md → rule
handle-one-directive: 3 modes (manual, feat-flow, rebuild-execute)
rebuild: table-driven, domain-grouped, phase 4 = evaluate, phase 6 = execute
audit: 0-100 scored health check, no context writes, leads to handle-one-directive
Removed: init, writing-principles.md, health-card.md (merged into writing-formats.md)
```

---

## Parallel execution opportunities

- Phase 1 (all 3 foundation files): fully parallel
- Phase 4 (rebuild-workflow.md and audit-workflow.md): fully parallel once Phase 3 done
- Phase 6 (all deletions): fully parallel

## Sequencing constraints

- Phase 3 must be complete before Phase 4
- Phase 4 must be complete before Phase 5 (SKILL.md references all files)
- Phase 6 (deletions) must happen after Phase 5 to avoid broken references during writing

## Risk items

1. **writing-formats.md** is the most referenced file — if health card content is
   incomplete when merged from health-card.md, audit-workflow.md Phase 6.1 scoring
   will have gaps. Solution: verify audit-workflow.md can compute all scores using
   only what's in writing-formats.md.

2. **handle-one-directive.md** rebuild-execute vs Phase 4 distinction — this is the
   most subtle design point. Implementer must be explicit that Phase 4 = evaluate
   (table only) and Phase 6 = execute (files + table status update).

3. **Domain grouping in rebuild Phase 3** remains a judgment call. Implementer should
   document that domain is a batching concept (efficiency), not a correctness constraint.
   Over-splitting is always safe.
