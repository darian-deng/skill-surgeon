# Directive

## Definition

A **directive** is a single piece of project-specific knowledge that Claude should
know when working on this project.

**Atomic rule:** one independently routable knowledge point. Removing it does not
affect the meaning of any other directive.

```
✅ "This project uses pnpm as the package manager"
✅ "The Electron main process uses four-layer architecture: index → app → services → utils"
✅ "Never use try-catch; use radash/tryit instead"
❌ "Project uses pnpm and main process uses four-layer architecture"
   → two directives merged; split before routing
```

**Atomicity test:** can you route this piece of knowledge to a layer independently
of all other knowledge? If two concepts share a layer but for different reasons,
they are separate directives.

---

## Scope

Every directive must have a scope. Scope determines which layer to target and which
toolchain to evaluate during the linter feasibility check.

| Scope value | Meaning |
|---|---|
| `root` | Applies to the entire project / all packages |
| `<package-path>` | Applies to a specific package, e.g. `apps/plaud-desktop/` |

**How to determine scope:**

1. Does this knowledge apply equally everywhere in the project? → `root`
2. Does it only apply within one subdirectory / package? → `<package-path>`
3. Ambiguous? → `root` (safe default; can be narrowed later)

**Monorepo rules:**
- `root` directives target the repo-level CLAUDE.md or repo-level linter configs.
- `<package-path>` directives target that package's `.claude/rules/` path rule or
  that package's linter config.
- In a monorepo, project-wide ADRs go in `<project-root>/docs/adr/`; package-specific
  ADRs go in `<package>/docs/adr/`. See the ADR path convention below.

---

## The Four Layers

Each directive lives in exactly one layer.

| Layer | Role | When loaded |
|---|---|---|
| **CLAUDE.md** | Behavioral rules — what Claude must always know and do | Every session, unconditionally |
| **Path rule** | Lookup / reference — conventions for specific file types | When touching files matching the glob |
| **Skill** | Workflow guide — multi-step procedures to execute | When user intent matches |
| **ADR** | Decision rationale — why a non-obvious decision was made | On demand by humans; never auto-loaded |

---

## Layer Selection Rules

Apply in this strict order:

| Condition | Target layer |
|---|---|
| Multi-step procedure (sequential, order matters, cross-file) | **Skill** — wins over all other conditions |
| scope=`root`, behavioral rule (must-always-do) | **CLAUDE.md** |
| scope=`<package-path>`, lookup / reference content | **Path rule** with `paths:` glob |
| Explanatory rationale ("why we chose X") | **ADR** |
| None of the above | deprecated |

**Key tiebreaker:** multi-step procedure → Skill, regardless of scope. A
cross-cutting procedure that applies to all packages is still a Skill, not CLAUDE.md.

**ADR path convention (authoritative):** ADRs live at `docs/adr/` (singular).
In a monorepo, project-wide ADRs go in `<project-root>/docs/adr/`; package-specific
ADRs go in `<package>/docs/adr/`. Dynamic discovery handles both `adr` and `adrs`
spellings for existing projects. When creating a new ADR directory, always use
`docs/adr/` — never `docs/adrs/`.

**Routing confidence:** every routing decision carries a confidence level (H/M/L)
that captures how many plausible layers exist for that directive. Confidence is a
decision-tree output concept, not part of the directive definition itself — see
`decision-tree.md` for calibration criteria.

**Procedure vs. rule distinction:**
- Procedure: sequential steps, order matters, involves multiple files or commands.
  Example: "How to add a new log category" (create file → register → update index).
- Rule: a single behavioral constraint with no ordered steps.
  Example: "Always use structuredLogger for runtime logging."

---

## Mutual Exclusivity

The same directive cannot appear in two layers. If a domain has related but
distinct content, route each piece separately:

```
domain: logging
  "Always use structuredLogger for all runtime logging"
    → behavioral rule, scope=root → CLAUDE.md

  "How to add a new log category"
    → multi-step procedure → Skill

  "Why we chose structuredLogger over console.log"
    → decision rationale → ADR
```

These are three separate directives covering different aspects of the same domain.
They are correctly distributed across three layers because each serves a different
loading purpose.
