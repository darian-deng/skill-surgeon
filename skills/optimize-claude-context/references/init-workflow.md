# Init Workflow

⚠️ **Under active development.** The design below is the agreed brief;
full implementation is being developed via skill-creator.

Builds a project's Claude Code context layer from scratch using parallel subagent
deep exploration. Prioritizes quality over speed — allows large token and time budgets.

---

## Entry: check for existing context

Before exploring, check for existing artifacts:
- `./CLAUDE.md`
- `./.claude/rules/*.md`
- `./.claude/skills/*/SKILL.md`
- `./docs/adr/` or equivalent ADR directory

**If any found → stop. Ask the user:**

> I found existing context artifacts:
> - [list each found item]
>
> How do you want to proceed?
> **(A) Delete and start fresh** — remove existing artifacts, build new from scratch
> **(B) Backup and rebuild** — move existing to `./claude-context-backup/`, build
> new, then review backup items to recover what's still valuable

If intent is unclear, ask again. Never guess.

- **(A) confirmed** → delete existing artifacts → proceed to Deep Exploration
- **(B) confirmed** → move all found artifacts to `./claude-context-backup/`
  (preserve directory structure) → proceed to Deep Exploration →
  after init completes, run Backup Comparison

**If nothing found** → proceed directly to Deep Exploration.

---

## Phase 1 — Scale detection (main agent)

Read: package manifests, workspace config, top-level directory structure.

Determine:
- **Single package**: 1 subagent
- **Small monorepo** (2–4 packages): 1 subagent per package
- **Large monorepo** (5+ packages): 1 subagent per major package, cap at 8

Identify each subagent's scope (module path + primary language/stack).

---

## Phase 2 — Parallel deep exploration (N subagents, concurrent)

Each subagent receives:
- Its module path
- The shared principles (decision tree, mechanism selection, writing principles)
- The structured output schema below

**Each subagent reads:**

*Config track (read in full — does not count toward source budget):*
- Linter config (`.eslintrc*`, `eslint.config.*`, `ruff.toml`, etc.)
- Formatter config (`.prettierrc*`, `rustfmt.toml`, etc.)
- Pre-commit hooks (`.husky/`, `.pre-commit-config.yaml`, etc.)
- Type checker config (`tsconfig*.json`, `mypy.ini`, etc.)

*Source track (~20 files):*
- Entry points
- Representative files across major directories
- Test files (understand testing patterns)

*Docs track:*
- `README.md`, `CONTRIBUTING.md`
- Check whether `docs/adr/README.md` or equivalent index already exists

**Each subagent produces a structured report:**

```
### Module: <path>

#### Toolchain
- Linter: <name or none>
- Formatter: <name or none>
- Hooks: <name or none>
- Type checker: <name or none>

#### Currently enforced (graduation already done — note only, no action needed)
- <pattern>: enforced by <rule/tool>

#### Graduation candidates (could be enforced, not yet configured)
- <pattern>: could be enforced by adding <rule> to <tool>

#### Rule candidates (context-layer only, passed mechanism filter)
- <rule text>: <rationale — why Claude would make a mistake without it>

#### Skill candidates
For each candidate, answer all four criteria:
1. Sequential + ordered: steps must happen in sequence, order matters
2. Cross-cutting: involves multiple files/systems not captured by one path glob
3. Knowledge-heavy: requires knowing WHY, not derivable from reading the code
4. Rare but critical: not every session, but when needed it must be right
Only list candidates that meet at least 3 of 4 criteria.

#### Existing index check
- docs/adr/README.md: <exists / missing>
- Other docs indexes: <list>
Any CLAUDE.md section that would duplicate an existing index → drop candidate.

#### Uncertainties
- <item>: <what is uncertain and why>
```

---

## Phase 3 — Aggregation (main agent)

Merge all subagent reports:
1. De-duplicate rule candidates (same concern expressed differently → keep best phrasing)
2. De-duplicate graduation candidates
3. Resolve conflicts (rules that contradict each other → flag for user)
4. Build draft:
   - `CLAUDE.md` (cross-cutting facts + rules that pass mechanism filter)
   - `.claude/rules/*.md` (path-scoped rules, one file per domain)
   - Skill scaffold list (names + descriptions for candidates)
5. Write uncertainties to `.claude/init-uncertainties.md`:

```markdown
## Init Uncertainties — <date>

Items that need user confirmation before the context layer is finalized.

- [ ] [Module X] Should the recording pipeline workflow become a skill?
      Meets 3/4 criteria (sequential, cross-cutting, knowledge-heavy).
      Not sure: is it used frequently enough to warrant skill overhead?

- [ ] [CLAUDE.md] try-catch rule: ESLint `no-restricted-syntax` could enforce
      this. Recommend graduating — confirm?
```

---

## Phase 4 — Backup comparison (rebuild path only)

If this is a rebuild, review each item in `./claude-context-backup/` in parallel
subagent batches.

Each item gets a verdict:
- **already covered** — equivalent rule/fact is in the new draft
- **adds value → merge** — not in draft, worth adding
- **obsolete → skip** — references removed tools/patterns, do not carry forward
- **uncertain → flag** — unclear if still relevant, add to uncertainties file

After comparison, re-check `.claude/init-uncertainties.md` — some uncertainties
from Phase 3 may now be resolved by backup content.

---

## Phase 5 — Health check (single subagent)

Run audit-report on the draft output.
Main agent reviews findings and refines draft before presenting to user.

---

## Phase 6 — User review

Present to user:
- Draft `CLAUDE.md`
- Draft rule files
- Skill scaffold proposals (name + description + body outline for each)
- `.claude/init-uncertainties.md`

Wait for confirmation. User can modify any item.

For confirmed skill scaffolds: hand off to `skill-creator` for full creation
workflow (eval loop, test cases, description optimization).

---

## Phase 7 — Apply

Write all confirmed files. Do not commit — user reviews the result.

Verify after writing:
- CLAUDE.md line count within budget (target 100–150, red line 200)
- All `paths:` frontmatter in rule files uses correct format
- No `@import` in CLAUDE.md
- No subdirectory CLAUDE.md created
- `.claude/init-uncertainties.md` exists if any uncertainties remain
