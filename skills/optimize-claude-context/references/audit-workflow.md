# Audit Workflow

Evaluates the health of the current context layer. Produces a scored, prioritized
report at `.claude/audit-report.md`.

**Scope:** audit writes `.claude/audit-report.md` only. It does NOT modify any
context layer file. After the report, each improvement is pursued separately via
`handle-one-directive`.

References:
- `decision-tree.md` — for per-directive layer check
- `linter-capabilities.md` — for linter graduation check methodology
- `writing-formats.md §Health Card` — for scoring metrics and thresholds

---

## Scan Scope

Audit scans project-level artifacts only:
- `./CLAUDE.md`
- `./.claude/rules/*.md`
- `./.claude/skills/*/SKILL.md` (project-level only)
- All ADR directories discovered dynamically:
  ```bash
  find . -type d \( -name "adr" -o -name "adrs" -o -name "decisions" \) \
    -not -path "*/node_modules/*" -not -path "*/.git/*"
  ```
- `~/.claude/CLAUDE.md` (global — read-only; check for project-specific directives)

---

## Execution Steps

Deduction values for all finding type codes are defined in `writing-formats.md
§Health Card — Scoring` (the single authoritative source). Each step below produces
finding type codes only; do not define or redefine deduction values here.

### Step 1 — File inventory

```bash
wc -l ./CLAUDE.md
find .claude/rules -name "*.md" | sort
find .claude/skills -name "SKILL.md" | sort
# Dynamic ADR discovery — same pattern as rebuild Phase 0
find . -type d \( -name "adr" -o -name "adrs" -o -name "decisions" \) \
  -not -path "*/node_modules/*" -not -path "*/.git/*" \
  | xargs -I{} find {} -name "*.md" | sort 2>/dev/null
```

Record file count and line counts.

### Step 2 — Per-directive layer check

For each directive in each file, apply the decision tree (Steps 0-3 from
`decision-tree.md`) **without writing anything**. Compare the actual layer to
the decision tree result.

| Actual vs. expected | Finding type |
|---|---|
| Matches decision tree | — |
| In wrong layer | `WRONG_LAYER` |
| Skill description > 15 words | `DESC_TOO_LONG` |
| Rule missing `paths:` frontmatter | `MISSING_PATHS` |
| Path rule missing `when:` field | `MISSING_WHEN` |
| ADR with empty Consequences | `INCOMPLETE_ADR` |
| Stub skill never completed | `STUB_INCOMPLETE` |

**`MISSING_WHEN` judgment:** a path-rule without `when:` is flagged regardless of glob
width. A broad package-root glob (`apps/X/**`) is NOT a problem — the issue is the
absence of purpose documentation. If a path-rule intentionally covers a wide scope, the
`when:` makes that intent explicit and lets Claude self-filter.

### Step 3 — Linter graduation check

For each behavioral directive in CLAUDE.md and path rules:

Ask: "If this directive CAN be enforced by [linter], write the exact rule
configuration and cite the official documentation URL."

See `linter-capabilities.md` for enforcement mechanisms. Do not count as a
graduation opportunity if you cannot produce a valid config + citation.

| Finding | Finding type |
|---|---|
| Linter-enforceable directive in context layer | `LINTER_GRADUATION` |
| `@import` reference in any context file | `IMPORT_REF` |

### Step 4 — Stale check

For each directive that references file paths or tool names, grep the codebase:

```bash
# File path check
find . -path "<referenced-path>" 2>/dev/null | head -3

# Tool/library name check
grep -r "<tool-name>" . --include="package.json" --include="*.toml" \
  --include="*.yaml" -l 2>/dev/null | head -3
```

| Finding | Finding type |
|---|---|
| Referenced path not found | `STALE` |
| Referenced tool name not found | `STALE` |

### Step 5 — ADR validation

For each ADR in the directories discovered in Step 1:

1. **Superseded orphan check**: does the ADR's `status` field contain `Superseded by`?
   If yes → `ADR_SUPERSEDED_ORPHAN`. Record the finding; do not run checks 2-5 for this ADR.
   (Auto-removal happens during rebuild Phase 7 Check 0, not during audit.)

2. **Definition check**: does it meet the ADR definition from `writing-formats.md §ADR definition`?
   All three must be true: (a) hard to reverse, (b) contrary to common practice, (c) has
   a concrete rejected alternative. If any fails → `ADR_INVALID`. Note: condition (b) is
   low-confidence for ADRs that align with industry practice — flag as M confidence, do not
   auto-recommend deletion.

3. **Duplicate check**: is the content already covered by a CLAUDE.md or path rule?
   If yes → `ADR_DUPLICATE`.

4. **Code comment check**: could it be replaced by a short inline comment at the relevant
   code location? If yes → `ADR_REPLACEABLE`.

5. **Cross-ADR consolidation check** (run once after processing all individual ADRs):
   read the title of every valid ADR (not flagged by checks 1-4). Group titles by decision
   domain. For each group of 2+ ADRs with no formal supersede relationship → `ADR_CONSOLIDATION`
   (report the group together, not as individual findings).

| Finding | Finding type |
|---|---|
| Superseded ADR still in active directory | `ADR_SUPERSEDED_ORPHAN` |
| ADR fails definition check | `ADR_INVALID` |
| ADR content duplicated in CLAUDE.md | `ADR_DUPLICATE` |
| ADR replaceable by code comment | `ADR_REPLACEABLE` |
| Multiple ADRs covering same decision domain | `ADR_CONSOLIDATION` |

### Step 6 — Global CLAUDE.md check

Read `~/.claude/CLAUDE.md`. For each directive found: is it project-specific (only
meaningful in the context of this project)?

| Finding | Finding type |
|---|---|
| Project-specific directive in global CLAUDE.md | `GLOBAL_MISPLACED` |

---

## Scoring

See `writing-formats.md §Health Card — Scoring` for the authoritative scoring table:
category definitions, per-finding deduction values, and score calculation formula.

**Finding → category mapping** (maps finding type codes defined above to categories):

| Finding type | Category |
|---|---|
| `WRONG_LAYER`, `STUB_INCOMPLETE` | Layer compliance |
| `LINTER_GRADUATION`, `IMPORT_REF`, `GLOBAL_MISPLACED` | Toolchain efficiency |
| `STALE` | Content freshness |
| `INCOMPLETE_ADR`, `ADR_INVALID`, `ADR_DUPLICATE`, `ADR_REPLACEABLE`, `ADR_SUPERSEDED_ORPHAN`, `ADR_CONSOLIDATION`, `MISSING_PATHS`, `MISSING_WHEN`, `DESC_TOO_LONG` | Format compliance |

---

## Output: `.claude/audit-report.md`

```markdown
## Context Layer Audit — <project name> — <date>

### Score: <total>/100

| Category | Score |
|---|---|
| Layer compliance | <N>/25 |
| Toolchain efficiency | <N>/25 |
| Content freshness | <N>/25 |
| Format compliance | <N>/25 |

### Priority Actions (high impact first, deduction from writing-formats.md §Scoring)

[1] LINTER_GRADUATION (-5): "<directive text>" can be enforced by ESLint
    `no-restricted-syntax: [{selector: "TryStatement", message: "..."}]`
    Docs: https://eslint.org/docs/latest/rules/no-restricted-syntax
    File: CLAUDE.md:34

[2] WRONG_LAYER (-5): "<directive text>" is in a path rule but is a multi-step
    procedure — route to skill instead.
    File: .claude/rules/logging.md:12-18

[3] STALE (-5): .claude/rules/auth.md references src/services/auth/oldModule.ts
    — file not found by find.
    File: .claude/rules/auth.md:8

[4] MISSING_PATHS (-5): .claude/rules/api.md has no paths: frontmatter — loads
    unconditionally, consuming every-session budget.
    File: .claude/rules/api.md:1

[5] GLOBAL_MISPLACED (-5): ~/.claude/CLAUDE.md line 23 contains a project-specific
    directive ("Use pnpm for this project") — move to ./CLAUDE.md.

[6] ADR_INVALID (-3): docs/adrs/0003-use-react.md explains an obvious choice with
    no trade-off analysis — does not meet ADR definition.
    File: docs/adrs/0003-use-react.md

### Files Scanned

| File | Lines | Status |
|---|---|---|
| ./CLAUDE.md | <N> | <within budget / over budget> |
| .claude/rules/api.md | <N> | <paths: present / missing> |
| .claude/skills/deploy/SKILL.md | <N> | <stub / complete> |
| docs/adrs/0001-use-pnpm.md | <N> | valid |

### How to act

Call handle-one-directive for each item you want to improve. Example:

> "The auth rule is stale — references src/services/auth/oldModule.ts which no
> longer exists."

handle-one-directive will route the updated directive to the correct layer.

To fix a linter graduation opportunity, provide the config + citation to
handle-one-directive, which will modify the linter config directly.
```

---

## Post-audit: Acting on Findings

Audit produces findings only — it does not fix them. Each finding becomes input to
`handle-one-directive`:

| Finding | What to tell handle-one-directive |
|---|---|
| `WRONG_LAYER` | "Move this directive to the correct layer: [paste directive text]" |
| `LINTER_GRADUATION` | "Graduate this to the linter: [paste config + citation]" |
| `STALE` | "Update this stale directive: [paste updated version]" or "Remove this stale directive: [paste text]" |
| `MISSING_PATHS` | "Add paths: frontmatter to this rule: [paste rule content]" |
| `MISSING_WHEN` | "Add when: field to this path rule: [paste rule file path]" |
| `GLOBAL_MISPLACED` | "Move this from global CLAUDE.md to project: [paste text]" |
| `INCOMPLETE_ADR` | "Add Consequences section to ADR: [paste ADR path]" |
| `STUB_INCOMPLETE` | "Complete this skill stub using skill-creator: [paste skill name]" |
| `DESC_TOO_LONG` | "Shorten this skill description to ≤15 words: [paste current description]" |
| `IMPORT_REF` | "Remove @import from CLAUDE.md and inline or migrate: [paste @import line]" |
| `ADR_SUPERSEDED_ORPHAN` | "Clean up this superseded ADR: run rebuild (Phase 7 auto-handles) or manually move `<adr-path>` to `<adr-dir>/deprecated/`." |
| `ADR_INVALID` | "Deprecate this ADR — it does not meet the ADR definition: [paste ADR path]" |
| `ADR_DUPLICATE` | "Deprecate this ADR — content already covered by CLAUDE.md directive: [paste both]" |
| `ADR_REPLACEABLE` | "Replace this ADR with a code comment: [paste suggested comment]" |
| `ADR_CONSOLIDATION` | "Consolidate these ADRs into one: [paste group titles and paths]" — handle manually or via handle-one-directive after merging content. |

One call per finding. handle-one-directive runs the full decision tree and writes
the fix.
