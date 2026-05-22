# Decision Tree — Where Content Belongs

## The three layers

### Layer 1: Root CLAUDE.md

**File:** `./CLAUDE.md` (single file at project root)

**Loaded:** Every session, unconditionally.

**Put here when:**

- Claude needs this fact in every session regardless of which files are being
  touched.
- Examples: package manager name, cross-cutting build/test commands, monorepo
  layout overview, top-3 critical rules (anchored at top and bottom).

**Do NOT put here when:**

- The rule only applies to certain file types or directories → use a rule.
- The content is a multi-step procedure or domain knowledge for a specific
  task → use a skill.

### Layer 2: Path-scoped project rules

**Directory:** `./.claude/rules/`

**Loaded:** When Claude reads a file matching the `paths:` glob in the rule's
YAML frontmatter. Rules without `paths:` load unconditionally (same as CLAUDE.md
content — use sparingly).

**Frontmatter format:**

```yaml
---
paths:
  - "src/api/**/*.ts"
  - "src/services/**/*.ts"
---
```

**Known issues (as of 2026-05):**

- YAML list format for `paths:` may only match the first pattern in some
  versions. If this occurs, use CSV format:
  `paths: "src/api/**/*.ts,src/services/**/*.ts"`
- `globs:` field may load unconditionally regardless of pattern. Use `paths:`
  instead.
- All rules in `.claude/rules/` live at the project root. Subdirectory
  `.claude/rules/` directories are not documented as supported.

**Put here when:**

- The rule applies only when Claude is working with files matching a specific
  glob pattern, **and** the glob fires with low collateral damage — i.e., it
  triggers only in situations where the rule is genuinely relevant.
- Examples: "React component conventions" scoped to `**/*.tsx`, "API design
  rules" scoped to `src/api/**/*`, "Terraform conventions" scoped to
  `infra/**/*`.
- Counter-example (use a skill instead): "When adding user-facing text, add i18n
  keys to `locales/`" scoped to `**/*.tsx` — the glob fires on every tsx change,
  even unrelated refactors. The intent is semantic ("adding user-facing text"),
  not file-path-triggered.

**Monorepo pattern:**

```
.claude/rules/
  ├── web.md         ← paths: "apps/web/**/*"
  ├── api.md         ← paths: "services/api/**/*"
  └── infra.md       ← paths: "infra/**/*"
```

### Layer 3: Skills

**Directory:** `./.claude/skills/<name>/SKILL.md`

**Loaded:** When Claude determines from its prompt analysis that the skill is
relevant (semantic trigger via the `description` field), or when the user
explicitly invokes `/<skill-name>`.

**Put here when:**

- The content cannot be triggered by a file-path glob.
- Examples: "API design workflow", "migration script procedure", "code review
  checklist", "onboarding documentation guide".

**Skill scaffold conventions:**

- **Directory name:** kebab-case, matching the `name` field in frontmatter.
- **`description` field:** the primary trigger — write it "pushy" enough to
  cover realistic user phrasings. Include both what the skill does AND specific
  trigger phrases.
- **Body:** imperative instructions, < 500 lines ideal.
- **`references/` subdirectory:** for content that exceeds ~200 lines in the
  SKILL.md body. The SKILL.md references these files with prose paths; the agent
  reads them on demand.

## Reverse-layer detection (for audits)

During audits, also check for content in the **wrong layer going the other
direction**:

| Symptom | Fix |
|---|---|
| Every-session fact buried in a path-scoped rule | Migrate to CLAUDE.md |
| Procedural workflow in a rule (no path trigger makes sense) | Migrate to a skill |
| Path-scoped content in CLAUDE.md wasting every-session budget | Migrate to a rule with `paths:` |
| Path rule whose glob fires in many irrelevant situations (high collateral damage) | Migrate to a skill |

## Out of scope: user-level rules

User-level rules in `~/.claude/rules/` apply to every project on the machine.
This skill manages project-level artifacts only. Note: user-level rules'
`paths:` frontmatter is currently ignored (see Claude Code issue #21858), so
they always load unconditionally. If encountered during audit, note their
existence but do not modify them.
