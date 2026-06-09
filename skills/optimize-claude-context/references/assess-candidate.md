# assess-candidate

Called by a Stage 4 implementer subagent after full unit tests pass, to assess a
knowledge candidate discovered during task implementation. **Not for interactive
user use** — use `handle-one-directive` for that.

Key difference from `handle-one-directive`:
- No conflict detection (deferred to Stage 6)
- No code enrichment (implementer is already in the code)
- Does NOT write to context files — returns the routing decision to the calling stage, which decides how to record it
- Linter graduation is the one exception: lint config CAN be written immediately

---

## Pre-Step: Comment Check

First question: "Can a code comment at this specific location fully explain this?"

| Result | Action |
|---|---|
| YES — file header, block, or inline comment suffices | Write the comment; report the comment location to the caller; **stop** |
| NO — knowledge spans files, affects future code, or has rejected alternatives | Continue to Step 1 |

---

## Step 1: Linter Feasibility Check

Same methodology as `handle-one-directive` Step 1 (see `linter-capabilities.md`).

Ask: "If this directive CAN be enforced by [linter], write the exact rule configuration
and cite the official documentation URL. If not, say 'not enforceable'."

| Result | Action |
|---|---|
| Valid config + citation produced | **Graduate immediately**: modify lint config, run linter to verify; report the linter graduation (rule + location) to the caller; **stop** |
| Not enforceable | Continue to Step 2 |

Linter graduation is the only write to project files that happens in Stage 4.
All other writes are deferred to Stage 6.

---

## Step 2: Layer Routing (No Conflict Detection)

**Persistence Gate first (`decision-tree.md` Step 3.0)** — a candidate that fails the
litmus test → return `deprecated` (the caller discards it); do not route it. Distinct
from the Pre-Step: the Pre-Step keeps the knowledge *as a code comment*; the gate
discards it entirely. You are already in the code, so the gate's "inferable from stable
code" check is well-grounded here.

Apply the routing priority table from `decision-tree.md` Step 3.

**Deliberately skip conflict detection.** Stage 4 operates per-task with isolated
context. Cross-task and cross-source conflict detection happens in Stage 6 via
`handle-one-directive`.

| Priority | Condition | Target |
|---|---|---|
| 1 | Multi-step procedure, Skill will be developed | **Skill** |
| 1b | Multi-step procedure, Skill won't be developed soon | **Path rule** |
| 2 | scope=root, behavioral rule every session | **CLAUDE.md** |
| 3 | scope=package-path, lookup/reference | **Path rule** |
| 4 | Explanatory rationale with all three ADR gate conditions met | **ADR** |
| 5 | None of the above | **deprecated** |

ADR gate — all three must be YES:
1. Hard to reverse
2. Contrary to common practice
3. Has a rejected alternative with non-obvious rejection reason

---

## Step 3: Return the Decision to the Caller

**Do NOT impose a record format and do NOT write to CLAUDE.md, rules, or ADR files.**
Return the routing decision to the calling stage; the caller decides how and where to record it.

For each candidate, return:

- **Target layer**: `Skill` | `Path rule` | `CLAUDE.md` | `ADR` | `deprecated`
- **Rationale**: why this layer. For an ADR target, include the trade-off + which alternative was rejected and why.
- **Source**: `file:line` where the decision / term is visible in code.
- For a naming / term candidate: the term + what it means + suggested layer.

`deprecated` → return "no context-layer value"; the caller discards it.

---

## What NOT to do

- Do not scan `CLAUDE.md`, `rules/*.md`, or `docs/adr/` for conflicts — Stage 6 handles this
- Do not write to any context layer file (except lint config graduation)
- Do not ask the user — this runs silently inside a Stage 4 subagent
- Do not run code-explorer — you are already in the code; use what you know
