# Init Workflow

Builds a project's Claude Code context layer from scratch using parallel subagent
deep exploration. Optimized for quality — expects large token and time budgets.

## Flow at a glance

```
Phase 0 — Check for existing artifacts
  → None found       → Phase 1
  → Found, choice A  → delete → Phase 1
  → Found, choice B  → backup to ./claude-context-backup/ → Phase 1 → Phase 4

Phase 1 — Scale detection → N module scopes + create working dirs
Phase 2 — N parallel subagents → N reports in .claude/init-progress/
Phase 3 — Read reports one at a time, write draft to .claude/init-draft/ + uncertainties
Phase 4 — (rebuild only) Parallel backup comparison → .claude/init-progress/backup-compare-*.md
Phase 5 — Health check subagent → draft refinements
Phase 6 — STOP. Present to user. Wait for explicit confirmation.
Phase 7 — Apply confirmed files only after Phase 6 confirmation received.
```

---

## Phase 0 — Check for existing artifacts

Scan before any exploration:

```bash
find . -maxdepth 1 -name "CLAUDE.md" 2>/dev/null
find .claude/rules -name "*.md" 2>/dev/null
find .claude/skills -name "SKILL.md" 2>/dev/null
ls docs/adr/ 2>/dev/null | head -5
```

**If nothing found:** proceed to Phase 1.

**If anything found:** stop and ask the user:

> I found existing context artifacts:
> - [list each: e.g., CLAUDE.md (87 lines), .claude/rules/api.md, .claude/skills/deploy/]
>
> How do you want to proceed?
> **(A) Delete and start fresh** — remove all existing artifacts, build new from scratch
> **(B) Backup and rebuild** — move existing to `./claude-context-backup/`, build
> new, then compare backup items to recover anything still valuable
>
> If unsure, B is safer — nothing is deleted.

- **(A) confirmed:** confirm once ("About to delete: [list]. Proceed?") → delete → Phase 1
- **(B) confirmed:** move all found artifacts to `./claude-context-backup/` preserving
  directory structure → Phase 1 (backup comparison happens in Phase 4, after draft exists)
- **Unclear:** ask again. Never guess.

---

## Phase 1 — Scale detection

Read these files to understand project structure:
- `pnpm-workspace.yaml`, `go.work`, `turbo.json`, `nx.json`, `lerna.json`
- `package.json` (workspaces field)
- Top-level `ls -la`

Determine subagent count and scope:

| Scale | N subagents |
|---|---|
| Single package | 1 |
| Small monorepo (2–4 packages) | 1 per package |
| Large monorepo (5+ packages) | 1 per major package, cap at 8 |
| Monorepo with `apps/` + `packages/` split | 1 per app, 1–2 for all shared packages |

For each scope record: module path + primary stack (from manifest).

**Module slug convention:** replace all slashes in the module path with hyphens,
lowercase. Example: `apps/plaud-desktop` → `apps-plaud-desktop`.

Create working directories now:

```bash
mkdir -p .claude/init-progress
mkdir -p .claude/init-draft/rules
```

---

## Phase 2 — Parallel deep exploration

Dispatch **all N subagents concurrently in the same turn.** Do not wait for one
before starting the next.

**Full subagent prompt** (fill `MODULE_PATH`, `STACK`, `N_MODULES`, `MODULE_SLUG`):

---

You are exploring module `MODULE_PATH` (stack: `STACK`) as part of a parallel
codebase scan. `N_MODULES` modules are running in parallel.

Your output must be written to `.claude/init-progress/MODULE_SLUG.md`.

**TRACK A — Config files (read every config file in FULL — no truncation):**
Linter: `eslint.config.*`, `.eslintrc*`, `ruff.toml`, `pyproject.toml [tool.ruff]`,
`clippy.toml`. Formatter: `.prettierrc*`, `rustfmt.toml`, `.editorconfig`. Hooks:
`.husky/*`, `.pre-commit-config.yaml`. Type checker: `tsconfig*.json`, `mypy.ini`,
`pyrightconfig.json`. Build: `Makefile`, `justfile`, `package.json` scripts section.

Do NOT truncate these files. Linter rules are scattered — a partial read causes
wrong conclusions about what is already enforced.

**TRACK B — Source files (~20 files):**
Entry points → largest files per major directory → 2–3 test files → README.md,
CONTRIBUTING.md. While reading source, actively look for multi-step workflows
where a developer must touch multiple files in a specific order (these are skill
candidates).

**Also check:** does `docs/adr/README.md` (or `docs/decisions/README.md`) exist?
Record it — if so, CLAUDE.md should not include an ADR index.

**Mechanism selection filter — apply to every candidate:**
Step 1: Could linter/formatter/hooks enforce this with one config addition, even
        if not currently configured?
        → Yes → graduation candidate (not a context-layer rule)
Step 2: If removed from context, would Claude make a mistake it otherwise wouldn't?
        → No → drop it
        → Yes → decision tree:
          - Every session → CLAUDE.md candidate
          - Path-triggerable, low collateral damage → path rule candidate
          - Semantic trigger only → skill candidate

**Skill candidate criteria — list only if meeting ≥ 3 of 4. When uncertain, mark Y:**
1. Sequential + ordered: steps must happen in a specific sequence; wrong order breaks things
2. Cross-cutting: touches multiple files/systems not capturable by one path glob
3. Knowledge-heavy: requires WHY that is not readable from the code itself
4. Rare but critical: not needed in every session, but mistakes are costly

**Write your output to `.claude/init-progress/MODULE_SLUG.md` in this exact format.
Keep total output under 200 lines — prioritize highest-impact items per section.**

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
- Description: <one sentence: what multi-step workflow this guides the agent through>

## ADR / docs index check
- docs/adr/README.md: <exists / missing>
- Other indexes: <list or none>
- CLAUDE.md must NOT include ADR index: <yes / no>

## Uncertainties
- <item>: <what is uncertain and why>
```

After writing the file, return exactly this line to the main agent:
`DONE: MODULE_SLUG.md written (<N> lines)`

---

**After all subagents return:** verify files exist before Phase 3.

```bash
ls .claude/init-progress/*.md
```

If a module's file is missing despite the subagent reporting success, treat it
as a failure: add "Module MODULE_PATH: file not written — manual review needed"
to `.claude/init-uncertainties.md`.

---

## Phase 3 — Aggregation

**Do not load all reports into context simultaneously.** Process one report at a
time. After processing each report, write the current draft state to disk before
reading the next report. This prevents context overflow and preserves progress
if something fails.

For each report in `.claude/init-progress/*.md`:

1. Read the report
2. Merge its candidates into the draft (see rules below)
3. Write current state of affected draft files to `.claude/init-draft/`
4. Read the next report

**Merge rules:**

*CLAUDE.md candidates:*
- Group semantically similar candidates → keep best phrasing (imperative, positive, with rationale)
- Contradictions across modules (A says "use X", B says "avoid X") → add to uncertainties
- Apply decision tree: genuinely every-session? If contextual → demote to path rule

*Path rule candidates:*
- Group by domain area (auth, api, ui, build, testing, etc.) → one file per domain
- Merge candidates sharing the same `paths:` glob
- Collateral damage check: if glob fires on many unrelated changes → move to skill

*Graduation candidates:*
- De-duplicate across modules
- Keep as proposals — do not modify linter config; user must confirm

*Skill candidates:*
- Merge semantic duplicates across modules
- Re-verify ≥ 3/4 criteria after cross-module view
- Survivors → skill scaffolds list in `.claude/init-draft/skill-scaffolds.md`

*Missing modules:*
- If a module's report was absent, note: "Module X: not explored" in uncertainties

**Write `.claude/init-uncertainties.md`:**

```markdown
## Init Uncertainties — <YYYY-MM-DD>

### Graduation candidates (confirm before removing from CLAUDE.md)
- [ ] `<rule>`: Could enforce via `<tool>` rule `<rule-name>`. Worth adding?

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

Read backup at `./claude-context-backup/`. Dispatch parallel subagent batches —
one batch per backup file. Each comparison subagent receives the backup file
content and the relevant draft file(s) from `.claude/init-draft/` to compare.

Each comparison subagent writes its findings to
`.claude/init-progress/backup-compare-<backup-slug>.md` using this format:

```
# Backup Comparison: <backup file path>

## Already covered
- <item>: matches <draft section> — "<quoted equivalent in draft>"

## Adds value → propose merge
- <item>: absent from draft. Grep confirms <tool/path> still exists.
  Proposed addition to: <draft file and section>

## Obsolete → skip
- <item>: references <tool/path> not found in current codebase (grep: no results)

## Uncertain → flag
- <item>: <verbatim backup text>. Closest draft section: <section>. Question for user: <one sentence>
```

**After all comparison subagents complete**, read all
`.claude/init-progress/backup-compare-*.md` files. Collect all "adds value"
items as proposed additions for Phase 6. Then re-read
`.claude/init-uncertainties.md` and mark any items now resolved by backup content.

---

## Phase 5 — Health check

Dispatch a single health check subagent. Provide it:
- Full content of `.claude/init-draft/CLAUDE.md`
- List of draft rule files (filenames + line counts)
- The writing principles from `references/writing-principles.md`

The subagent evaluates against the health card template (health-card.md) and
returns a health card. Main agent reviews and fixes before Phase 6:

- Line count > 150 → identify lowest-value lines to cut (apply litmus test per line)
- If line count still > 200 after cuts → do not fix further; present to user in
  Phase 6 with warning: "Draft is N lines, above the 200-line red line. Please
  review for additional cuts before applying."
- MUST/IMPORTANT count > 5 → soften overuse
- Any `@import` → remove
- Graduation candidates missed by subagents → add to uncertainties
- Path rules with high collateral damage → flag or move to skill candidates

---

## Phase 6 — STOP AND PRESENT

**Do not proceed to Phase 7 in this turn. End your response after presenting.**
Phase 7 runs only after the user explicitly replies to confirm.

Present in this order:

**1. Draft CLAUDE.md** — full content

**2. Draft rule files** — filename + first 5 lines preview for each

**3. Skill scaffolds** — for each: name, trigger phrase, one-paragraph description

**4. Backup additions** (rebuild path only) — "These items from your backup were
not in the new draft. Add each? Review and decide:"

**5. Open uncertainties** — each item from `.claude/init-uncertainties.md`
requiring user decision

Close with:

> When you're ready, reply with your changes or "apply as-is" to proceed.

**End your response here. Wait for the user's reply.**

---

## Phase 7 — Apply

**Only run this phase after the user explicitly confirmed in Phase 6.**
If unsure whether Phase 6 was completed, ask before writing any file.

Apply user modifications to draft files. Write in this order:

1. `.claude/rules/*.md` — copy from `.claude/init-draft/rules/`
2. `./CLAUDE.md` — copy from `.claude/init-draft/CLAUDE.md` to project root
3. Clean up: `rm -rf .claude/init-progress/ .claude/init-draft/`
4. Keep `.claude/init-uncertainties.md` if any unresolved items remain

**Do not commit.** Tell the user: "Files written — review before committing,
especially rule file `paths:` globs and CLAUDE.md line count."

**Verify after writing:**

```bash
wc -l CLAUDE.md                                                    # target 100-150, red line 200
grep -r "@import" CLAUDE.md                                        # should be empty
find . -maxdepth 4 -name "CLAUDE.md" | grep -v "^./CLAUDE.md"    # no subdirectory CLAUDE.md
```

- Each rule file has `paths:` frontmatter
- No rule file exceeds 200 lines
- `.claude/init-uncertainties.md` exists if unresolved items remain

**For confirmed skill scaffolds:** after verification, hand off to `skill-creator`
one at a time. Provide: skill name, trigger phrasing, and workflow description.
