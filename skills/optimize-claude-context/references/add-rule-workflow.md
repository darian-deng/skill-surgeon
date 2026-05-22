# Add-Rule Workflow

Full procedure for adding, moving, or removing a specific rule.

## Step 1 — Receive candidate

The user provides a rule they want to add, move, or remove. Clarify intent if
ambiguous.

## Step 2 — Lightweight codebase verification

Do NOT blindly trust the user's input. Verify against the codebase before
proceeding. This does not require a subagent — use Read, Grep, and Glob
directly.

### 2a — Duplicate and conflict check

Read all existing context files:

- `./CLAUDE.md`
- `./.claude/rules/*.md`
- `./.claude/skills/*/SKILL.md`

Check for:

- **Exact duplicates**: the same rule already exists (possibly worded
  differently).
- **Semantic duplicates**: a rule covering the same concern exists (e.g., user
  wants to add "avoid class components" but "prefer functional components"
  already exists).
- **Conflicts**: a rule that contradicts the candidate (e.g., user wants "use
  default exports" but a rule says "use named exports").

### 2b — Verify referenced entities

If the candidate mentions specific paths, tools, modules, libraries, or
configuration:

- Glob / Grep the codebase to confirm they exist.
- If something does not exist, flag it as a **phantom dependency**.

### 2c — Lint / formatter coverage check

Read the project's linter, formatter, pre-commit hooks, and config files
(`.editorconfig`, `tsconfig.json`, etc.). Determine whether the candidate rule
is already enforced by any of these tools, or could be enforced by enabling a
rule or adding a hook.

If no toolchain config exists for the relevant category, that check is N/A —
note it and skip.

### 2d — Sample verification (when needed)

If the candidate makes a specific claim about code patterns (e.g., "all API
handlers return standardized error format"), sample-grep a few files to verify
the claim holds.

## Step 3 — Push back (when issues found)

If any verification step found a problem, push back boldly. Do not soften the
message. Explain exactly what was found and propose alternatives.

**Response patterns by issue type:**

| Issue | Response |
|---|---|
| Exact duplicate | "This rule already exists at [location]: `<quoted>`. No change needed unless you want to rephrase it." |
| Semantic duplicate | "A rule covering the same concern exists at [location]: `<quoted>`. Adding another would create redundancy. Consider updating the existing rule instead." |
| Conflict | "This conflicts with [location]: `<quoted>`. Which should take precedence? Or should we rephrase to reconcile?" |
| Phantom dependency | "`<path/tool>` referenced in the rule does not exist in this repository. Is this a future addition? If so, I'll add it with a note." |
| Lint coverage | "The project's `<linter>` already enforces this via `<rule-name>`. Propose graduating to lint config instead of adding to CLAUDE.md." |
| Claim not verified | "I sampled `<files>` and found `<evidence>` that contradicts the claim. Please verify." |

**Escape hatch:** If the user explicitly confirms a phantom dependency is
intentional (future state) or insists on adding a lint-covered rule, proceed
with a note explaining the user's rationale.

## Step 4 — Mechanism selection protocol

Run the two-step filter:

1. Can the project's linter / formatter / hooks / config enforce this?
   → Yes → propose graduation (see audit-refactor-workflow.md § Graduation).
2. If I remove this line, will Claude make a mistake it wouldn't otherwise make?
   → No → do not add. Explain why.
   → Yes → route through the decision tree (see decision-tree.md):
     - Every session? → CLAUDE.md
     - File-path triggerable? → `.claude/rules/<name>.md` with `paths:`
     - Semantically triggerable? → `.claude/skills/<name>/SKILL.md`

## Step 5 — Propose

Present the proposal:

- **Content**: the exact text to write (following all writing principles —
  imperative, positive phrasing preferred, rationale for any prohibition, single
  instruction per bullet, command in code fence). Specific failure-mode
  prohibitions with rationale are allowed and should not be forcibly rewritten
  to positive form.
- **Location**: which file, which section.
- **Rationale**: why this location, why this phrasing.
- **Anchor update**: if the target is CLAUDE.md and the rule is among the top-3
  most critical for the project, add it to both `# CRITICAL — Read first` and
  `# CRITICAL — Read last` sections.

**When the decision tree routes to a skill:**

- If creating a **new** skill: present the scaffold (name, description, body
  outline) and note that after the user confirms, execution should be handed off
  to `skill-creator` for the full creation workflow (eval loop, test cases,
  description optimization). Do not attempt to fully build the skill here.
- If updating an **existing** skill: present the specific change as a
  before/after diff and note that the edit should be performed via
  `skill-surgeon` to prevent unintended rewrites.

## Step 6 — Remove / demote workflow

When the user wants to **remove** a rule:

1. **Protected-rule check.** Is this a failure-driven rule (highly specific,
   likely incident-driven)? If so, flag it and ask the user to confirm removal.
   Default recommendation: keep.
2. **Anchor sync.** If the rule appears in a `# CRITICAL` anchoring section,
   it must be removed from both the top and bottom anchor sections.
3. **Dependent content.** Check whether other rules reference or depend on this
   one. Flag if so.
4. **Migrate vs delete.** If the rule is being "demoted" (e.g., from CLAUDE.md
   to a path-scoped rule), treat as a migration, not a deletion.

When the user wants to **move** a rule:

1. Identify the source and target layers.
2. Remove from source (following the remove checklist above).
3. Add to target (following Steps 4–5 above).

## Step 7 — Apply

Always generate a diff preview before writing, even for single-rule changes.
Show the before and after state of every file that will change.

**Graduation in add-rule:** If the mechanism selection protocol routes to
graduation (Step 4), follow the same 2-pass apply as audit-refactor:
1. Show the toolchain config diff alongside the context-layer diff.
2. Apply toolchain config change first (Pass 1).
3. Verify config is in place.
4. Only then apply context-layer change (Pass 2).

See `audit-refactor-workflow.md` § Graduation for the full checklist and
proposal format. If config verification fails, abort and report — do not
delete the context-layer rule.

User confirms → write the change.

After writing, verify:

- CLAUDE.md line count still within budget.
- No `@import` introduced.
- If the rule was added to or removed from `# CRITICAL` sections, both anchor
  sections are in sync.
- If a new rule file was created, `paths:` frontmatter is correct.
- If a new skill was scaffolded, `name` and `description` frontmatter are
  present.
