# optimize-claude-context — System Design

## 1. Purpose

optimize-claude-context manages a project's AI context layer: the collection of
files that help Claude work correctly on a specific project. The goal is that every
piece of project-specific knowledge Claude should know is captured, correctly
classified, and stored in exactly one place.

Three commands:
- **handle-one-directive** — process a single knowledge point
- **rebuild** — reprocess all existing project knowledge from scratch
- **audit** — evaluate the health of the current context layer with a 0-100 score

---

## 2. Core Concepts

### 2.1 Directive

A **directive** is a single piece of project-specific knowledge that Claude should
know when working on this project.

**Atomic definition:** one independently routable knowledge point. Removing it does
not affect the meaning of any other directive.

```
✅ "This project uses pnpm as the package manager"
✅ "The Electron main process uses four-layer architecture: index → app → services → utils"
✅ "Never use try-catch; use radash/tryit instead"
❌ "Project uses pnpm and main process uses four-layer architecture" (two directives merged)
```

**Required property — Scope:** every directive must have a scope.

| Scope value | Meaning |
|---|---|
| `root` | Applies to the entire project / all packages |
| `<package-path>` | Applies to a specific package, e.g. `apps/plaud-desktop/` |

Scope determines the target layer and which toolchain to evaluate. In a monorepo,
a directive applying to all packages has scope `root`; one specific to a package
has scope `apps/x/`.

### 2.2 The Four Layers

Each directive lives in exactly one layer.

| Layer | Role | Loaded |
|---|---|---|
| **CLAUDE.md** | Behavioral rules — what Claude must always know and do | Every session, unconditionally |
| **Path rule** | Lookup/reference — conventions for specific file types | When touching files matching the glob |
| **Skill** | Workflow guide — multi-step procedures to execute | When user intent matches |
| **ADR** | Decision rationale — why a non-obvious decision was made | On demand by humans; never auto-loaded |

**Mutual exclusivity:** the same directive cannot appear in two layers. If a domain
has both "always use structuredLogger" (behavioral → CLAUDE.md) and "how to add a
new log category" (multi-step procedure → skill), these are two distinct directives
covering different aspects.

### 2.3 Layer Selection Rules

| Condition | Target layer |
|---|---|
| Multi-step procedure (sequential, order matters, cross-file) | Skill — this takes precedence over all other conditions |
| scope=root, behavioral rule (must-always-do) | CLAUDE.md |
| scope=`<package-path>`, lookup/reference content | Path rule with `paths:` glob |
| Explanatory rationale ("why we chose X") | ADR |
| None of the above | deprecated |

**Key tiebreaker:** multi-step procedure always routes to skill regardless of scope.
A cross-cutting procedure (applies to all packages) is still a skill, not CLAUDE.md.

ADRs always live at `<project-root>/docs/adrs/` regardless of monorepo structure.

---

## 3. Decision Tree

### Step 0 — Parse

1. One directive or multiple? Apply the atomic definition. If multiple, split and
   process each independently (handle-one-directive calls itself recursively).
2. Determine scope: which part of the project does this apply to? Cross-cutting → `root`.
   Package-specific → `<package-path>`.

### Step 1 — Linter feasibility check

Identify the language and primary linter for the directive's scope. Read the toolchain
config files for that scope (`package.json`, `pyproject.toml`, `Cargo.toml`, etc.).

**The question to ask:**
> "If this directive CAN be enforced by [linter], write the exact rule configuration
> and cite the official documentation URL (or parent page URL for rules without
> individual docs). If you cannot produce a configuration backed by documentation,
> say 'not enforceable'."

This forces production of a verifiable artifact. A valid config with citation →
graduate. No valid config → continue to Step 2.

On graduation: directly modify (or create) the linter configuration file. The
directive does NOT enter the context layer.

Refer to `linter-capabilities.md` for a reference of linter mechanisms that AI
commonly misses (e.g., `no-restricted-syntax` for AST-based bans).

### Step 2 — Check for existing directives

Scan `.claude/rules/*.md` and `.claude/skills/*/SKILL.md` (project-level only,
not global `~/.claude/`) and `./CLAUDE.md` for semantic overlap.

| Result | Action |
|---|---|
| New | No semantic overlap — continue to Step 3 |
| Merge | Overlapping directive exists with less/different info — combine, continue |
| Conflict | Sources contradict — flag CONFLICT, surface to user, halt |

### Step 3 — Layer routing

Apply the layer selection rules from Section 2.3 in this order:
1. Is this a multi-step procedure? → Skill
2. scope=root + behavioral rule? → CLAUDE.md
3. scope=`<package-path>` + lookup/reference? → Path rule
4. Explanatory rationale? → ADR
5. None → deprecated

### Step 4 — Content enrichment

Before writing, gather evidence:
- **Project-related directive**: use `feature-dev:code-explorer` to explore
  relevant code. Write based on what actually exists.
- **External knowledge directive**: research industry best practices. Cite sources.

### Step 5 — Write in target layer's format

See `writing-formats.md` for complete specifications.

Hard constraints:
- CLAUDE.md: 14 writing principles, imperative form, target 100-150 lines total
- Path rule: `paths:` YAML frontmatter required
- Skill: description ≤15 words (hard limit); write stub SKILL.md with frontmatter only;
  hand off to skill-creator for body development (do not auto-invoke skill-creator)
- ADR: Nygard template (Context / Decision / Consequences / Alternatives Considered);
  Consequences must not be empty

---

## 4. handle-one-directive

The atomic unit. All other workflows call this.

### 4.1 Inputs

```
directive_text   required  Natural language description of the directive
source           optional  Origin file + line range (always known in rebuild-execute mode)
scope_hint       optional  Suggested scope (known in rebuild, inferred in Step 0 otherwise)
mode             required  "manual" | "feat-flow" | "rebuild-execute"
```

### 4.2 Three Call Modes

**manual:** User describes a directive. handle-one-directive runs the full flow
(Steps 0-7). Writes file directly.

**feat-flow (pre-parsed):** Any external workflow that guarantees the directive is
already atomic, scope is already determined, and source is known can use this mode.
handle-one-directive starts at Step 1 (skips Step 0 parse/split).
Writes file directly. The caller is responsible for ensuring atomicity.

**rebuild-execute:** Called from rebuild Phase 6. Decision tree result is already
in the rebuild table (Steps 0-3 were run in Phase 4). handle-one-directive starts
at Step 4 (content enrichment). Writes the actual file. Updates the table row
status to `done`.

### 4.3 Execution Flow

```
                    [manual]          [feat-flow]    [rebuild-execute]
                       │                  │                 │
                   Step 0             Step 1            Step 4
                  (parse)           (linter check)   (enrichment)
                       │                  │                 │
                   Step 1                 │                 │
                  (linter)                │                 │
                       │                  │                 │
                   Step 2                 │                 │
               (existing check)           │                 │
                       │                  │                 │
                   Step 3                 │                 │
                 (routing)                │                 │
                       │                  │                 │
                   ─────────────────────────────────────────
                                          │
                                      Step 4
                                    (enrichment)
                                          │
                                      Step 5
                                    (generate content
                                     per writing-formats.md)
                                          │
                                        WRITE
                                        file
                                          │
                                       VERIFY
                      paths: frontmatter | description ≤15w | CLAUDE.md lines
```

**On split (Step 0):** multiple directives detected → call self recursively for each
before continuing.

**On CONFLICT (Step 2):** flag, surface to user, halt. Do not write anything.

**On skill routing (Step 3/5):** write stub SKILL.md with:
```yaml
---
name: <kebab-case>
description: <≤15 words>
status: stub
---
```
Notify user to complete with skill-creator. Step 2's existing-directive scan treats
`status: stub` entries as pending (not as conflicting existing directives).

---

## 5. Rebuild Workflow

### 5.1 Rebuild Progress Table

State persists in `.claude/rebuild-progress.md`.

**Non-ADR table:**

| directive | sources (file:lines) | conflict | stale | decision result | confidence | status | final file |
|---|---|---|---|---|---|---|---|

Column definitions:
- **directive**: atomic knowledge point (one sentence, one concept)
- **sources**: all files with line ranges containing this directive or semantic overlap
- **conflict**: Y + conflicting content shown inline
- **stale**: Y if grep cannot find referenced file paths or tool names in current codebase
  - Grep targets: file paths (patterns like `src/**/*.ts`), tool/library names from code blocks
  - File paths: checked via `find`. Tool names: checked in `package.json` + codebase.
- **decision result**: Graduate / CLAUDE.md / path-rule:\<glob\> / skill:\<name\> / ADR / deprecated
- **confidence**: H / M / L — defined below
- **status**: pending / in-progress / done / deprecated / conflict-blocked
- **final file**: exact target file path

**Confidence calibration:**
- **H**: routing is unambiguous — only one valid layer given the decision tree
- **M**: two plausible layers exist and one was chosen; OR linter check result was uncertain
- **L**: directive is ambiguous in scope or type; a different reasonable agent might choose differently

**ADR table** (separate section in same file):

| ADR file | meets definition | covered by directive | replaceable by code comment | action | target path |
|---|---|---|---|---|---|

### 5.2 Execution Phases

**Phase 0 — Dependency check**

```bash
ls ~/.claude/skills/skill-creator/ 2>/dev/null \
  || ls ~/.agents/skills/skill-creator/ 2>/dev/null \
  || (check available_skills context as fallback)
ls ~/.claude/skills/skill-surgeon/ 2>/dev/null \
  || ls ~/.agents/skills/skill-surgeon/ 2>/dev/null \
  || (check available_skills context as fallback)
```

**Phase 1 — Backup**

Paths to back up (copy, then delete originals):
- `./CLAUDE.md`
- `./.claude/rules/`
- `./.claude/skills/` (project-level skills only)
- `./docs/adrs/`

Destination: `./claude-context-backup/` preserving structure.

If `./claude-context-backup/` already exists: **abort and warn user.** Do not
overwrite a previous backup automatically.

Global `~/.claude/CLAUDE.md` is NOT backed up here — it is read in Phase 2 but
never automatically modified.

**Phase 2 — Collection (sequential, writes to table as it reads)**

Subagent A (non-ADR) processes in order:
1. `~/.claude/CLAUDE.md` (global) — for each directive: is it truly global (applies
   to all projects) or project-specific? Project-specific directives enter the
   non-ADR table with note "source: global CLAUDE.md — needs manual removal"
2. `<project-root>/CLAUDE.md`
3. `./.claude/rules/*.md`
4. Any subdirectory CLAUDE.md files

Each extracted directive is written to `.claude/rebuild-progress.md` immediately.
Dynamic batching: if a directory has many large files, process in sub-batches.

Subagent B (ADR): all files in `docs/adrs/` → write to ADR table.

**Phase 3 — Domain grouping + deduplication**

Main agent reads complete table. Groups directives covering the same domain.

Domain definition: directives are in the same domain if they share the same
functional area (auth, logging, API, build, testing, error-handling, architecture,
etc.) AND the same scope. Over-splitting into more domains is always safe (only
affects efficiency, not correctness).

Per domain group:
- Consistent sources → merge into one row, all sources listed
- Contradicting sources → mark CONFLICT + show both positions
- Stale detection → grep per definition above, mark STALE

**Phase 4 — Decision tree evaluation (domain-grouped subagent batches)**

NOTE: This phase evaluates decisions and writes results to the table. It does NOT
call handle-one-directive and does NOT write any context layer files.

Group table by domain. For each batch of 3-4 directives from the same domain,
one subagent:
- Applies decision tree Steps 0-3 for each directive
- Uses `feature-dev:code-explorer` to explore relevant code (shares one exploration
  per domain batch, improving efficiency)
- Writes: decision result + confidence back to each table row

**Phase 5 — User alignment**

Present `.claude/rebuild-progress.md`. User must resolve:
- All CONFLICT rows (required before proceeding)
- All L (low confidence) rows

User may also adjust any decision result. After user confirms → proceed to Phase 6.

**Phase 6 — Execution (non-ADR)**

For each confirmed, non-deprecated table row:
- Call handle-one-directive (rebuild-execute mode)
- Batches: 3-4 directives from the same domain per subagent

rebuild-execute mode starts at Step 4 (enrichment), reads the already-decided layer
from the table row, writes the actual file, updates row status to `done`.

**Phase 7 — ADR processing** (after Phase 6 complete)

For each ADR in the ADR table:
1. Meets industry ADR definition (non-obvious architectural decision, explains why)?
   If not → deprecated
2. Content already covered by a Phase 6 directive? → deprecated (directive is sufficient)
3. Can be replaced by a code comment in the relevant code? → deprecated; write the
   suggested comment to the table as a note for the user
4. Valid and necessary → archive to `<project-root>/docs/adrs/`; update or create
   `docs/adrs/README.md` index

**Phase 8 — Final verification + cleanup**

User reviews the new context layer.

If directives were migrated from global `~/.claude/CLAUDE.md`: display which ones,
tell user to manually remove them. Rebuild does not touch the global file.

After user confirms:
```bash
rm .claude/rebuild-progress.md
```
`./claude-context-backup/` stays unless user explicitly removes it.

---

## 6. Audit Workflow

Evaluates health of the current context layer. Writes one file (`audit-report.md`)
but does not modify any context layer file. User decides which improvements to
pursue; each is then handled by handle-one-directive.

### 6.1 Scoring (0-100)

Scoring criteria and deduction values are defined in `writing-formats.md §Health Card`.
Category scores: Layer compliance / Toolchain efficiency / Content freshness / Format compliance.

(Audit-workflow.md references writing-formats.md for the authoritative scoring table.
No duplication.)

### 6.2 Audit Execution

1. Scan:
   - `./CLAUDE.md`, `.claude/rules/*.md`, `.claude/skills/*/SKILL.md` (project-level)
   - `./docs/adrs/`
   - `~/.claude/CLAUDE.md` (global) — check for project-specific directives
     mistakenly placed here; each found = -5 deduction
2. For each directive: apply decision tree check (no writing); compare actual layer
   to decision tree result
3. Check linter graduation opportunities (linter-capabilities.md methodology)
4. Run stale grep checks per stale definition in Section 5.1
5. Check ADRs against definition + code comment replaceability
6. Generate `.claude/audit-report.md`

### 6.3 Output: `.claude/audit-report.md`

```markdown
## Context Layer Audit — <project> — <date>

### Score: 74/100

| Category | Score |
|---|---|
| Layer compliance | 85/100 |
| Toolchain efficiency | 55/100 |
| Content freshness | 90/100 |
| Format compliance | 70/100 |

### Priority Actions (high impact first)

[1] LINTER_GRADUATION (+3): "禁止 try-catch" can be enforced by ESLint
    `no-restricted-syntax: [{selector: "TryStatement"}]`
    Docs: https://eslint.org/docs/rules/no-restricted-syntax

[2] WRONG_LAYER (+5): "logging-setup procedure" is a path rule but is a multi-step
    workflow — route to skill.

[3] STALE (+5): auth-conventions.md references src/services/auth/oldModule.ts
    — file not found by find.

### How to act
Call handle-one-directive for each item you want to improve.
```

---

## 7. Skill Architecture

### 7.1 File Structure

```
skills/optimize-claude-context/
├── SKILL.md                        # router: 3 commands
└── references/
    ├── directive.md                # directive definition + scope rules
    ├── decision-tree.md            # complete decision tree
    ├── linter-capabilities.md      # linter mechanism reference
    ├── handle-one-directive.md     # atomic operation
    ├── rebuild-workflow.md         # rebuild process
    ├── audit-workflow.md           # audit process + scoring
    └── writing-formats.md          # writing spec per layer (replaces writing-principles.md)
```

Deleted from current version:
- `init-workflow.md`
- `audit-report-workflow.md`
- `optimize-workflow.md`
- `update-rule-workflow.md`
- `writing-principles.md` (content merged into `writing-formats.md`)
- `health-card.md` (content merged into `audit-workflow.md`)

### 7.2 SKILL.md Router

Three commands with distinct signals:

```
handle-one-directive
  Signals: "add a directive", "I want Claude to know X", "update this rule",
           "change this skill", "evaluate this", "这个要加进去", "帮 Claude 记住",
           feat-flow stage 6 calls
  → references/handle-one-directive.md

rebuild
  Signals: "rebuild", "重建", "reprocess everything", "context layer 重写",
           "start fresh with directives"
  → references/rebuild-workflow.md

audit
  Signals: "audit", "health check", "评分", "score my context",
           "what's wrong with my CLAUDE.md", "how good is my context"
  → references/audit-workflow.md
```

### 7.3 Shared Foundations

All three commands depend on:
- `directive.md` — the concept
- `decision-tree.md` — the routing logic
- `linter-capabilities.md` — the linter check methodology
- `writing-formats.md` — how to write each artifact type
