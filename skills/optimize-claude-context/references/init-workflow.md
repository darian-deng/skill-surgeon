# Init Workflow

Builds a project's Claude Code context layer from scratch using parallel subagent
deep exploration. Optimized for quality — expects large token and time budgets.

## Flow at a glance

```
Phase 0 — Check for existing artifacts
  → None found       → Phase 1
  → Found, choice A  → delete → Phase 1
  → Found, choice B  → backup to ./claude-context-backup/ → Phase 1 → Phase 4

Phase 1 — Scale detection → N module scopes + slug map + create working dirs
Phase 2 — N parallel subagents → N reports in .claude/init-progress/
Phase 3 — Bootstrap draft → read reports one at a time → .claude/init-draft/ + uncertainties
Phase 4 — (rebuild only) Batched backup comparison → .claude/init-progress/backup-compare-*.md
Phase 5 — Health check subagent → inline health card → draft refinements
Phase 6 — STOP. Present to user. Wait for explicit confirmation.
Phase 7 — Read scaffolds + backup approvals → apply → cleanup → verify → skill handoff
```

---

## Phase 0 — Check for existing artifacts

**Dependency check (filesystem — do not rely on available_skills context):**

```bash
ls ~/.claude/skills/skill-creator/ 2>/dev/null || ls ~/.agents/skills/skill-creator/ 2>/dev/null
ls ~/.claude/skills/skill-surgeon/ 2>/dev/null || ls ~/.agents/skills/skill-surgeon/ 2>/dev/null
```

If either is missing, stop and give the user the install command before continuing.

**Artifact scan:**

```bash
find . -maxdepth 1 -name "CLAUDE.md" 2>/dev/null
find .claude/rules -name "*.md" 2>/dev/null
find .claude/skills -name "SKILL.md" 2>/dev/null
[ -d docs/adr ] && echo "docs/adr/ exists" || true
find . -maxdepth 1 \( -name "AGENTS.md" -o -name ".cursorrules" \) 2>/dev/null
```

Note: AGENTS.md and .cursorrules are out of scope for this skill — detect and mention
to user, but do not modify.

**If nothing found (excluding AGENTS.md / .cursorrules):** proceed to Phase 1.

**If anything found:** stop and ask the user:

> I found existing context artifacts:
> - [list each: e.g., CLAUDE.md (87 lines), .claude/rules/api.md, .claude/skills/deploy/]
> - [if AGENTS.md or .cursorrules exist: note them as out of scope]
>
> How do you want to proceed?
> **(A) Delete and start fresh** — remove all existing artifacts, build new from scratch
> **(B) Backup and rebuild** — move existing to `./claude-context-backup/`, build
> new, then compare backup items to recover anything still valuable
>
> If unsure, B is safer — nothing is deleted.

- **(A) confirmed:** confirm once showing exact commands, then execute:
  ```bash
  rm -f CLAUDE.md
  rm -rf .claude/rules/ .claude/skills/
  ```
  Then proceed to Phase 1.
- **(B) confirmed:** move all found artifacts to `./claude-context-backup/` preserving
  directory structure → Phase 1 (backup comparison in Phase 4, after draft exists)
- **Unclear:** ask again. Never guess.

---

## Phase 1 — Scale detection

Read these files to understand project structure:
- Workspace config: `pnpm-workspace.yaml`, `go.work`, `turbo.json`, `nx.json`,
  `lerna.json`, `package.json` (workspaces field), `Cargo.toml` ([workspace] section)
- Top-level `ls -la`

If none of the above files indicate a multi-package structure, treat as single package
regardless of language. Set MODULE_PATH to `.` (project root).

Determine subagent count and scope:

| Scale | N subagents |
|---|---|
| Single package | 1 |
| Small monorepo (2–4 packages) | 1 per package |
| Large monorepo (5+ packages) | 1 per major package, cap at 8 |
| Monorepo with `apps/` + `packages/` split | 1 per app, 1–2 for all shared packages |

For each scope record: module path + primary stack (from manifest).

**Module slug convention:** replace all slashes with hyphens, lowercase.
Example: `apps/plaud-desktop` → `apps-plaud-desktop`.

**Uniqueness check:** after generating all slugs, verify no two are identical. If any
collide (e.g., `apps/api` and a root-level `apps-api` both yield `apps-api`), append
`-2`, `-3`, etc. to disambiguate. Record the final MODULE_PATH → MODULE_SLUG mapping.

Create working directories now:

```bash
mkdir -p .claude/init-progress
mkdir -p .claude/init-draft/rules
touch .claude/init-uncertainties.md
```

---

## Phase 2 — Parallel deep exploration

Dispatch **all N subagents concurrently in the same turn.**

**Full subagent prompt** (fill `MODULE_PATH`, `STACK`, `N_MODULES`, `MODULE_SLUG`):

---

You are exploring module `MODULE_PATH` (stack: `STACK`) as part of a parallel
codebase scan. `N_MODULES` modules are running in parallel.

Your output must be written to `.claude/init-progress/MODULE_SLUG.md`.

**TRACK A — Config files (read every config file in FULL into your context — no
truncation, no skimming). The 200-line limit below applies to your OUTPUT REPORT,
not to what you read:**
Linter: `eslint.config.*`, `.eslintrc*`, `ruff.toml`, `pyproject.toml [tool.ruff]`,
`clippy.toml`. Formatter: `.prettierrc*`, `rustfmt.toml`, `.editorconfig`. Hooks:
`.husky/*`, `.pre-commit-config.yaml`. Type checker: `tsconfig*.json`, `mypy.ini`,
`pyrightconfig.json`. Build: `Makefile`, `justfile`, `package.json` scripts section.

The 200-line report cap means: summarize what you found, do not copy config file
contents verbatim into the report.

**TRACK B — Source files (~20 files):**
Entry points → largest files per major directory → 2–3 test files → README.md,
CONTRIBUTING.md. While reading source, actively look for multi-step workflows where a
developer must touch multiple files in a specific order.

**Also check:** does `<project-root>/docs/adr/README.md` exist? (Always the
project root, regardless of monorepo structure.) If it exists, CLAUDE.md must
not duplicate an ADR index.

**Layer routing — route by CONTENT TYPE, then by triggering:**

Step 0: Is this explanatory rationale ("why we chose X")?
        → Yes → flag as ADR candidate; do NOT put in CLAUDE.md, rules, or skills
Step 1: Could linter/formatter/hooks enforce this with one config addition?
        → Yes → graduation candidate (not a context-layer rule)
Step 2: If removed from context, would Claude make a mistake it otherwise wouldn't?
        → No → drop it
        → Yes → route by content type:
          - Global behavioral rule (every session) → CLAUDE.md
          - Lookup/reference table for specific file types → path rule
          - Multi-step procedure (sequential, order matters) → skill

**Each concern lives in exactly ONE layer.** If a topic has both lookup content
and procedural content, split into non-overlapping aspects with distinct names.

**Skill criteria — list only if meeting ≥ 3 of 4. When uncertain, mark Y:**
1. Sequential + ordered: specific sequence required; wrong order breaks things
2. Cross-cutting: multiple files/systems, not one path glob
3. Knowledge-heavy: WHY not readable from the code
4. Rare but critical: not every session, but mistakes are costly

**Write output to `.claude/init-progress/MODULE_SLUG.md`. Keep under 200 lines —
prioritize highest-impact items.**

```
# Module Report: MODULE_PATH

## Toolchain
- Package manager: <name> (from <file>)
- Linter: <name> / none
- Formatter: <name> / none
- Hooks: <name> / none
- Type checker: <name> / none
- Build: `<exact build command>` / `<exact test command>`

## Currently enforced (already in toolchain — no action needed)
- <pattern>: enforced by <tool> <rule-name>

## Graduation candidates (could enforce, not yet configured)
- <pattern>: add <rule-name> to <tool>

## CLAUDE.md candidates (cross-cutting, needed every session)
- <exact rule text>: <why Claude would mistake without it>

## Path rule candidates
- <exact rule text> | paths: <glob> | <why this glob, why not broader>

## Skill candidates

### <kebab-case-name>
- Trigger: "<what user says>"
- Criteria: sequential=Y/N, cross-cutting=Y/N, knowledge-heavy=Y/N, rare-critical=Y/N
- Description: <one sentence: what multi-step workflow this guides through>

## ADR / docs index check
- docs/adr/README.md: <exists / missing>
- Other indexes: <list or none>
- CLAUDE.md must NOT include ADR index: <yes / no>

## Uncertainties
- <item>: <what is uncertain and why>
```

If you cannot write the file for any reason, return:
`FAILED: MODULE_SLUG — reason: <one sentence>`
Do not return DONE unless the file is fully written.

After writing, return: `DONE: MODULE_SLUG.md written (<N> lines)`

---

**After all subagents return:** verify before Phase 3:

```bash
ls .claude/init-progress/*.md
```

If a module file is missing (absent or subagent returned FAILED), append to
`.claude/init-uncertainties.md`: `- [ ] Module MODULE_PATH: not explored — review manually.`

---

## Phase 3 — Aggregation

**Bootstrap draft files before reading any report:**

```bash
printf "# CLAUDE.md\n\n" > .claude/init-draft/CLAUDE.md
printf "# Skill Scaffolds\n\n" > .claude/init-draft/skill-scaffolds.md
```

**Do not load all reports into context simultaneously.** Process one at a time.
After each report, write updated draft to disk before reading the next. This prevents
context overflow and preserves progress.

For each report in `.claude/init-progress/*.md` (excluding backup-compare-* files):

1. Read the report
2. Merge its candidates (rules below)
3. Write updated draft state to `.claude/init-draft/`
4. Read next report

**Merge rules:**

*CLAUDE.md candidates:*
- Group semantically similar candidates → keep best phrasing
- **First module:** write its rules to the draft without any contradiction check —
  the draft is empty at this point. Contradiction checks apply only when merging
  subsequent modules against already-written draft content.
- **Contradiction handling:** if Module B contradicts a rule already written from
  Module A — (1) find the rule in `.claude/init-draft/CLAUDE.md` and replace it with
  `<!-- CONTRADICTION: <topic> — both positions in uncertainties, see below -->`;
  (2) add both positions to `.claude/init-uncertainties.md` under Contradictions
- Apply decision tree: genuinely every-session? If contextual → demote to path rule

*Path rule candidates:*
- Group by domain (auth, api, ui, build, testing, etc.) → one file per domain:
  `.claude/init-draft/rules/<domain>.md`
- Merge candidates sharing the same `paths:` glob
- Collateral damage check: glob too broad → move to skill

*Graduation candidates:* de-duplicate across modules; keep as proposals (user confirms)

*Skill candidates:*
- Merge semantic duplicates; re-verify ≥ 3/4 criteria
- Survivors → append to `.claude/init-draft/skill-scaffolds.md`

**Cross-layer deduplication (run after all reports are merged, before writing
any draft file):**

Group all candidates by TOPIC. For any topic appearing in candidates for more
than one layer, apply the content-type test to choose exactly one:
- Is it explanatory rationale? → ADR candidate, remove from all context layers
- Is it a global behavioral command (always/never)? → CLAUDE.md
- Is it a lookup/reference table? → path rule
- Is it a sequential procedure? → skill

Remove the duplicate from the losing layer. If genuinely different aspects of
the same topic belong in different layers, rename them to make the distinction
explicit (e.g., "logging-conventions" for the rule, "add-new-logger" for the
skill). The goal: no user reading the output should find the same concern
addressed in two different files.

*Missing modules:* note in `.claude/init-uncertainties.md` under Missing coverage

**Write `.claude/init-uncertainties.md` (append, don't overwrite):**

```markdown
## Init Uncertainties — <YYYY-MM-DD>

### Graduation candidates (confirm before considering adding to toolchain)
- [ ] `<rule>`: Could enforce via `<tool>` rule `<rule-name>`. Worth configuring?

### Layer decisions
- [ ] `<item>`: Uncertain whether every-session or path-scoped. Your call.

### Contradictions
- [ ] `<topic>`: Module A: approach X. Module B: approach Y. Which wins?

### Skill threshold
- [ ] `<workflow>`: Meets 3/4 criteria. Frequent enough to warrant a skill?

### Missing coverage
- [ ] Module `<path>`: not explored — review manually.
```

---

## Phase 4 — Backup comparison (rebuild path only)

Skip this phase if the user chose option A.

**Cap: dispatch at most 8 comparison subagents at a time.** If the backup contains
more files, process in sequential batches of 8.

**Relevant draft file algorithm:** for each backup file, find the corresponding draft
file by matching the filename stem to a domain name in `.claude/init-draft/rules/`
(e.g., `backup/api.md` → `.claude/init-draft/rules/api.md`). If no match exists or
the domain was split/renamed, provide the full CLAUDE.md and all rule files.

**Full comparison subagent prompt** (fill `BACKUP_FILE`, `BACKUP_SLUG`, `DRAFT_FILES`):

---

Compare the backup file `BACKUP_FILE` against the new draft context layer.

Draft files provided for comparison: `DRAFT_FILES`

For each item in the backup file:

1. Check whether a semantic equivalent exists in the draft files. "Semantic equivalent"
   means the same concern is addressed — not keyword matching. A rule about "use named
   exports" and one about "avoid default exports" are semantic equivalents.

2. For items not in the draft, grep the current codebase to verify whether referenced
   tools, paths, or patterns still exist:
   ```bash
   grep -r "<tool_or_path>" . --include="*.json" --include="*.ts" --include="*.toml" -l 2>/dev/null | head -5
   ```

3. Record verdict for each item.

Write your findings to `.claude/init-progress/backup-compare-BACKUP_SLUG.md`:

```
# Backup Comparison: BACKUP_FILE

## Already covered
- <item>: matches <draft section> — "<quoted equivalent in draft>"

## Adds value → propose merge
- <item>: absent from draft. Grep confirms <tool/path> exists at <file>.
  Proposed addition to: <draft file and section>
  Proposed text: <exact text to add>

## Obsolete → skip
- <item>: references <tool/path> not found by grep (no results in codebase)

## Uncertain → flag
- <item>: <verbatim backup text>
  Closest draft section: <section>
  Question for user: <one sentence>
```

Return: `DONE: backup-compare-BACKUP_SLUG.md written`

---

**After all comparison subagents complete**, read all
`.claude/init-progress/backup-compare-*.md`. Collect all "adds value" items as
proposed additions (with exact proposed text) for Phase 6. Then re-read
`.claude/init-uncertainties.md` and mark any items now resolved by backup content.

---

## Phase 5 — Health check

**Before dispatching:** read these two sources and hold them in context:
1. `references/writing-principles.md` — full text (will be included in subagent prompt)
2. Full content of `.claude/init-draft/CLAUDE.md` and all `.claude/init-draft/rules/*.md`

**Dispatch a single health check subagent. Provide in the prompt:**
- The full writing principles text (copy it verbatim into the prompt)
- The full content of `.claude/init-draft/CLAUDE.md`
- The full content of each rule file in `.claude/init-draft/rules/`

The subagent evaluates against the health card template (health-card.md) and
**returns the health card inline as text in its reply** (no file write needed).

Main agent reviews the inline health card and fixes before Phase 6:

- Line count > 150 → cut lowest-value lines (apply litmus test per line)
- If line count still > 200 after cuts → keep as-is and present to user in Phase 6
  with warning: "Draft is N lines, above the 200-line red line."
- MUST/IMPORTANT count > 5 → soften overuse
- Any `@import` → remove
- Wrong-layer content (rules that should be CLAUDE.md or vice versa) → migrate
- Graduation candidates missed → add to uncertainties
- Rule files missing `paths:` frontmatter → add before Phase 6
- Path rules with high collateral damage → flag or move to skill candidates
- **Structural problems** (e.g., draft is near-empty after deduplication, major
  domain entirely uncovered, most content wrongly layered): do not attempt to fix
  silently. Surface as a "Structural issues" block at the top of the Phase 6
  presentation, before the draft, with a clear description of what's wrong and
  what the user should do.

---

## Phase 6 — STOP AND PRESENT

**Do not proceed to Phase 7 in this turn. End your response after presenting.**
Phase 7 runs only after the user explicitly replies to confirm.

Present in this order:

**1. Draft CLAUDE.md** — full content

**2. Draft rule files** — filename + first 5 lines preview for each

**3. Skill scaffolds** — for each: name, trigger phrase, one-paragraph description

**4. Backup additions** (rebuild path only) — present each "adds value" item:
   "This item from your backup wasn't covered in the new draft — add it? (Y/N):"

**5. Graduation candidates** — present each as a distinct action item:
   "This rule could be enforced by `<tool>` rule `<rule-name>` — worth configuring
   rather than keeping in CLAUDE.md? (Y/N):"

**6. Remaining uncertainties** — each unresolved item from `.claude/init-uncertainties.md`
   (layer decisions, contradictions, skill threshold calls, missing coverage)

Close with:

> When you're ready, reply with your changes or "apply as-is" to proceed.

**End your response here. Wait for the user's reply.**

---

## Phase 7 — Apply

**Only run after the user explicitly confirmed in Phase 6.**
If unsure, ask before writing any file.

**Step 1 — Read scaffolds and backup approvals before any cleanup:**

```bash
cat .claude/init-draft/skill-scaffolds.md    # store confirmed scaffold info in context
```

For each user-approved backup item (from Phase 6), re-read the corresponding
`.claude/init-progress/backup-compare-<slug>.md` to retrieve the exact proposed
addition text. Apply additions to the appropriate draft file.

**Step 2 — Apply user modifications** to all draft files.

**Step 3 — Write to project:**

```bash
# Copy rule files
cp .claude/init-draft/rules/*.md .claude/rules/ 2>/dev/null || true
# Write CLAUDE.md to project root
cp .claude/init-draft/CLAUDE.md ./CLAUDE.md
```

**Step 4 — Cleanup:**

```bash
rm -rf .claude/init-progress/ .claude/init-draft/
```

Note: the module exploration reports in `.claude/init-progress/` will be deleted.
Tell the user before running: "About to delete exploration reports. Move them elsewhere
first if you want to keep them."

**Step 5 — Keep uncertainties file** if any items remain unresolved:
`.claude/init-uncertainties.md` stays in place.

**Step 6 — Verify:**

```bash
wc -l CLAUDE.md                                                    # target 100-150, red line 200
grep "@import" CLAUDE.md                                           # should be empty
find . -maxdepth 4 -name "CLAUDE.md" | grep -v "^./CLAUDE.md"    # no subdirectory CLAUDE.md
```

- Each rule file has `paths:` frontmatter
- No rule file exceeds 200 lines

**Step 7 — Skill scaffold handoff:**

For each confirmed skill scaffold (from Step 1's stored info):

Check if `skill-creator` appears in your available skills. Then:

- **If available:** invoke `skill-creator` with the skill name, trigger phrasing, and
  workflow description. Do this one at a time.
- **If not available:** write a stub for each:
  ```
  .claude/skills/<kebab-name>/SKILL.md
  ---
  name: <name>
  description: <trigger phrasing>
  ---
  # <name>
  TODO: flesh out this skill using skill-creator.
  Install: npx skills add anthropics/skills --skill skill-creator -g -y
  ```
  Tell the user: "Skill stubs written — complete them with skill-creator when available."

**Do not commit.** Tell the user: "Files written — review before committing,
especially rule file `paths:` globs and CLAUDE.md line count."
