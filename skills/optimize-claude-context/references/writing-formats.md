# Writing Formats

Format specifications for each context layer artifact. An agent reading only this
file can correctly format any directive for any target layer.

---

## CLAUDE.md Format

### 14 Writing Principles

**1. Imperative form**

Write direct commands, not passive preferences.

- ✅ `Use named exports exclusively.`
- ❌ `It's preferred that default exports be avoided.`

**2. Positive phrasing**

State what to do, not what to avoid. Flipping negatives to positives cuts rule
violations roughly in half.

- ✅ `Use the project's structured logger for all runtime logging.`
- ❌ `Never use console.log.`

Exception: specific negatives describing a concrete failure mode Claude has hit are
acceptable — e.g., "Do not add event handlers when the framework manages reactivity."
The test is specificity: vague negatives are the problem, not negatives in general.
When keeping a negative, always include rationale and an alternative (principle 3).

**3. Every prohibition carries rationale + alternative**

Lets Claude generalize to related situations not explicitly covered.

- ✅ `Never force-push to shared branches — it rewrites history other collaborators
  depend on. Use --force-with-lease on personal branches only.`
- ❌ `Never force-push.`

Do not forcibly rewrite specific failure-mode prohibitions to positive form. A
well-justified prohibition with rationale is more informative than a vague positive
restatement.

**4. MUST / IMPORTANT ≤ 3–5 per file**

Reserve emphasis for rules that truly cannot be broken. If everything is IMPORTANT,
nothing is.

**5. Never write what a linter / formatter / hook / config file can enforce**

If the project's toolchain can mechanically enforce a rule, it does not belong in
CLAUDE.md. "Always run the formatter" should graduate to a pre-commit hook. "Use
2-space indentation" belongs in `.editorconfig` or Prettier config.

Inspect the project's actual config files to determine coverage. See
`linter-capabilities.md` for enforcement mechanisms.

**6. Never write what Claude already does correctly unprompted**

"Write clean code", "Handle errors gracefully", "Follow DRY" are pure noise. If
Claude would do it anyway without the instruction, cut the instruction.

**7. Domain concepts over file paths**

Describe the concept, not the location. Concepts survive refactors; paths break.

- ✅ `Authentication uses session tokens with Redis-backed storage.`
- ❌ `Auth logic lives in src/auth/handlers.ts.`

**8. No @import**

Every `@path` reference expands into context at session launch, consuming tokens
whether or not the content is relevant. This defeats lazy loading.

If content does not need to be in every session, move it to a path-scoped rule
(triggered by file glob) or a skill (triggered semantically). If it does need to be
in every session, inline it directly in CLAUDE.md.

**9. One instruction per bullet**

Combining multiple rules in one bullet reduces compliance with all of them.

- ✅ Three separate bullets, each with one rule.
- ❌ `Use named exports, prefer const over let, and avoid default parameters.`

**10. Commands in code fences with exact flags**

A command in a fenced code block gets followed. A command buried in prose gets lost.

- ✅ `` `pnpm test --filter=api` `` — run API tests
- ❌ "Run the tests for the API package."

**11. Markdown structure as parsing landmarks**

Use H1 for critical anchors and top-level splits, H2 for sections, H3 only when
truly needed. Never H4+. Flat beats deep.

Bold text within sections serves as a secondary landmark. Do not strip markdown
formatting — headers and bold help smaller models parse structure reliably.

**12. Line budget**

- **Target:** 100–150 total lines per CLAUDE.md (total lines = `wc -l`).
- **Hard limit:** 200 total lines. Exceeding this triggers a mandatory audit.
- Principle: "fewer high-quality lines > more mediocre lines."
- Small projects (< 10 source files) may have well under 100 lines — do not pad.

Path-scoped rules: target ~150 total lines, hard limit ~200 (per file). Skills:
SKILL.md body < 500 lines ideal; overflow goes to `references/`. The same "every
line must earn its place" standard applies to all file types.

**13. Litmus test**

For every line: _"If I remove this line, will Claude make a mistake it wouldn't
otherwise make?"_

If no → cut it. If unsure → cut it and observe.

**14. Failure-driven rules default to keep**

During audits, a rule that looks odd but is highly specific (e.g., "Do not add
event handlers when the framework manages reactivity") likely exists because of a
real incident. Default to keeping it unless the user explicitly confirms obsolescence.

### Structure Constraints

- H1 / H2 / H3 only; no H4+
- Commands in code fences (principle 10)
- One instruction per bullet (principle 9)
- No `@import` references (principle 8)

---

## Path Rule Format

```yaml
---
paths:
  - "src/api/**/*.ts"
when: "When working with API endpoint code"
---
# Rule title

Content here: lookup/reference conventions for files matching the glob.
```

**Known issue:** YAML list format (`paths:` with `- items`) may only match the
first pattern in some Claude Code versions. Use CSV format as fallback:

```yaml
---
paths: "src/api/**/*.ts,src/api/**/*.js"
when: "When working with API endpoint code"
---
```

**`when:` field (required):**

One sentence describing the **work scenario** where this content is relevant —
not a restatement of the glob pattern.

- ✅ `"When adding or modifying user-visible text, UI copy, or translations"`
- ✅ `"When working on authentication flows or session management"`
- ❌ `"When files match renderer/**/*.tsx"` — this is the glob, not the scenario

`when:` is a **post-load behavioral hint**, not a pre-load filter. The file is
loaded into context whenever the glob matches — `when:` does NOT prevent this
and does NOT reduce token cost. Its value is behavioral: it tells Claude when
to ACT on the content vs. treat it as background context.

This means `when:` alone does NOT solve the context-budget problem for HIGH
collateral damage rules. The correct solutions for HIGH collateral damage are:
1. **Narrow the glob** — find a more specific file pattern that only matches
   when the content is genuinely needed (e.g., `logger.ts` instead of `main/**/*.ts`)
2. **Route to Skill** — when no narrower glob exists and the trigger is purely
   semantic (e.g., "when adding user-visible text" has no corresponding file pattern)

**`when:` is always auto-written — no user confirmation needed.** Infer it from
the file's content and glob together, not from the glob alone. Write the work scenario
the developer would be in when this file's guidance is most needed. For rebuild, the
`when:` is written during Phase 3B code exploration when the full codebase context
is available, resulting in more accurate scenario descriptions.

**Content constraints:**
- Lookup / reference only — not workflow instructions (those go to Skill)
- One file per domain (`auth.md`, `api.md`, `logging.md`, etc.)
- Target ~150 lines; hard limit ~200
- Include concrete examples, not abstract descriptions

**Collateral damage check (before finalizing path-rule routing):**

After deciding to route to path-rule, ask: "Does this glob fire frequently in scenarios where the content is irrelevant?"

- LOW collateral damage → keep as path-rule (the `when:` field guides Claude to self-filter)
- HIGH collateral damage AND the `when:` scenario has clear semantic boundaries → route to Skill instead

Example: `renderer/**/*.tsx` for i18n rules has HIGH collateral damage — any tsx change triggers it, but i18n rules are only relevant when adding user-visible text. Route to Skill. Example: `apps/plaud-desktop/**` for desktop conventions has LOW collateral damage — any desktop work needs the conventions. Keep as path-rule with `when: "When working on any Electron desktop feature"`.

---

## Skill Format

### Stub (written by handle-one-directive)

```yaml
---
name: <kebab-case>
description: <≤15 words>
status: stub
---
```

handle-one-directive writes only the frontmatter stub. Notify user to complete the
body using `skill-creator`. Step 2's existing-directive scan treats `status: stub`
entries as pending, not as conflicting existing directives.

### Full skill (written by skill-creator)

```yaml
---
name: <kebab-case>
description: <≤15 words>
---
# <Name>

## When to use
...

## Steps
1. ...
2. ...
```

**Constraints:**
- `description` hard limit: **15 words**. Verified via skill-creator eval loop.
- Body: step-by-step, imperative form
- SKILL.md body < 500 lines ideal; overflow goes to `references/`
- Trigger accuracy: description must match current project patterns and tool names

---

## ADR Format (Nygard Template)

```yaml
---
adr: NNNN
title: <short title>
status: Accepted
date: YYYY-MM-DD
---
## Context

<What situation or constraint led to this decision?>

## Decision

<What was decided? State the decision directly.>

## Consequences

<What are the results of this decision — positive, negative, and neutral?>

## Alternatives Considered

<What other options were evaluated and why were they rejected?>
```

**Quality gates:**
- `Consequences` must not be empty — a decision without consequences is incomplete.
- `Alternatives Considered` must name at least one alternative with a rejection reason.
- ADRs always live at `<project-root>/docs/adrs/` regardless of monorepo structure.
- ADR number is zero-padded 4-digit, e.g., `0001`, `0042`.

**ADR definition:** a non-obvious architectural decision that required trade-off
analysis. If someone reading the code could easily infer why a decision was made,
it does not need an ADR. If the rationale would surprise or confuse a future
contributor, it does.

---

## Health Card (used by audit-workflow.md)

### Line counting convention

**Total lines** (including blank lines) is the standard. Use `wc -l` or equivalent.

### CLAUDE.md Health Card Template

```
CLAUDE.md Health Card — <file path>
──────────────────────────────────────────────

Volume
  Line count:              <N> (target 100–150, red line 200)   <✓|⚠|✗>
  MUST/IMPORTANT count:    <N> / 5 (max)                        <✓|⚠|✗>

Separation of concerns
  Overlaps with linter/formatter/
    hooks/config:                        <N> items   → graduation candidates
  Non-root CLAUDE.md files:              <N> items   → consolidate/delete
  Rules missing paths: frontmatter:      <N> items   → add paths: or justify
  Duplicates README/package.json:        <N> items   → drop candidates
  Prohibitions without rationale:        <N> items   → rewrite candidates
  File paths instead of domain concepts: <N> items   → rewrite candidates
  Should be in rules (not CLAUDE.md):    <N> items   → migrate candidates
  Should be in skills (not CLAUDE.md):   <N> items   → migrate candidates
  Wrong direction (every-session fact
    buried in rule/skill):               <N> items   → migrate-up candidates
  @import references:                    <N> items   → inline or migrate

Structure
  Heading structure:             <well-structured / needs improvement>
  Compound bullets (multi-rule): <N> items   → split candidates
  Commands outside code fences:  <N> items   → rewrite candidates

Content quality
  Failure-driven rules (protected):          <N> items   → default keep
  Stale references (paths/tools not found):  <N> items   → verify candidates
  Content Claude does correctly unprompted:  <N> items   → drop candidates

──────────────────────────────────────────────
```

### Indicator Thresholds

| Metric | ✓ | ⚠ | ✗ |
|---|---|---|---|
| Line count | ≤ 150 | 151–200 | > 200 |
| MUST/IMPORTANT count | ≤ 3 | 4–5 | > 5 |
| Linter/formatter/hooks/config overlaps | 0 | 1–2 | ≥ 3 |
| Non-root CLAUDE.md files | 0 | — | ≥ 1 |
| Rules missing paths: frontmatter | 0 | 1 | ≥ 2 |
| Prohibitions without rationale | 0 | 1–2 | ≥ 3 |
| @import references | 0 | — | ≥ 1 |

For other metrics, any count > 0 triggers a proposal. The threshold table above
determines the severity indicator only.

### How to populate each metric

**Volume**
- Line count: total lines (`wc -l`). Compare against 100–150 target and 200 red line.
- MUST/IMPORTANT: count occurrences (case-insensitive). Exclude any `# CRITICAL — Read first`
  section header. Do NOT count "CRITICAL" or "NEVER" here; NEVER prohibitions are tracked
  under "Prohibitions without rationale".

**Separation of concerns**
- Overlaps with linter/formatter/hooks/config: check project's linter config, formatter
  config, pre-commit hooks, `.editorconfig`. If a matching mechanical enforcement exists
  or could trivially be enabled, count it.
- Non-root CLAUDE.md files: glob `**/CLAUDE.md` and `.claude/CLAUDE.md`. Any CLAUDE.md
  not at `./CLAUDE.md` is a consolidation candidate.
- Rules missing `paths:` frontmatter: check each `.claude/rules/*.md`. Rules without it
  load unconditionally — flag unless content genuinely applies to all files.
- Duplicates README/package.json: check if information is already in README.md or
  package.json scripts.
- Prohibitions without rationale: any bullet saying "never/don't/avoid X" without
  explaining why or offering an alternative.
- File paths instead of domain concepts: references to specific file paths that could
  be expressed as domain concepts.
- Should be in rules/skills: content that does not apply to every session but lacks
  a `paths:` scope, or would benefit from semantic triggering.
- Wrong direction: every-session facts buried in a path-scoped rule or skill.
- @import references: count all `@path` references.

**Structure**
- Heading structure: verify H1/H2/H3 hierarchy, no H4+, logical grouping.
- Compound bullets: bullets containing multiple unrelated instructions joined by
  commas or "and".
- Commands outside code fences: commands mentioned in prose rather than fenced blocks.

**Content quality**
- Failure-driven rules: highly specific rules appearing to stem from real incidents.
  Flag for protection, not removal.
- Stale references: grep the codebase for paths, tool names, or module names mentioned
  in the file. Flag anything not found.
- Content Claude does correctly unprompted: generic advice like "write clean code",
  "handle errors", "add tests", "use meaningful names".

### Rules / Skills Summary Template

For each `.claude/rules/*.md` and `.claude/skills/*/SKILL.md`, produce a shorter
summary:

```
Rule/Skill Summary — <file path>
──────────────────────────────────────────────
  Type:                    rule / skill
  Line count:              <N> (rule target ~150, skill < 500)
  paths: / trigger:        <glob pattern> / <description field> / missing
  Layer correct?           <yes / should migrate to ...>
  Overlaps with CLAUDE.md: <N> items → deduplicate
  Overlaps with toolchain: <N> items → graduation candidates
  Stale references:        <N> items → verify
  [rules only]
  Collateral damage:       <low / high — glob fires in many irrelevant situations>
  [skills only]
  Trigger accuracy:        <accurate / stale — description no longer matches project>
──────────────────────────────────────────────
```

### Scoring (used by audit-workflow.md)

Audit scores each category 0-100. Deductions accumulate per finding.

| Category | Max score | Key deductions |
|---|---|---|
| Layer compliance | 25 | Wrong layer: -5 per item; stub skills never completed: -3 each |
| Toolchain efficiency | 25 | Linter-enforceable directive in context: -5 per item; @import: -3 each; project-specific directive in `~/.claude/CLAUDE.md`: -5 each |
| Content freshness | 25 | Stale reference: -5 per item; obsolete tool name: -3 each |
| Format compliance | 25 | Missing `paths:` frontmatter: -5 per item; missing `when:` in path rule (`MISSING_WHEN`): -3 per item; description >15 words: -5; empty Consequences: -5; ADR fails definition (`ADR_INVALID`): -3; ADR content duplicated (`ADR_DUPLICATE`): -3; ADR replaceable by code comment (`ADR_REPLACEABLE`): -2 |

Total score = sum of four category scores (0-100).
