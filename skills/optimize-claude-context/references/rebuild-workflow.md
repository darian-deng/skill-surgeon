# Rebuild Workflow

Reprocesses all existing project knowledge from scratch. Two-track approach:
- **Track A** — CLAUDE.md directives: atomized, routed individually
- **Track B** — path-rule files: reviewed holistically per file (never atomized)

State persists in `.claude/rebuild-progress.md`.

References:
- `directive.md` — atomic definition + scope rules (Track A)
- `decision-tree.md` — routing logic (Track A Phase 4)
- `handle-one-directive.md` — execution unit (Track A Phase 6)
- `writing-formats.md` — format specs including `when:` field
- `linter-capabilities.md` — linter check methodology

---

## Rebuild Progress Tables

State persists in `.claude/rebuild-progress.md`. Three sections.

### Track A Table — CLAUDE.md Directives

| directive | sources (file:lines) | conflict | stale | decision result | domain-file | confidence | status | final file |
|---|---|---|---|---|---|---|---|---|

**Column definitions:**
- **directive**: atomic knowledge point extracted from CLAUDE.md
- **sources**: file + line ranges, e.g. `CLAUDE.md:12-14`
- **conflict**: `Y — <A> vs <B>` if contradicting; empty otherwise
- **stale**: `Y` if grep cannot find referenced paths or tool names
- **decision result**: `Graduate` / `CLAUDE.md` / `path-rule:<glob>` / `skill:<name>` / `ADR` / `deprecated`
- **domain-file**: for CLAUDE.md rows — target section heading (e.g., `## Architecture`); for path-rule rows — domain file stem (e.g., `api`)
- **confidence**: `H` / `M` / `L`
- **status**: `pending` → `in-progress` → `done` / `deprecated` / `conflict-blocked`
- **final file**: exact path written (filled in Phase 6)

### Track B Table — Path-Rule Files

| file | glob | when: (auto) | stale | linter-graduate | scope-issue | recommendation | developer-action | status |
|---|---|---|---|---|---|---|---|---|

**Column definitions:**
- **file**: path-rule file path (from backup)
- **glob**: the `paths:` value from the file's frontmatter
- **when: (auto)**: AI-inferred work scenario — written automatically, no user confirmation needed
- **stale**: `Y` if glob matches no files in current codebase, or content references patterns/APIs no longer present
- **linter-graduate**: `FULL` if entire core purpose is achievable by linter + code comments combined (→ delete candidate); `PARTIAL` if some rules can be linted; empty if not applicable
- **scope-issue**: `Skill` if content is highly scenario-specific within the glob's scope and no file pattern captures the trigger; empty otherwise
- **recommendation**: `KEEP` / `DELETE` / `SKILL` (no REVIEW — partial linter opportunities are noted in the evaluation output and kept as KEEP; developer follows up manually)
- **developer-action**: filled by user in Phase 5: `keep` / `delete` / `skill` / `keep-with-notes:<notes>`
- **status**: `pending` → `evaluated` → `done`

### ADR Table

| ADR file | meets definition | covered by directive | replaceable by code comment | action | target path |
|---|---|---|---|---|---|

### Status State Machine (Track A)

```
pending → in-progress → done
                      → deprecated
                      → conflict-blocked
```

---

## Phase 0 — Dependency Check + ADR Discovery

```bash
ls ~/.claude/skills/skill-creator/ 2>/dev/null \
  || ls ~/.agents/skills/skill-creator/ 2>/dev/null \
  || echo "not found on filesystem"

ls ~/.claude/skills/skill-surgeon/ 2>/dev/null \
  || ls ~/.agents/skills/skill-surgeon/ 2>/dev/null \
  || echo "not found on filesystem"
```

Fallback: if not found on filesystem, check the `available_skills` context for
`skill-creator` and `skill-surgeon` entries.

If either dependency is missing: stop and provide the install command before
continuing. Do not proceed without both.

**ADR directory discovery (dynamic — never hardcode):**

```bash
find . -type d \( -name "adr" -o -name "adrs" -o -name "decisions" \) \
  -not -path "*/node_modules/*" \
  -not -path "*/.git/*" \
  -not -path "*/claude-context-backup/*"
```

Record all discovered ADR directories. In a monorepo each package may have its own.
Store the list as `ADR_DIRS` — used in Phase 1 backup and Phase 2 Subagent B.

---

## Phase 1 — Backup

Paths to back up:
- `./CLAUDE.md`
- `./.claude/rules/`
- `./.claude/skills/` (project-level only)

Destination: `./claude-context-backup/` preserving directory structure.

```bash
mkdir -p ./claude-context-backup/docs
cp ./CLAUDE.md ./claude-context-backup/ 2>/dev/null || true
cp -r ./.claude ./claude-context-backup/ 2>/dev/null || true
# Copy all ADR directories discovered in Phase 0
for adr_dir in $ADR_DIRS; do
  mkdir -p "./claude-context-backup/${adr_dir}"
  cp -r "./${adr_dir}/." "./claude-context-backup/${adr_dir}/" 2>/dev/null || true
done
```

**Abort condition:** if `./claude-context-backup/` already exists, **abort and warn
the user.** Abort fires before any files are copied or deleted — the originals remain
intact. Tell the user to delete or rename the backup before rerunning.

After copying: remove the originals.

```bash
rm -f ./CLAUDE.md
rm -rf ./.claude/rules/ ./.claude/skills/
# ADR directories preserved in place; processed in Phase 7
```

Global `~/.claude/CLAUDE.md` is NOT backed up — read in Phase 2 but never modified.

**Recovery from mid-rebuild interruption (Phase 2–5):**

If the rebuild is interrupted after Phase 1 (originals deleted, backup exists),
`claude-context-backup/` will block a fresh restart. To recover:

1. Restore originals from backup: `cp -r claude-context-backup/.claude . && cp claude-context-backup/CLAUDE.md .`
2. Rename backup: `mv claude-context-backup/ claude-context-backup-prev/`
3. Check `.claude/rebuild-progress.md` — if rows exist with `in-progress` or
   `evaluated` status, you can resume from the last completed phase rather than
   restarting from Phase 2. Read the table to determine which phases are done.
4. If restarting from Phase 6, all Track A rows should be `in-progress` and Track B
   rows `evaluated` — jump directly to Phase 6 execution.

---

## Phase 2 — Collection

Two parallel subagents.

### Subagent A — Track A: CLAUDE.md (atomize directives)

Processes in order:

1. `~/.claude/CLAUDE.md` (global): for each directive, determine if truly global or
   project-specific (see directive.md scope rules). Project-specific entries go into
   the Track A table with source note `"~/.claude/CLAUDE.md — needs manual removal"`.
   Global directives: note only, do NOT add to table.

2. `<project-root>/CLAUDE.md` (from backup)

3. Any CLAUDE.md files in subdirectories (from backup)

For each source, extract every directive, write immediately to Track A table as `pending`.

### Subagent B — Track B: path-rule files (collect as units)

For each file in `claude-context-backup/.claude/rules/*.md`:
- Record file path, glob (from `paths:` frontmatter), and line count
- Write one row to the Track B table as `pending`
- Do NOT extract individual directives — treat the file as an atomic unit

**Subagent B (ADR) — after Track B:**
All files in discovered ADR directories → write each to the ADR table.

---

## Phase 3 — Track A: Domain Grouping + Dedup

Main agent reads the complete Track A table.

**Domain definition:** directives in the same functional area AND same scope.
Over-splitting is always safe.

Per domain group:
- **Consistent sources** → merge into one row, keep clearest phrasing
- **Contradicting sources** → mark `conflict: Y — <A> vs <B>`, set `conflict-blocked`

**Stale detection (Track A):** for each directive referencing file paths or tool names:

```bash
find . -path "<referenced-path>" 2>/dev/null | head -3
grep -r "<tool-name>" . --include="package.json" --include="*.toml" -l 2>/dev/null | head -3
```

No results → `stale: Y`. Stale is advisory — still goes through Phase 4 review.

---

## Phase 3B — Track B: Holistic Path-Rule File Evaluation

For each path-rule file in the Track B table, dispatch one subagent per file
(or 2-3 small files per subagent). **Evaluation order is critical:**

**Step 1 — Code exploration (always first)**

Using `feature-dev:code-explorer`, explore the actual code matching the file's glob:
- What patterns, APIs, and conventions currently exist in those files?
- Do the referenced patterns in the rule file still match the codebase?
- Does the glob itself still match meaningful files?

This step establishes ground truth before any judgment is made.

**Step 2 — Stale assessment**

Based on code exploration:
- Does the glob match any files in the current codebase?
- Do the specific APIs, module names, or patterns referenced in the rule still exist?
- If the rule references patterns that no longer exist → `stale: Y`

**Step 3 — Holistic linter + comment graduation check**

Ask: **"Can the CORE purpose of this entire file — the thing that would cause Claude
to make mistakes without it — be fully achieved by linter rules + strategic code
comments combined?"**

This is a global question about the file's net value, not about individual lines.
See `linter-capabilities.md` for enforcement mechanisms.

- If YES (linter + comments fully cover the core purpose) → `linter-graduate: FULL`,
  `recommendation: DELETE`. The file has no remaining value in the context layer.
- If PARTIAL (some rules can be linted, but the remaining content still has value) →
  `linter-graduate: PARTIAL`, note the linter opportunity. `recommendation: KEEP`.
  Do NOT strip individual lines — surface as a note for developer follow-up.
- If NO → proceed to Step 4.

**Why Track B allows "linter + code comments" while Track A (individual directives)
only uses linter:** Track B evaluates whole domain files that contain both enforcement
rules (linter-addressable) AND explanatory content (why this pattern, what context,
what to watch for). Explanatory content cannot be linter-enforced but CAN be captured
in JSDoc or inline comments in the relevant code. Track A deals with single atomic
rules where the question is simply "can this constraint be mechanically enforced?"

**Step 4 — Scope vs content alignment**

If the file survives Step 3, assess: is the content scenario-specific relative to
the glob's scope?

"Is this content relevant for ALL work within the glob's scope, or only for a
SPECIFIC TYPE of work within the scope?"

- ALL work in scope → `recommendation: KEEP`
- SPECIFIC TYPE of work, and the trigger is purely semantic (no file pattern can
  capture "the developer is doing X") → `recommendation: SKILL`

**Do NOT attempt to suggest a narrower glob.** Narrowing requires understanding the
original author's intent, which cannot be reliably inferred from content alone. A
wrong narrowing is worse than the current broad glob.

**Step 5 — Auto-write `when:` (always, no user confirmation)**

Based on the file's content AND code exploration findings, write a `when:` statement
that describes the work scenario where this file's guidance is relevant.

The `when:` is written to the Track B table automatically. It will be inserted into
the file's frontmatter during Phase 6 execution for every KEEP file. It serves future
`handle-one-directive` calls and `audit` assessments.

Track B has exactly three recommendation values: `KEEP` / `DELETE` / `SKILL`.
There is no `REVIEW` state. If `linter-graduate: PARTIAL`, recommendation is `KEEP`
with a note in the output describing the linter opportunity — the developer decides
whether to follow up manually. Never split or partially strip the file.

**Subagent output format per file:**

```
File: logging.md
Glob: src/main/**/*.ts
Code exploration: found createLogger at src/main/utils/logger.ts; ESLint has
  no-restricted-syntax but no rule for console.log; pattern is used in 12 files
Stale: No — glob matches 47 files, all patterns exist
Linter-graduate: FULL — core purpose (use createLogger, not raw console)
  can be enforced by ESLint no-restricted-syntax on ConsoleExpression +
  createLogger's own JSDoc documents domain/level pattern
  Reference: https://eslint.org/docs/latest/rules/no-restricted-syntax
Recommendation: DELETE
when: "When adding or modifying log calls in Electron main process or preload"

File: ui-tokens.md
Glob: renderer/**/*.tsx
Code exploration: CSS variables and shadcn components used across all renderer files
Stale: No
Linter-graduate: No
Linter-graduate-partial: Line 8 (no hardcoded color values) could be enforced via
  ESLint no-restricted-syntax on Literal hex patterns — but remaining content has
  standalone value. Noting as linter opportunity for developer follow-up.
Recommendation: KEEP
when: "When developing Renderer UI components, selecting colors, spacing, or tokens"

File: i18n.md
Glob: apps/plaud-desktop/src/renderer/**/*.tsx
Code exploration: 10 languages in locales/, react-i18next in use
Stale: No
Linter-graduate: No
Scope-issue: SPECIFIC TYPE — content is scenario-specific (adding user-visible text);
  no file pattern captures "adding user-facing copy" vs. "refactoring logic"
Recommendation: SKILL
when: "When adding or modifying user-visible text, UI copy, or translations"
```

---

## Phase 4 — Track A: Decision Tree Evaluation + ADR Pre-screening

**Track A (CLAUDE.md directives)** is evaluated here. Track B was fully evaluated in Phase 3B.

**IMPORTANT:** This phase writes results to the table only. No context layer files
are written here.

Skip rows with status `conflict-blocked`.

**ADR Pre-screening (parallel with Track A evaluation):**

ADR Check 2 (covered by a Phase 6 directive?) depends on Phase 6 results and must
wait for Phase 7. But Check 1 and Check 3 are fully independent — run them now in
parallel with Track A subagents to give the user richer information at Phase 5.

Dispatch one ADR pre-screening subagent alongside Track A batches:
- **Check 1**: Does each ADR meet the ADR definition? (Non-obvious architectural
  decision, trade-off analysis, would confuse future contributors without it?)
  If not → mark `pre-screen: fails-definition`
- **Check 3**: Can the rationale be expressed as a short inline code comment?
  If yes → mark `pre-screen: replaceable-by-comment`, note suggested comment text

Only run Check 1 and Check 3. Do NOT run Check 2 here. Results are written to the
ADR table's `pre-screen` column. Phase 7 will run Check 2 (and confirm/override
pre-screen results) after Phase 6 completes.

Skip rows with status `conflict-blocked`.

Group remaining Track A rows by domain. For each batch of 3-4 directives from the
same domain, dispatch one subagent:

1. Uses `feature-dev:code-explorer` to explore relevant code (shared per batch)
2. Applies decision-tree.md Steps 1-3 for each directive (Step 0 skipped — Phase 2 already guarantees atomicity)
3. Writes `decision result`, `domain-file`, `confidence` back to each row
4. Does NOT write any context layer files

**Subagent output format:**

```
Batch: <domain> (<scope>)
Code exploration: <what was found>

Row updates:
- directive: "<text>"
  decision: CLAUDE.md
  domain-file: "## Architecture"
  confidence: H

- directive: "<text>"
  decision: path-rule:apps/plaud-desktop/**
  domain-file: api
  confidence: H

- directive: "<text>"
  decision: skill:add-rpc-method
  confidence: H
  Note: Multi-step procedure AND developer confirms will flesh out with skill-creator

- directive: "<text>"
  decision: deprecated
  confidence: H
  Reason: <why>
```

After all subagents complete, merge row updates into `.claude/rebuild-progress.md`.
Set all updated rows from `pending` to `in-progress`.

---

## Phase 5 — User Alignment

Present both tracks to the user.

**Required resolutions before Phase 6:**
- Track A: all `conflict: Y` rows (must resolve)
- Track A: all `confidence: L` rows (must resolve)
- Track B: all `recommendation: DELETE` or `recommendation: SKILL` rows
  (developer must choose: keep / delete / skill / keep-with-notes)

**Optional adjustments:**
- User may change any Track A decision result
- User may mark any row as deprecated
- User may edit any `when:` statement (though auto-written values are usually correct)

**Presentation format:**

```
## Rebuild Progress — User Alignment Required

### Track A — CLAUDE.md Directives

#### CONFLICT rows (must resolve):
| directive | conflict | sources |
| ...       | ...      | ...     |

#### Low confidence rows (review recommended):
| directive | decision result | confidence | reason |
| ...       | ...             | ...        | ...    |

#### All decisions (informational):
| directive | decision | domain-file | confidence |
| ...       | ...      | ...         | ...        |

---

### Track B — Path-Rule Files

#### DELETE candidates (linter + comments fully cover core purpose):
For each: confirm delete, or override to keep. If overriding, state reason.
| file | glob | why linter covers it | linter config hint | your decision |
| ...  | ...  | ...                  | ...                | keep / delete |

#### SKILL candidates (content scenario-specific, no file pattern captures trigger):
For each: confirm migrate to Skill, or override to keep.
| file | glob | why Skill | your decision |
| ...  | ...  | ...       | keep / skill  |

#### KEEP files (informational — `when:` auto-written, shown for transparency):
Partial linter opportunities are noted in the evaluation output — developer may
follow up manually. No action required to proceed.
| file | glob | when: (auto) | linter-partial note |
| ...  | ...  | ...          | ...                 |
```

**Do not proceed to Phase 6 until the user explicitly confirms.**

---

## Phase 6 — Execution

### Track B Execution (path-rule files — execute first)

For each Track B file with developer action confirmed:

**`keep` or `keep-with-notes`:** write the file to `.claude/rules/<filename>` with
`when:` added to the frontmatter. Include any developer notes as comments.
Write is **overwrite** (Phase 1 cleared originals).

**`delete`:** file is not written. Log to rebuild-progress.md as done/deleted.

**`skill`:** write a Skill stub to `.claude/skills/<name>/SKILL.md` with frontmatter
only. Notify user to complete with skill-creator.

### Track A Execution (CLAUDE.md directives)

**Pre-execution: CLAUDE.md section plan**

Before dispatching subagents, the main agent looks at all CLAUDE.md-destined rows
and creates a section plan. Group by `domain-file` heading, determine order
(behavioral rules first, architecture second, conventions third, commands last).
This plan is passed to every subagent so all directives land in coherent structure.

For each confirmed, non-deprecated, non-conflict-blocked Track A row:

Call handle-one-directive in **rebuild-execute mode** (starts at Step 4, enrichment).

Batch by domain: 3-4 directives per subagent.

Each subagent:
1. Reads table row: directive text, sources, decision result, domain-file, scope
2. Calls handle-one-directive in rebuild-execute mode
3. For **CLAUDE.md** rows: write under the pre-planned section heading
4. For **path-rule** rows (from CLAUDE.md wrong-layer content): write to
   `.claude/rules/<domain-file>.md` with `when:` in frontmatter; overwrite
5. Updates table row `status` to `done` and `final file` to exact path written

**Recovery:** any `in-progress` rows after all subagents complete → call
handle-one-directive rebuild-execute mode manually for each, then set `done`.

---

## Phase 7 — ADR Processing

Runs after Phase 6 is complete.

For each ADR in the ADR table, apply four checks in order. Stop at first match.

**Check 1 — Meets ADR definition?**
Non-obvious architectural decision, trade-off analysis, would confuse future
contributor without it? If no → `action: deprecated`.

**Check 2 — Content covered by a Phase 6 directive?**
Already captured in CLAUDE.md or path rule? → `action: deprecated (directive sufficient)`.

**Check 3 — Replaceable by a code comment?**
Can be expressed as a short comment at the relevant code location? →
`action: deprecated`; write suggested comment text to table row as note for user.

**Check 4 — Valid and necessary.**
ADRs are preserved in place during Phase 1 — no file move needed.
Update or create the index at `<adr-dir>/README.md` (using the discovered path).

**ADR directory convention:** canonical path is `docs/adr/` (singular — matches
adr-tools origin and grill-with-docs). If creating a new ADR directory, always
use `docs/adr/`; never `docs/adrs/`. In a monorepo, package-specific ADRs go in
`<package>/docs/adr/`. Dynamic discovery in Phase 0 handles both spellings for
existing projects.

**Deprecated ADRs:** move to `./claude-context-backup/<adr-dir>/deprecated/`.
Do not delete permanently, do not leave in the active ADR directory.

---

## Phase 8 — Final Verification + Cleanup

```bash
wc -l ./CLAUDE.md                    # target 100-150, hard limit 200
find .claude/rules -name "*.md" -exec head -6 {} \;   # confirm paths: + when: frontmatter
find .claude/skills -name "SKILL.md" | xargs grep -l "status: stub" 2>/dev/null
```

**Global CLAUDE.md migration note:** if directives were collected from
`~/.claude/CLAUDE.md` in Phase 2, list them and tell user to remove manually.

**Present new context layer for review.**

After user confirms:
```bash
rm .claude/rebuild-progress.md
```

`./claude-context-backup/` stays. Tell user: "Delete when satisfied with the new layer."

---

## Parallelism Summary

| Phase | Parallelism |
|---|---|
| Phase 2 Subagent A + B | Parallel (different files, no shared writes) |
| Phase 3B Track B evaluation | Parallel (one subagent per file or 2-3 small files) |
| Phase 4 Track A domain batches | Parallel (3-4 directives per subagent) |
| Phase 6 Track B | Sequential per file (simple writes) |
| Phase 6 Track A domain batches | Parallel (same batching as Phase 4) |
| Phase 7 ADR checks | Sequential (one ADR at a time) |
