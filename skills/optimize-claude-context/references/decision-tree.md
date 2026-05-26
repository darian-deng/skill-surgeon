# Decision Tree

Routes a directive to its correct layer. Run all steps in order. Stop at the first
terminal outcome (Graduate, CONFLICT, or a layer assignment).

Foundational concepts: `directive.md`
Linter methodology: `linter-capabilities.md`
Writing specs: `writing-formats.md`

---

## Step 0 — Parse

**0a. Atomicity check**

Apply the atomic definition from `directive.md`: one independently routable
knowledge point. If the input contains two or more directives:
- Split into individual directives.
- Process each one independently from Step 1.
- Do not continue with a merged directive.

```
input: "Project uses pnpm and main process uses four-layer architecture"
→ split into two directives; run decision tree twice
```

**0b. Scope determination**

Assign scope before continuing.

- Does this knowledge apply equally everywhere in the project? → `root`
- Does it only apply within one subdirectory or package? → `<package-path>`
- Ambiguous? → `root` (safe default; can be narrowed later)

Scope is required for Step 1 (which toolchain to check) and Step 3 (which layer
within scope to target).

---

## Step 1 — Linter Feasibility Check

Identify the language and primary linter for the directive's scope. Read the
toolchain config files for that scope:

- JavaScript/TypeScript: `package.json`, `eslint.config.*`, `.eslintrc.*`, `tsconfig.json`
- Python: `pyproject.toml`, `ruff.toml`, `.flake8`
- Rust: `Cargo.toml`, `clippy.toml`
- Any scope: `.pre-commit-config.yaml`, `.editorconfig`

**Ask exactly this question:**

> "If this directive CAN be enforced by [linter], write the **exact rule
> configuration** and cite the **official documentation URL** (or parent page URL
> for rules without individual docs). If you cannot produce a configuration backed
> by documentation, say 'not enforceable'."

| Result | Action |
|---|---|
| Valid config + citation produced | **Graduate** — modify (or create) linter config file; directive does NOT enter context layer |
| "Not enforceable" or no valid config | Continue to Step 2 |

See `linter-capabilities.md` for enforcement mechanisms commonly missed:
`no-restricted-syntax` (AST-based bans), `no-restricted-imports`,
`no-restricted-globals`, TypeScript compiler options, pre-commit hooks.

After graduating: verify by running the linter against code that should trigger the rule.
If the linter does not produce an error, the config is incorrect — do not graduate.
Revert the config change and continue to Step 2.

---

## Step 2 — Check for Existing Directives

Scan project-level context files only (never `~/.claude/`):
- `./CLAUDE.md`
- `./.claude/rules/*.md`
- `./.claude/skills/*/SKILL.md`

Search for semantic overlap — same domain, same intent — not just keyword matches.

| Result | Action |
|---|---|
| **New** — no semantic overlap | Continue to Step 3 |
| **Merge** — overlapping directive exists with less or different info | Combine; continue to Step 3 with merged directive |
| **Conflict** — sources contradict each other | Flag CONFLICT; surface to user; **halt** — do not write anything. Wait for user to resolve the conflict, then rerun from Step 2 with the resolved directive. |

**Conflict surface format:**

```
CONFLICT detected
  Existing: [file:line] "Never use try-catch"
  New:      "Use try-catch for async boundary errors only"
  Resolution required before proceeding.
```

**Stub entries:** `status: stub` skill entries count as pending, not as conflicting
existing directives.

---

## Step 3 — Layer Routing

Apply the layer selection rules in strict order. Stop at the first match.

| Priority | Condition | Target layer |
|---|---|---|
| 1 | Multi-step procedure **AND** Skill will be fully developed (not a stub) | **Skill** |
| 1b | Multi-step procedure, but Skill body won't be developed soon | **Path rule** (keep procedure in lookup/reference form) |
| 2 | scope=`root`, behavioral rule (must-always-do, applies every session) | **CLAUDE.md** |
| 3 | scope=`<package-path>`, lookup / reference content | **Path rule** with `paths:` glob |
| 3b | scope=`<package-path>`, content scenario-specific within glob scope AND no file pattern captures the trigger | **Skill** |
| 4 | Explanatory rationale ("why we chose X over Y") | **ADR** |
| 5 | None of the above | deprecated |

**Priority 1 condition explained:** a Skill stub without a body is a negative asset —
it loads semantically but provides no guidance. If the procedure is already documented
in an existing path-rule with a sufficiently precise glob, keeping it in path-rule is
better than an empty stub. Only route to Skill when the user commits to developing it.

**Path rule is the default for single rules.** Skill (Priority 3b) is the fallback
only when the trigger is purely semantic — no file pattern captures "the user is doing X".

**Scope vs content judgment (Priority 3 vs 3b):**
> "Is this content relevant for ALL work within the glob's scope, or only for a SPECIFIC TYPE of work within that scope?"
- ALL work → path-rule (LOW collateral damage; `when:` provides behavioral guidance)
- SPECIFIC TYPE → no file pattern captures the scenario → Skill

Note: for **existing path-rule files** in rebuild, this judgment is made holistically
per file in Phase 3B (not per directive). AI must not attempt to "narrow the glob"
for existing files — that requires understanding the original author's intent, which
cannot be reliably inferred from content alone.

**Procedure vs. rule distinction:**
- Procedure: has ordered steps, involves multiple files or commands, order matters.
  Example: "How to add a new log category" (create file → register → update index).
- Rule: a single behavioral constraint with no ordered steps.
  Example: "Always use structuredLogger for runtime logging."

**When routing to path-rule, immediately determine:**
1. **`domain-file`**: which `.claude/rules/<domain>.md` file receives this directive.
   Group by functional area (auth, api, logging, testing, etc.) — use the Phase 3
   domain grouping name as the domain file stem.
2. **`when:` statement**: one sentence describing the work scenario where this content
   is relevant. See `writing-formats.md §Path Rule Format` for guidance.
   Write automatically — no user confirmation needed. The `when:` is a behavioral
   hint for Claude (when to act on the content), not a pre-load filter.

**ADR vs. CLAUDE.md distinction:**
- ADR: explains *why* — the trade-off analysis, what alternatives were rejected.
- CLAUDE.md: states *what* — the rule itself, what Claude must do.
- If both a rule and its rationale exist: rule → CLAUDE.md, rationale → ADR.
  These are two separate directives.

**Deprecated criteria:** content that is obsolete, superseded by code structure,
or so generic that it adds no value (principle 6 from `writing-formats.md`).

---

## Confidence Calibration

Assign a confidence level to each routing decision. Used in rebuild Phase 4 table.

| Level | Criteria |
|---|---|
| **H** | Routing is unambiguous — only one valid layer given the decision tree |
| **M** | Two plausible layers exist and one was chosen; OR linter check result was uncertain |
| **L** | Directive is ambiguous in scope or type; a different reasonable agent might choose differently |

**Examples:**

```
"Use pnpm as the package manager"
  → not enforceable (package manager choice is not a lint rule)
  → no conflict
  → scope=root, behavioral, single constraint → CLAUDE.md
  → confidence: H

"How to release a new version"
  → not enforceable
  → no conflict
  → multi-step procedure → Skill
  → confidence: H

"API response shape conventions"
  → not enforceable via linter
  → no conflict
  → scope=apps/api/, lookup/reference → Path rule (paths: "apps/api/**/*.ts")
  OR scope=root → CLAUDE.md?
  → confidence: M (chose path rule because it's lookup/reference content for a
    specific package, not a universal behavioral rule)

"Authentication — use JWT or sessions?"
  → scope is unclear (root or apps/auth/?)
  → confidence: L
```

---

## Scope + Monorepo Routing

| Directive type | Scope | Target |
|---|---|---|
| Cross-cutting behavioral rule | `root` | `./CLAUDE.md` |
| Cross-cutting procedure | `root` | `./.claude/skills/<name>/SKILL.md` |
| Package-specific conventions | `apps/x/` | `./.claude/rules/<domain>.md` with `paths: "apps/x/**"` |
| Package-specific procedure | `apps/x/` | Still Skill (priority 1 always wins) |
| Decision rationale | any | `./docs/adr/NNNN-<slug>.md` (project root) or `<package>/docs/adr/NNNN-<slug>.md` (package-specific in monorepo) |

Monorepo layout example:
```
.claude/rules/
  ├── desktop.md    ← paths: "apps/plaud-desktop/**"
  ├── api.md        ← paths: "services/api/**"
  └── shared.md     ← paths: "packages/shared/**"
```

Global `~/.claude/CLAUDE.md` is never modified by this skill. If a project-specific
directive is found there during rebuild or audit, flag it for manual removal.

---

## Reverse-Layer Detection (for audit)

Check for content in the wrong layer. Each symptom maps to a fix.

| Symptom | Diagnosis | Fix |
|---|---|---|
| Every-session fact buried in a path-scoped rule | Rule glob fires too narrowly for universal content | Migrate to CLAUDE.md |
| Procedural workflow in a rule (no file-path trigger makes sense) | Should be Skill | Migrate to Skill |
| Path-scoped content in CLAUDE.md wasting every-session budget | Should be Rule | Migrate to Rule with `paths:` |
| Path rule whose glob fires in many irrelevant situations | High collateral damage | Migrate to Skill (semantic trigger) |
| Decision rationale in CLAUDE.md | Rationale ≠ behavioral rule | Migrate to ADR |
| Behavioral rule buried in ADR body | Rule ≠ rationale | Extract rule to CLAUDE.md; keep only rationale in ADR |
| Skill description > 15 words | Exceeds hard limit | Trim or run skill-creator eval loop |
| Rule without `paths:` frontmatter | Loads unconditionally | Add `paths:` or migrate to CLAUDE.md |
| Rule with `paths:` but no `when:` | Claude cannot self-filter relevance | Add `when:` field describing the work scenario |
| Rule where glob fires much broader than `when:` scenario | High collateral damage | Evaluate migration to Skill |
