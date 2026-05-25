# Rebuild Workflow

Reprocesses all existing project knowledge from scratch. Table-driven, domain-grouped,
subagent-parallel. State persists in `.claude/rebuild-progress.md`.

References:
- `directive.md` — atomic definition + scope rules
- `decision-tree.md` — routing logic (used in Phase 4)
- `handle-one-directive.md` — execution unit (used in Phase 6)
- `writing-formats.md` — format specs (used in Phase 6)
- `linter-capabilities.md` — linter check methodology (used in Phase 4)

---

## Rebuild Progress Table

State persists in `.claude/rebuild-progress.md`. Written continuously throughout
the run. Two sections: non-ADR directives and ADR entries.

### Non-ADR Table

| directive | sources (file:lines) | conflict | stale | decision result | domain-file | when (path-rule) | when-confidence | confidence | status | final file |
|---|---|---|---|---|---|---|---|---|---|---|

**Column definitions:**

- **directive**: atomic knowledge point (one sentence, one concept)
- **sources**: all files + line ranges where this directive or semantic overlap appears,
  comma-separated, e.g. `CLAUDE.md:12-14, .claude/rules/api.md:20`
- **conflict**: `Y — <content A> vs <content B>` if sources contradict; empty otherwise
- **stale**: `Y` if grep cannot find referenced file paths or tool names in current
  codebase; empty otherwise
- **decision result**: `Graduate` / `CLAUDE.md` / `path-rule:<glob>` / `skill:<name>` /
  `ADR` / `deprecated`
- **domain-file**: for `CLAUDE.md` rows — the target section heading (e.g., `## Architecture`);
  for `path-rule` rows — the target domain file stem (e.g., `api`, `auth`, `logging`)
- **when (path-rule)**: only for path-rule rows — one sentence describing the work scenario
  where this content is relevant (see `writing-formats.md §Path Rule Format`)
- **when-confidence**: only for path-rule rows — `H` / `M` / `L` (see below)
- **confidence**: `H` / `M` / `L` for the routing decision itself
- **status**: `pending` → `in-progress` → `done` / `deprecated` / `conflict-blocked`
- **final file**: exact target file path (filled in Phase 6 after writing)

**`when-confidence` for path-rule rows:**
- **H**: glob alone implies the scenario (package-root globs, standard file types) — auto-written, no user confirmation needed
- **M**: inferred from glob + content analysis — shown in Phase 5 Tier-B for batch confirm
- **L**: broad glob with specialized content — shown in Phase 5 Tier-A for explicit confirm

### ADR Table (separate section in same file)

| ADR file | meets definition | covered by directive | replaceable by code comment | action | target path |
|---|---|---|---|---|---|

### Status State Machine

```
pending → in-progress → done
                      → deprecated
                      → conflict-blocked
```

Only Phase 6 sets `done`. Phase 3 sets `conflict-blocked`. Phase 5 (user) may set
`deprecated`. All other transitions are set by the phase that processes the row.

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

Paths to back up (copy structure, then delete originals):
- `./CLAUDE.md`
- `./.claude/rules/`
- `./.claude/skills/` (project-level only — not `~/.claude/skills/`)
- `./docs/adrs/`

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
the user.** Do not overwrite a previous backup automatically. Tell the user to
delete or rename it before rerunning. Abort fires before any files are copied or
deleted — the originals remain intact.

After copying: remove the originals.

```bash
rm -f ./CLAUDE.md
rm -rf ./.claude/rules/ ./.claude/skills/
# docs/adrs/ is preserved in place; ADR processing is in Phase 7
```

Global `~/.claude/CLAUDE.md` is NOT backed up — it is read in Phase 2 but never
automatically modified.

---

## Phase 2 — Collection

Two subagents run sequentially (Subagent A must complete before Subagent B, since
both write to the same file).

**Subagent A (non-ADR) — processes in order:**

1. `~/.claude/CLAUDE.md` (global):
   - For each directive: is it truly global (applies to all projects equally) or
     project-specific? Use these criteria:
     - **Truly global**: applies regardless of which project is open — e.g., "always
       respond in Markdown", "prefer functional style", "use British English". Contains
       no project names, no specific paths, no tool names unique to one codebase.
     - **Project-specific**: mentions project name, a tool configured only for this
       repo, a path pattern specific to this codebase, or would not make sense in a
       different project context — e.g., "Use pnpm for this project", "The Electron
       main process uses four-layer architecture".
   - Project-specific directives enter the non-ADR table with note in `sources`:
     `"source: ~/.claude/CLAUDE.md — needs manual removal from global file"`
   - Global directives: note existence only; do NOT add to table

2. `<project-root>/CLAUDE.md` (from backup: `claude-context-backup/CLAUDE.md`)

3. `./.claude/rules/*.md` (from backup: `claude-context-backup/.claude/rules/`)

4. Any `CLAUDE.md` files in subdirectories (from backup)

For each file, extract every directive. Write each to the non-ADR table immediately
(do not wait until the file is fully read). Use `pending` as initial status.

Dynamic batching: if a directory has many large files, process in sub-batches of
3-5 files at a time to avoid context overflow.

**Subagent B (ADR) — processes after Subagent A:**

All files in `claude-context-backup/docs/adrs/` → write each to the ADR table.
For each ADR, record: file path, title, status (from frontmatter).

---

## Phase 3 — Domain Grouping + Deduplication

Main agent reads the complete non-ADR table.

**Domain definition:** directives are in the same domain if they share the same
functional area (auth, logging, API, build, testing, error-handling, architecture,
monorepo-layout, etc.) AND the same scope. Over-splitting into more domains is
always safe — it only affects batching efficiency, not correctness.

For each domain group:

**Consistent sources:** directives from different sources that say the same thing →
merge into one row, all sources listed. Write the clearest phrasing.

**Contradicting sources:** directives from different sources that say different
things → mark `conflict: Y — <content A> vs <content B>`, set status to
`conflict-blocked`.

**Stale detection:** for each directive that references file paths or tool names,
run grep checks:

```bash
# File path check
find . -path "<referenced-path>" 2>/dev/null | head -3

# Tool/library name check
grep -r "<tool-name>" . --include="package.json" --include="*.toml" \
  --include="*.yaml" -l 2>/dev/null | head -3
```

If no results: set `stale: Y` in the table row. The stale flag is advisory —
stale directives still go through Phase 4 and Phase 5 user review.

---

## Phase 4 — Decision Tree Evaluation

**IMPORTANT:** This phase evaluates decisions and writes results to the table only.
It does NOT call handle-one-directive and does NOT write any context layer files.
File writing happens in Phase 6.

Skip rows with status `conflict-blocked` — do not include them in any subagent batch.

Group the remaining table rows by domain. For each batch of 3-4 directives from the
same domain, dispatch one subagent with the following inputs:
- The directive rows (text + sources + conflict + stale flags)
- The domain name and scope
- Instructions to apply decision-tree.md Steps 0-3 for each directive

Each subagent:
1. Uses `feature-dev:code-explorer` to explore relevant code for the domain
   (one exploration shared across all directives in the batch — reuse the findings)
2. Applies decision tree Steps 0-3 for each directive
3. Writes: `decision result` + `confidence` back to each table row
4. Does NOT write any context layer files

**Subagent output format:**

```
Batch: <domain> (<scope>)
Code exploration summary: <what was found>

Row updates:
- directive: "<text>"
  decision: CLAUDE.md
  domain-file: "## Architecture"   ← section heading for CLAUDE.md
  confidence: H

- directive: "<text>"
  decision: path-rule:apps/plaud-desktop/**
  domain-file: api                 ← domain file stem → .claude/rules/api.md
  when: "When working on any Electron desktop feature"
  when-confidence: H               ← auto from package-root glob pattern
  confidence: H

- directive: "<text>"
  decision: path-rule:renderer/**/*.tsx
  domain-file: i18n
  when: "When adding or modifying user-visible text or translations"
  when-confidence: L               ← broad glob + specialized content
  confidence: M
  Reason: <why M and not H>

- directive: "<text>"
  decision: skill:add-log-category
  confidence: H

- directive: "<text>"
  decision: deprecated
  confidence: H
  Reason: <why deprecated>
```

After all subagents complete, the main agent merges the row updates into
`.claude/rebuild-progress.md`. Set all updated rows from `pending` to `in-progress`.

---

## Phase 5 — User Alignment

Present `.claude/rebuild-progress.md` to the user.

**Required resolutions (user MUST resolve before Phase 6 can proceed):**
- All rows with `conflict: Y` (status: `conflict-blocked`)
- All rows with `confidence: L`

**Optional adjustments:**
- User may change any `decision result`
- User may mark any row as `deprecated`
- User may adjust any `when:` statement
- User may add notes to any row

**Presentation format:**

```
## Rebuild Progress — User Alignment Required

### CONFLICT rows (must resolve):
| directive | conflict | sources |
| ...       | ...      | ...     |

### Low confidence rows (review recommended):
| directive | decision result | confidence | reason |
| ...       | ...             | ...        | ...    |

### `when:` confirmation — Tier-A (explicit confirm required, when-confidence: L):
These path-rule `when:` statements are uncertain — broad globs with specialized content.
Approve or rewrite each one before Phase 6 executes.
| directive | glob | proposed when: | approve/rewrite |
| ...       | ...  | ...            | ...             |

### `when:` confirmation — Tier-B (batch confirm, when-confidence: M):
Review and approve as a group, or flag individual items to adjust.
| directive | glob | proposed when: |
| ...       | ...  | ...            |

### `when:` auto-applied (informational, when-confidence: H):
Package-root globs — no confirmation needed. Shown for transparency.
| directive | glob | when: |
| ...       | ...  | ...   |

### Full table (for reference):
[link to .claude/rebuild-progress.md]
```

**Do not proceed to Phase 6 until the user explicitly confirms.** End your response
after presenting. Wait for the user's reply.

---

## Phase 6 — Execution (non-ADR)

**Pre-execution: CLAUDE.md section plan**

Before dispatching any subagents, the main agent looks at all CLAUDE.md-destined rows
and creates a section plan. Group directives by their `domain-file` heading value,
then determine the final section order (behavioral rules first, architecture second,
conventions third, commands last). This plan is passed to every CLAUDE.md subagent so
all directives land in a coherent structure rather than being appended ad-hoc.

Example section plan:
```
CLAUDE.md section plan:
  ## Architecture     → directives: [row 3, row 7, row 12]
  ## Conventions      → directives: [row 5, row 8]
  ## Build & Test     → directives: [row 14, row 19]
```

For each confirmed, non-deprecated table row (status: `in-progress`, not
`conflict-blocked` or `deprecated`):

Call handle-one-directive in **rebuild-execute mode**. This mode starts at Step 4
(content enrichment) using the already-decided layer from the table row.

Batch by domain: 3-4 directives from the same domain per subagent. Batching is a
parallelism optimization — correctness is not affected by batch boundaries.

Each subagent:
1. Reads the table row: directive text, sources (file:lines for original content),
   decision result, domain-file, when: (for path-rule rows), scope
2. Calls handle-one-directive in rebuild-execute mode
3. handle-one-directive runs Step 4 (enrichment) + Step 5 (write)
4. For **path-rule** rows: write to `.claude/rules/<domain-file>.md` —
   **overwrite** (not append) since Phase 1 cleared originals; include `when:` in frontmatter
5. For **CLAUDE.md** rows: write under the pre-planned section heading
6. After writing: updates the table row `status` to `done` and `final file` to the
   exact path written

After all subagents complete, verify all non-deprecated rows have status `done`.
Any still `in-progress` rows indicate a subagent failure.

**Recovery:** for each `in-progress` row, call handle-one-directive in
rebuild-execute mode with that directive's table data (directive text, decision result,
scope from the table row), then set its status to `done` manually. Do not proceed
to Phase 7 until all non-deprecated rows are `done`.

---

## Phase 7 — ADR Processing

Runs after Phase 6 is complete.

For each ADR in the ADR table, apply four checks in order. Stop at the first that
applies.

**Check 1 — Meets ADR definition?**

An ADR must be a non-obvious architectural decision that required trade-off analysis.
Would a future contributor be surprised or confused by the decision without this
rationale? If no → mark `action: deprecated`.

**Check 2 — Content already covered by a Phase 6 directive?**

If the ADR's rationale is already captured by a CLAUDE.md or path rule written in
Phase 6 → mark `action: deprecated (directive is sufficient)`.

**Check 3 — Replaceable by a code comment?**

Can the rationale be expressed in a short comment at the relevant code location?
If yes → mark `action: deprecated`; write the suggested comment text to the table
row as a note for the user.

**Check 4 — Valid and necessary.**

Archive to `<project-root>/docs/adrs/` with the original filename. (ADRs were
preserved in place during Phase 1 — no file move needed for Check 4.) Update or
create `docs/adrs/README.md` as an index:

```markdown
# Architecture Decision Records

| ADR | Title | Status | Date |
|---|---|---|---|
| [0001](0001-use-pnpm.md) | Use pnpm as package manager | Accepted | 2025-01-15 |
```

**Deprecated ADRs (Checks 1/2/3):** move to `./claude-context-backup/docs/adrs/deprecated/` — do not delete permanently, and do not leave them in `docs/adrs/`. Tell the user which ADRs were deprecated and why.

---

## Phase 8 — Final Verification + Cleanup

**Verification checks:**

```bash
wc -l ./CLAUDE.md                    # target 100-150, hard limit 200
find .claude/rules -name "*.md" -exec head -5 {} \;   # confirm paths: frontmatter
find .claude/skills -name "SKILL.md" | xargs grep -l "status: stub" 2>/dev/null
# list stub skills for user — remind to complete with skill-creator
```

**Global CLAUDE.md migration note:**

If any directives were collected from `~/.claude/CLAUDE.md` in Phase 2, display
which ones were migrated. Tell the user:

> "The following directives were found in your global `~/.claude/CLAUDE.md` and
> are now in the project context layer. Please remove them from the global file
> manually — rebuild does not modify that file."
> [list of migrated directives]

**Present the new context layer to the user for review.**

After user confirms:

```bash
rm .claude/rebuild-progress.md
```

`./claude-context-backup/` stays in place. Tell the user: "Backup preserved at
`./claude-context-backup/` — delete it when you're satisfied with the new layer."

---

## Parallelism Summary

| Phase | Parallelism |
|---|---|
| Phase 2 Subagent A + B | Sequential (A before B) |
| Phase 4 domain batches | Parallel (3-4 directives per subagent, all domains concurrent) |
| Phase 6 domain batches | Parallel (same batching as Phase 4) |
| Phase 7 ADR checks | Sequential (one ADR at a time) |
