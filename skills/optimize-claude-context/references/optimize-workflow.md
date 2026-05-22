# Audit-Refactor Workflow

Full procedure for initializing or optimizing a project's Claude Code context
layer.

## Phase 1 — Bootstrap check

If no `CLAUDE.md` exists at the project root:

1. Suggest the user run Claude Code's built-in `/init` command first. `/init`
   analyzes the codebase and generates a starter CLAUDE.md with build commands,
   test instructions, and detected conventions.
2. If the user prefers to skip `/init`, proceed directly — Phase 2 will build
   from scratch using discovery.
3. If the user just ran `/init`, treat its output as a hypothesis to audit —
   skip suggesting `/init` again and go straight to Phase 2.

If `CLAUDE.md` already exists, proceed to Phase 2.

**Small projects (< 10 source files):** Skip the subagent in Phase 2. Use the
main agent to Read/Glob the handful of files directly. The CLAUDE.md target can
be well under 100 lines for small projects — do not pad to reach the target.

## Phase 2 — Deep exploration (subagent)

Unless the Phase 1 small-project exception applies, dispatch a subagent to
explore the codebase.

### Must-read files (config / metadata)

- Package manifests: `package.json`, `Cargo.toml`, `pyproject.toml`, `go.mod`,
  `mix.exs`, `Gemfile`, etc.
- Lockfiles: `pnpm-lock.yaml`, `bun.lockb`, `yarn.lock`, `package-lock.json`,
  `uv.lock`, `poetry.lock`, `Cargo.lock`, `go.sum`, etc. Identify the package
  manager — naming it explicitly has ~160x agent behavior leverage.
- Build/task config: `Makefile`, `justfile`, `turbo.json`, `nx.json`,
  `lerna.json`, workspace files (`pnpm-workspace.yaml`, `go.work`).
- Linter/formatter config: `.eslintrc*`, `eslint.config.*`, `.prettierrc*`,
  `ruff.toml`, `pyproject.toml [tool.ruff]`, `rustfmt.toml`, `.editorconfig`.
  **Read these in full — never truncate.** Rules are scattered throughout and
  a truncated read will miss existing enforcement, producing false graduation
  proposals or, worse, redundant CLAUDE.md rules that shadow linter coverage.
- Type checker config: `tsconfig*.json`, `mypy.ini`, `pyrightconfig.json`.
- CI config: `.github/workflows/`, `.gitlab-ci.yml`, `Jenkinsfile`.
- Existing agent config: `.claude/`, `.cursor/`, `.cursorrules`,
  `.github/copilot-instructions.md`, `AGENTS.md`. Note existence but do NOT
  import content — this skill is Claude-only.
- `README.md`, `CONTRIBUTING.md` — commands and conventions often live here.
- **Non-root CLAUDE.md detection:** Glob `**/CLAUDE.md` and check for
  `.claude/CLAUDE.md`. Flag any CLAUDE.md not at `./CLAUDE.md` as a
  consolidation candidate.
- **Rule frontmatter check:** For each `.claude/rules/*.md`, note whether it
  has a `paths:` field. Rules without `paths:` load unconditionally.
- **Existing index check:** Before assuming CLAUDE.md needs an ADR or docs
  index, check whether `docs/adr/README.md`, `docs/decisions/README.md`, or
  equivalent already exists. If it does, any CLAUDE.md section duplicating
  that index is a drop candidate — not a content to carry forward.
- **Skill deep-read:** For each `.claude/skills/*/SKILL.md`, read the full
  file and check: (a) does the `description` trigger still accurately reflect
  what the skill does and when it should fire, given the current project? (b)
  does the body reference tools, paths, or workflows that no longer exist in
  the codebase? (c) is the skill's intended trigger better expressed as a
  path-scoped rule (low collateral damage possible) — or vice versa, is a rule
  that fires too broadly better moved here?

### Source sampling

- Entry point files (`src/index.*`, `main.*`, `app.*`).
- A representative sample of source files across major directories.
- Test files to understand testing patterns.
- **Skill candidate hunting:** While reading source, actively look for
  multi-step workflows that recur but can't be path-triggered — onboarding
  sequences, migration procedures, cross-cutting debugging patterns, release
  checklists, domain-specific code-generation flows. These are skill
  candidates. If you find none after reading ~10 source files, state that
  explicitly rather than silently skipping this step.

### Monorepo sampling strategy

For monorepos with multiple workspace members:

1. List all workspace members from the workspace config file.
2. For each member, read at least 1 config file + 1 source file.
3. Distribute the ~30-file budget proportionally across members.
4. Identify which members have distinct stacks that warrant separate
   paths-scoped rules.

### Budget and confidence

Explore up to ~30 distinct files or until 80% confidence that the objective
facts are captured, whichever comes first.

If confidence remains below 80% after 30 files, list the gaps to the user and
ask whether to expand sampling or proceed with known gaps.

### Ground truth verification

**Treat existing CLAUDE.md content as hypotheses, not ground truth.** Bloated
files routinely describe stacks that drifted — libraries removed, conventions
abandoned, patterns that never shipped. Before carrying any claim forward, verify
it against actual config files, lockfiles, and source code.

### Toolchain detection

Report the detected toolchain status explicitly:

- **Linter:** `<tool name>` / none detected
- **Formatter:** `<tool name>` / none detected
- **Pre-commit hooks:** `<tool name>` / none detected
- **Type checker:** `<tool name>` / none detected

When "none detected," the mechanism selection protocol Step 1 is N/A for that
category — skip directly to Step 2.

### Subagent output template

The subagent must return a structured report in this format:

```markdown
## Exploration Report

### Confidence: <N>% (based on <files read> / <estimated relevant files>)

### Toolchain
- Package manager: <name> (detected from <file>)
- Linter: <name> / none detected
- Formatter: <name> / none detected
- Type checker: <name> / none detected
- Pre-commit hooks: <name> / none detected
- CI: <system> (detected from <file>)

### Commands
- Build: `<exact command>`
- Test: `<exact command>`
- Lint: `<exact command>`
- Format: `<exact command>`
- Typecheck: `<exact command>`

### Monorepo structure (if applicable)
- Workspace config: <file>
- Members:
  - <member>: <stack summary>
  - ...

### Framework / runtime
<Only if it informs a non-obvious convention>

### Existing context layer
- CLAUDE.md: <exists / missing> (<N> lines)
- .claude/rules/: <list of files with paths: scope>
- .claude/skills/: <list of skills>
- Other agent configs: <AGENTS.md / .cursorrules / etc. — noted, not imported>

### Toolchain overlaps with CLAUDE.md (linter/formatter/hooks/config)
- <CLAUDE.md line N>: "<quoted>" → covered by <tool> <rule/setting>
- ...

### Candidate rules (from code exploration)
- <candidate>: <rationale>
- ...

### Gaps (confidence < 80%)
- <what remains unexplored>
```

## Phase 3 — Health card

Read [health-card.md](health-card.md) for the template.

Generate a health card for the root CLAUDE.md and a summary for each rules/skills
file. The health card is a diagnostic snapshot — it categorizes issues by type so
the user can see what kinds of improvements are available.

## Phase 4 — Proposals

For each issue found, generate a proposal:

```
[N] <action>: line <line> "<quoted content>"
    → <rationale>
    → Proposed change: <what to do>
    Target layer: CLAUDE.md | .claude/rules/<name>.md | .claude/skills/<name>/
    Decision: [keep / keep (protected) / drop / rewrite / migrate / graduate / add]
```

**Actions:**

- **graduate** — move to linter / formatter / hook / config (see Graduation
  section below).
- **drop** — remove (duplicates README, states the obvious, AI already knows).
- **rewrite** — fix phrasing (passive → imperative, negative → positive, add
  rationale to prohibition, split compound bullet).
- **migrate** — move to a different layer (CLAUDE.md → rule, rule → skill, or
  reverse: rule → CLAUDE.md when content is every-session).
- **keep (protected)** — failure-driven rule, default keep.
- **add** — new content discovered during exploration.

Each proposal must include:

- **What** changes (quoted original if editing/removing).
- **Why** the change helps (tied to a specific writing principle or mechanism
  selection protocol step).
- **Where** the content should live (decision tree layer).

When proposing a new skill scaffold, include:

- Suggested `name` (kebab-case).
- Draft `description` (semantic trigger — pushy enough to cover realistic
  phrasings).
- Outline of the SKILL.md body content.
- A note that after the user confirms, actual creation should be handed off
  to `skill-creator`, which provides the full workflow: eval loop, test
  cases, and description optimization. This skill decides the layer;
  skill-creator builds the content.

When proposing an update to an existing skill (stale content, inaccurate
trigger, layer migration), flag that the edit should be performed via
`skill-surgeon` to preserve unintended-rewrite protection. Include the
specific change as a concrete before/after diff so the user can hand it to
skill-surgeon directly.

## Phase 5 — User review

Present the health card first, then all proposals grouped by action type.
The user decides yes/no (or modifies) each proposal.

## Phase 6 — Diff preview

After collecting all decisions, generate a complete diff preview showing the
final state of every file that will be created, modified, or deleted:

- The full new content of CLAUDE.md.
- Any new or modified rule files (with correct `paths:` frontmatter).
- Any new skill scaffolds (with `name` and `description` frontmatter).
- Any files to delete (e.g., removing a redundant nested CLAUDE.md).
- **Graduation config changes:** For each `graduate` proposal, include the diff
  of the toolchain config file (e.g., `.eslintrc`, pre-commit config). These
  must be visible in the preview alongside the context-layer deletions.

The user reviews the final state and confirms.

## Phase 7 — Apply

Apply changes in two passes:

**Pass 1 — Graduation config changes.** For each `graduate` proposal, apply the
toolchain config change first (e.g., enable the lint rule, add the hook).
Verify the config change is valid (e.g., run `eslint --print-config` or
equivalent). If verification fails, abort the graduation for that rule, report
the failure to the user, and keep the context-layer rule in place. Only proceed
to Pass 2 after all graduation configs are verified.

**Pass 2 — Context-layer changes.** Write all CLAUDE.md / rules / skills
changes. This includes deleting context-layer rules that graduated in Pass 1.

Verify after both passes:

- CLAUDE.md line count is within budget (target 100–150, red line 200).
- All `paths:` frontmatter in rules uses the correct format.
- No `@import` remains in CLAUDE.md.
- No subdirectory CLAUDE.md was created.
- Every graduated rule has its toolchain enforcement in place before context
  deletion.

---

## Graduation

When the mechanism selection protocol Step 1 identifies a rule that should
graduate to a toolchain-level enforcement, follow this checklist:

### Graduation checklist

1. **Identify the target tool.** Which linter / formatter / hook / config file
   should enforce this rule?
2. **Locate the rule.** Does the tool already have an equivalent rule? What is
   its name/ID?
3. **Propose the config change.** Show a diff of the config file that enables
   the rule (e.g., adding a rule to `.eslintrc`, adding a pre-commit hook).
4. **User confirms the config change.** The user decides whether to apply.
5. **Only after the config change is confirmed:** remove the rule from
   CLAUDE.md / rules / skills.

Never delete a rule from the context layer before the toolchain enforcement is
in place. The user must confirm both sides of the graduation.

### Graduation proposal format

```
[N] graduate: line <line> "<quoted content>"
    → This is enforced by <tool> rule `<rule-name>`.
    → Config change needed: <file> — <diff or description>
    → After config is confirmed, remove from <CLAUDE.md / rule file>.
    Decision: [graduate / keep / keep (protected)]
```
