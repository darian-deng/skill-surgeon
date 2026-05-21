# Writing Principles

Apply every principle below when writing or reviewing content for CLAUDE.md,
path-scoped rules, or skills.

## 1. Imperative form

Write direct commands, not passive preferences.

- ✅ `Use named exports exclusively.`
- ❌ `It's preferred that default exports be avoided.`

## 2. Positive phrasing

State what to do, not what to avoid. Negated concepts still activate the concept
being negated — flipping to positive has been shown to cut rule violations
roughly in half.

- ✅ `Use the project's structured logger for all runtime logging.`
- ❌ `Never use console.log.`

**Exception (cross-reference with principle 3):** Specific negatives describing
a concrete failure mode Claude has hit are acceptable — e.g., "Do not add event
handlers when the framework manages reactivity." The test is specificity: vague
negatives are the problem, not negatives in general. When keeping a negative,
always include rationale and an alternative (principle 3).

## 3. Every prohibition carries rationale + alternative

This lets Claude generalize to related situations it was not explicitly told
about.

- ✅ `Never force-push to shared branches — it rewrites history other
  collaborators depend on. Use --force-with-lease on personal branches only.`
- ❌ `Never force-push.`

**Cross-reference with principle 2:** Do not forcibly rewrite specific
failure-mode prohibitions to positive form. A well-justified prohibition with
rationale is more informative than a vague positive restatement.

## 4. MUST / IMPORTANT ≤ 3–5 per file

Reserve emphasis for rules that truly cannot be broken. If everything is
IMPORTANT, nothing is.

## 5. Primacy–recency anchoring (unconditional)

LLMs weight the beginning and end of context most heavily. Mirror the 3 most
critical rules in a `# CRITICAL — Read first` section at the top AND a
`# CRITICAL — Read last` section at the bottom. Intentional duplication.

Template skeleton:

```markdown
# CRITICAL — Read first

- [Most critical rule #1, with rationale]
- [Most critical rule #2, with rationale]
- [Most critical rule #3, with rationale]

## Commands
...

## Conventions
...

# CRITICAL — Read last

- [Same 3 rules repeated]
```

## 6. Never write what a linter / formatter / hook / config file can enforce

If the project's toolchain can mechanically enforce a rule, it does not belong in
CLAUDE.md. "Always run the formatter" should graduate to a pre-commit hook.
"Use 2-space indentation" belongs in `.editorconfig` or Prettier config.

The skill does not maintain a hardcoded list of lint rules per language. Instead,
inspect the project's actual config files to determine coverage.

## 7. Never write what Claude already does correctly unprompted

"Write clean code", "Handle errors gracefully", "Follow DRY" are pure noise.
If Claude would do it anyway without the instruction, cut the instruction.

## 8. Domain concepts over file paths

Describe the concept, not the location. Concepts survive refactors; paths break
the moment someone moves a file.

- ✅ `Authentication uses session tokens with Redis-backed storage.`
- ❌ `Auth logic lives in src/auth/handlers.ts.`

## 9. No @import

Every `@path` reference in CLAUDE.md expands into context at session launch,
consuming tokens whether or not the content is relevant. This defeats lazy
loading.

If content does not need to be in every session, move it to a path-scoped rule
(triggered by file glob) or a skill (triggered semantically). If it does need to
be in every session, inline it directly in CLAUDE.md.

## 10. One instruction per bullet

Combining multiple rules in one bullet reduces compliance with all of them.

- ✅ Three separate bullets, each with one rule.
- ❌ `Use named exports, prefer const over let, and avoid default parameters.`

## 11. Commands in code fences with exact flags

A command in a fenced code block gets followed. A command buried in prose gets
lost.

- ✅ `` `pnpm test --filter=api` `` — run API tests
- ❌ "Run the tests for the API package."

## 12. Markdown structure as parsing landmarks

Use H1 for critical anchors and top-level splits, H2 for sections, H3 only when
truly needed. Never H4+. Flat beats deep.

Bold text within sections serves as a secondary landmark. Do not strip markdown
formatting — headers and bold help smaller models parse structure reliably.

## 13. Line budget

- **Target:** 100–150 total lines per CLAUDE.md (total lines = `wc -l`).
- **Red line:** 200 total lines. Exceeding this triggers a mandatory audit.
- Principle: "fewer high-quality lines > more mediocre lines."
- Small projects (< 10 source files) may have well under 100 lines — do not
  pad to reach the target.

Path-scoped rules: target ~150 total lines, red line ~200 (per file). Skills:
SKILL.md body < 500 lines ideal; overflow goes to `references/`. The same "every
line must earn its place" standard applies to all file types.

## 14. Litmus test

For every line, ask: _"If I remove this line, will Claude make a mistake it
wouldn't otherwise make?"_

If no → cut it.
If unsure → cut it and observe. The user can ask for it back.

## 15. Failure-driven rules default to keep

During audits, a rule that looks odd but is highly specific (e.g., "Do not add
event handlers when the framework manages reactivity") likely exists because of
a real incident. Default to keeping it unless the user explicitly confirms it is
obsolete.
