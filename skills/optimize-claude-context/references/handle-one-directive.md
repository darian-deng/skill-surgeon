# handle-one-directive

The atomic unit of the optimize-claude-context skill. All other workflows
(rebuild, feat-flow) ultimately call this operation.

References:
- `decision-tree.md` — Steps 0-3
- `writing-formats.md` — Step 5 content specs
- `linter-capabilities.md` — Step 1 methodology
- `directive.md` — atomic definition + scope rules

---

## Three Modes

Choose the mode based on the calling context. The mode determines where in the
decision tree execution begins.

| Mode | Who calls it | Starts at | What it does |
|---|---|---|---|
| `manual` | User describes a directive directly | Step 0 | Full flow; writes file |
| `feat-flow` | External workflow (directive pre-parsed, scope known) | Step 1 | Skips Step 0; writes file |
| `rebuild-execute` | rebuild Phase 6 | Step 4 | Decision already in table; writes file + updates table row status |

**feat-flow contract:** any external workflow that pre-parses its directives before
calling handle-one-directive can use this mode. The caller guarantees: directive is
already atomic, scope is already determined, source (file:lines) is known.
handle-one-directive trusts these inputs and does not re-validate atomicity or scope.
Conflict detection (Step 2) and layer routing (Step 3) still run — only Step 0
(parse/split) is skipped. The caller is responsible for ensuring atomicity upstream.

**rebuild-execute contract:** the rebuild progress table already contains the
decision result for this directive (layer assigned in Phase 4). handle-one-directive
reads the layer from the table row, skips Steps 0-3, and executes Steps 4-5 only.
After writing the file, update the table row status to `done`.

---

## Execution Flow

```
[manual]              [feat-flow]          [rebuild-execute]
    │                     │                       │
Step 0                Step 1                  Step 4
(parse/split)       (linter check)          (enrichment)
    │                     │                       │
Step 1                Step 2                      │
(linter check)      (existing check)              │
    │                     │                       │
Step 2                Step 3                      │
(existing check)    (routing)                     │
    │                     │                       │
Step 3                    │                       │
(routing)                 │                       │
    │                     │                       │
    ──────────────────────────────────────────────
                          │
                      Step 4
                    (enrichment)
                          │
                      Step 5
                  (write content per
                   writing-formats.md)
                          │
                        WRITE file
                          │
                        VERIFY
```

---

## Step 0 — Parse (manual mode only)

Apply the atomic definition from `directive.md`.

**If one directive:** assign scope; continue to Step 1.

**If multiple directives detected:** split into N directives. Call handle-one-directive
recursively for each. Collect all results before continuing. Do not proceed with a
merged directive.

Scope determination: see `directive.md` scope rules. Default to `root` when ambiguous.

---

## Step 1 — Linter Feasibility Check

Read toolchain config files for the directive's scope. Ask:

> "If this directive CAN be enforced by [linter], write the **exact rule
> configuration** and cite the **official documentation URL** (or parent page URL
> for rules without individual docs). If you cannot produce a configuration backed
> by documentation, say 'not enforceable'."

| Result | Action |
|---|---|
| Valid config + citation | **Graduate** — modify (or create) linter config; directive does NOT enter context layer; **halt** |
| Not enforceable | Continue to Step 2 |

After graduating: verify by running the linter against code that violates the rule.
See `linter-capabilities.md` for mechanisms commonly missed.
If the linter does not produce an error, the config is incorrect — do not graduate.
Revert the config change and continue to Step 2.

---

## Step 2 — Check for Existing Directives

Scan project-level only (never `~/.claude/`):
- `./CLAUDE.md`
- `./.claude/rules/*.md`
- `./.claude/skills/*/SKILL.md`

| Result | Action |
|---|---|
| New — no semantic overlap | Continue to Step 3 |
| Merge — overlapping directive with less/different info | Combine; continue to Step 3 |
| Conflict — sources contradict | **Flag CONFLICT; surface to user; halt** |

Conflict surface format:
```
CONFLICT detected
  Existing: [file:line] "<existing directive>"
  New:      "<incoming directive>"
  Resolution required before proceeding.
```

`status: stub` skill entries count as pending, not as conflicting.

---

## Step 3 — Layer Routing

Apply the layer selection rules from `decision-tree.md` Step 3 in priority order:

1. Multi-step procedure? → **Skill**
2. scope=`root`, behavioral rule? → **CLAUDE.md**
3. scope=`<package-path>`, lookup/reference? → **Path rule**
4. Explanatory rationale? → **ADR**
5. None → **deprecated**

---

## Step 4 — Content Enrichment

Before writing, gather evidence.

**Project-related directive** (references code, tools, patterns specific to this
project): use `feature-dev:code-explorer` to explore relevant code. Write based on
what actually exists in the codebase.

**External knowledge directive** (industry patterns, third-party tool conventions):
research best practices. Cite sources in the content.

**rebuild-execute mode:** read the layer assignment from the rebuild progress table
row. Read `source` field (file:lines) to know where the original content came from;
use it as the starting point for enrichment.

---

## Step 5 — Write in Target Layer's Format

See `writing-formats.md` for complete specifications. Summary:

### CLAUDE.md

- Apply all 14 writing principles from `writing-formats.md §CLAUDE.md Format`
- Imperative form, positive phrasing, one instruction per bullet
- Structure: H1/H2/H3 only; no H4+; no `@import`; commands in code fences
- Hard limit: 200 total lines; target 100-150
- Append to the relevant section, or create a new section with H2 heading
- After writing: run `wc -l ./CLAUDE.md` and report line count

### Path rule

Create or update `./.claude/rules/<domain>.md`:

```yaml
---
paths:
  - "<glob>"
when: "<one sentence: the work scenario where this content is relevant>"
---
# <Domain> Conventions
```

- Lookup/reference content only; not workflow instructions
- `paths:` frontmatter required; use CSV format if YAML list is unreliable
- `when:` required; write it at Step 3 — see confidence guidance in `decision-tree.md §Step 3`
- In rebuild-execute mode: **overwrite** the file (Phase 1 cleared originals); do not append
- After writing: confirm both `paths:` and `when:` frontmatter are present

### Skill

Write stub SKILL.md only — frontmatter with no body:

```yaml
---
name: <kebab-case>
description: <≤15 words>
status: stub
---
```

Path: `./.claude/skills/<name>/SKILL.md`

After writing: notify user to complete the skill body using `skill-creator`. Do not
auto-invoke skill-creator.

After writing: count words in description; confirm ≤15 words.

### ADR

Create `./docs/adrs/NNNN-<slug>.md` using the Nygard template from `writing-formats.md`.

Assign the next sequential ADR number. Check existing ADRs to determine next number.

Update or create `./docs/adrs/README.md` index with the new entry.

After writing: confirm Consequences section is non-empty.

---

## Verification Checklist (post-write)

Run these checks after every write, regardless of mode:

| Target | Check | Command |
|---|---|---|
| CLAUDE.md | Line count within budget | `wc -l ./CLAUDE.md` |
| Path rule | `paths:` frontmatter present | `head -6 .claude/rules/<file>.md` |
| Path rule | `when:` frontmatter present | `head -6 .claude/rules/<file>.md` |
| Skill | description ≤ 15 words | count words in description field |
| ADR | Consequences non-empty | read Consequences section |
| Any | File actually created/modified | `ls -la <target-path>` |

---

## Special Cases

### Linter graduation

Modify the existing linter config file directly. If no config file exists for the
linter, create one with the standard name for that toolchain (e.g., `eslint.config.js`,
`ruff.toml`). The directive does not appear in any context layer file.

### Skill stub

Write only the frontmatter. Do not write a body. Notify user:

> "Skill stub written at `.claude/skills/<name>/SKILL.md`. Run skill-creator to
> develop the body."

Do not auto-invoke skill-creator.

### Conflict halt

When CONFLICT is detected in Step 2: stop. Do not write anything. Surface the
conflict to the user with both sides shown. Wait for user resolution before
rerunning.

### Global CLAUDE.md

`~/.claude/CLAUDE.md` is never modified by handle-one-directive. If a
project-specific directive is found there, note it and tell the user to remove it
manually from that file.

### rebuild-execute mode table update

After successfully writing the file in rebuild-execute mode, update the
corresponding row in `.claude/rebuild-progress.md`:

- Set `status` column to `done`
- Set `final file` column to the exact path of the file written

Do this immediately after writing; do not batch updates.
