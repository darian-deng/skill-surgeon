# assess-candidate

Called by a Stage 4 implementer subagent after full unit tests pass, to assess a
knowledge candidate discovered during task implementation. **Not for interactive
user use** — use `handle-one-directive` for that.

Key difference from `handle-one-directive`:
- No conflict detection (deferred to Stage 6)
- No code enrichment (implementer is already in the code)
- Does NOT write to context files — records routing decision to task report only
- Linter graduation is the one exception: lint config CAN be written immediately

---

## Pre-Step: Comment Check

First question: "Can a code comment at this specific location fully explain this?"

| Result | Action |
|---|---|
| YES — file header, block, or inline comment suffices | Write the comment; record in `INLINE_COMMENTS_ADDED`; **stop** |
| NO — knowledge spans files, affects future code, or has rejected alternatives | Continue to Step 1 |

---

## Step 1: Linter Feasibility Check

Same methodology as `handle-one-directive` Step 1 (see `linter-capabilities.md`).

Ask: "If this directive CAN be enforced by [linter], write the exact rule configuration
and cite the official documentation URL. If not, say 'not enforceable'."

| Result | Action |
|---|---|
| Valid config + citation produced | **Graduate immediately**: modify lint config, run linter to verify; record as linter-graduated in `INLINE_COMMENTS_ADDED`; **stop** |
| Not enforceable | Continue to Step 2 |

Linter graduation is the only write to project files that happens in Stage 4.
All other writes are deferred to Stage 6.

---

## Step 2: Layer Routing (No Conflict Detection)

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

## Step 3: Record in Task Report

Append the routing decision to the current task's section in `task-reports.md`.
**Do NOT write to CLAUDE.md, rules, or ADR files.**

### For ADR_CANDIDATES routing:

```
### ADR_CANDIDATES
- [one-line decision title]: [trade-off rationale; what alternative was rejected and why]
  Source: [file:line where decision is visible in code]
```

### For CONTEXT_CANDIDATES routing:

```
### CONTEXT_CANDIDATES
- `rules/<domain>.md`: [description of constraint + source file:line]
- `CLAUDE.md`: [global behavioral rule description]
- `skill`: [procedure name + what it does]
```

### For NEW_TERMS_OR_PATTERNS routing (naming conventions, new terms):

```
### NEW_TERMS_OR_PATTERNS
- [term]: [what it means, suggested layer: rules/<domain>.md or CLAUDE.md]
```

### For deprecated routing:

Do not record. Silently discard — the candidate has no context-layer value.

---

## What NOT to do

- Do not scan `CLAUDE.md`, `rules/*.md`, or `docs/adr/` for conflicts — Stage 6 handles this
- Do not write to any context layer file (except lint config graduation)
- Do not ask the user — this runs silently inside a Stage 4 subagent
- Do not run code-explorer — you are already in the code; use what you know
