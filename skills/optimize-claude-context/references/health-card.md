# Health Card Template

The health card is a diagnostic snapshot for a CLAUDE.md or rules file. It
categorizes issues by type rather than assigning abstract scores. Each metric
directly maps to a writing principle or mechanism selection protocol step.

## Line counting convention

**Total lines** (including blank lines) is the standard throughout this skill.
Use `wc -l` or equivalent. This matches what humans see when they open the file.

## Template

```
CLAUDE.md Health Card — <file path>
──────────────────────────────────────────────

Volume
  Line count:              <N> (target 100–150, red line 200)   <✓|⚠|✗>
  MUST/IMPORTANT count:    <N> / 5 (max)                        <✓|⚠|✗>

Separation of concerns
  Overlaps with linter/formatter/
    hooks/config:                        <N> items   → graduation candidates
  Non-root CLAUDE.md files:              <N> items   → consolidate/delete
  Rules missing paths: frontmatter:      <N> items   → add paths: or justify
  Duplicates README/package.json:        <N> items   → drop candidates
  Prohibitions without rationale:        <N> items   → rewrite candidates
  File paths instead of domain concepts: <N> items   → rewrite candidates
  Should be in rules (not CLAUDE.md):    <N> items   → migrate candidates
  Should be in skills (not CLAUDE.md):   <N> items   → migrate candidates
  Wrong direction (every-session fact
    buried in rule/skill):               <N> items   → migrate-up candidates
  @import references:                    <N> items   → inline or migrate

Structure
  Heading structure:             <well-structured / needs improvement>
  Compound bullets (multi-rule): <N> items   → split candidates
  Commands outside code fences:  <N> items   → rewrite candidates

Content quality
  Failure-driven rules (protected):          <N> items   → default keep
  Stale references (paths/tools not found):  <N> items   → verify candidates
  Content Claude does correctly unprompted:   <N> items   → drop candidates

──────────────────────────────────────────────
```

## Indicator thresholds

| Metric | ✓ | ⚠ | ✗ |
|---|---|---|---|
| Line count | ≤ 150 | 151–200 | > 200 |
| MUST/IMPORTANT count | ≤ 3 | 4–5 | > 5 |
| Linter/formatter/hooks/config overlaps | 0 | 1–2 | ≥ 3 |
| Non-root CLAUDE.md files | 0 | — | ≥ 1 |
| Rules missing paths: frontmatter | 0 | 1 | ≥ 2 |
| Prohibitions without rationale | 0 | 1–2 | ≥ 3 |
| @import references | 0 | — | ≥ 1 |

For other metrics, any count > 0 triggers a proposal. The threshold table above
determines the severity indicator only.

## How to populate each metric

### Volume

- **Line count**: total lines (`wc -l`). Compare against 100–150 target and
  200 red line.
- **MUST/IMPORTANT**: count occurrences of "MUST" and "IMPORTANT"
  (case-insensitive). Exclude the `# CRITICAL — Read first` section — it is
  expected to use strong language. Do NOT count "CRITICAL" or "NEVER" as part
  of this metric; NEVER prohibitions are tracked under "Prohibitions without
  rationale" instead (and are fine if they have rationale).

### Separation of concerns

- **Overlaps with linter/formatter/hooks/config**: for each rule, check the
  project's linter config, formatter config, pre-commit hooks, `.editorconfig`,
  and other config files for an equivalent enforcement. If a matching mechanical
  enforcement exists or could trivially be enabled, count it.
- **Non-root CLAUDE.md files**: glob `**/CLAUDE.md` and `.claude/CLAUDE.md`.
  Any CLAUDE.md not at `./CLAUDE.md` is a consolidation candidate.
- **Rules missing paths: frontmatter**: check each `.claude/rules/*.md` for
  `paths:` frontmatter. Rules without it load unconditionally, consuming
  every-session budget — flag unless the content genuinely applies to all files.
- **Duplicates README/package.json**: check if the information is already
  present in README.md, package.json scripts, or other config files that Claude
  reads on demand.
- **Prohibitions without rationale**: any bullet that says "never/don't/avoid X"
  without explaining why or offering an alternative.
- **File paths instead of domain concepts**: references to specific file paths
  that could be expressed as domain concepts.
- **Should be in rules/skills**: content that does not apply to every session
  but lacks a `paths:` scope or would benefit from semantic triggering.
- **Wrong direction**: every-session facts (package manager, cross-cutting
  commands) that are buried in a path-scoped rule or skill instead of CLAUDE.md.
- **@import references**: count all `@path` references.

### Structure

- **Heading structure**: verify H1/H2/H3 hierarchy, no H4+, logical grouping.
- **Compound bullets**: bullets containing multiple unrelated instructions
  joined by commas or "and".
- **Commands outside code fences**: commands mentioned in prose rather than
  in fenced code blocks.

### Content quality

- **Failure-driven rules**: highly specific rules that appear to stem from real
  incidents. Flag for protection, not removal.
- **Stale references**: grep the codebase for paths, tool names, or module names
  mentioned in the file. Flag anything not found.
- **Content Claude does correctly unprompted**: generic advice like "write clean
  code", "handle errors", "add tests", "use meaningful names".

## Rules / Skills Summary Template

For each `.claude/rules/*.md` and `.claude/skills/*/SKILL.md`, produce a shorter
summary using this template.

```
Rule/Skill Summary — <file path>
──────────────────────────────────────────────
  Type:                    rule / skill
  Line count:              <N> (rule target ~150, skill < 500)
  paths: / trigger:        <glob pattern> / <description field> / missing
  Layer correct?           <yes / should migrate to ...>
  Overlaps with CLAUDE.md: <N> items → deduplicate
  Overlaps with toolchain: <N> items → graduation candidates
  Stale references:        <N> items → verify
  [rules only]
  Collateral damage:       <low / high — glob fires in many irrelevant
                            situations → consider converting to skill>
  [skills only]
  Trigger accuracy:        <accurate / stale — description no longer matches
                            current project patterns or tool names>
──────────────────────────────────────────────
```
