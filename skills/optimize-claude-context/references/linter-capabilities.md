# Linter Capabilities

The linter feasibility check (Step 1 of the decision tree) prevents directives from
entering the context layer when the toolchain can enforce them mechanically. This file
defines the correct methodology and reference examples.

---

## The Correct Question

> "If this directive CAN be enforced by [linter], write the **exact rule
> configuration** and cite the **official documentation URL** (or parent page URL
> for rules without individual docs). If you cannot produce a configuration backed
> by documentation, say 'not enforceable'."

This phrasing forces production of a verifiable artifact:

- **Valid config + citation → graduate.** Modify (or create) the linter config
  file directly. The directive does NOT enter the context layer.
- **No valid config → not enforceable.** Continue to Step 2.

**Why exact config + citation:** prevents confident-sounding but incorrect
responses. AI commonly says "ESLint can enforce this" without being able to produce
the actual rule. The citation requirement forces verification.

**Parent page citations:** for rules that share a docs page (e.g., all TypeScript
compiler options at `https://www.typescriptlang.org/tsconfig`), the parent page URL
is acceptable.

---

## ESLint / TypeScript Mechanisms

### `no-restricted-syntax` — AST-based bans

Enforces arbitrary syntactic bans via AST selectors. Commonly missed by AI.

```json
// .eslintrc or eslint.config.js
{
  "rules": {
    "no-restricted-syntax": [
      "error",
      {
        "selector": "TryStatement",
        "message": "Use radash/tryit instead of try-catch."
      },
      {
        "selector": "ForInStatement",
        "message": "Use Object.keys() with for-of instead of for-in."
      },
      {
        "selector": "LabeledStatement",
        "message": "Labeled statements are not allowed."
      }
    ]
  }
}
```

Docs: https://eslint.org/docs/latest/rules/no-restricted-syntax

AST selector reference: https://eslint.org/docs/latest/extend/selectors

**Common patterns:**
- `TryStatement` → ban try-catch
- `ForInStatement` → ban for-in loops
- `AwaitExpression[argument.callee.name="fetch"]` → ban raw fetch
- `CallExpression[callee.name="setTimeout"]` → ban setTimeout

### `no-restricted-imports` — enforce import alternatives

```json
{
  "rules": {
    "no-restricted-imports": [
      "error",
      {
        "name": "lodash",
        "message": "Use radash instead of lodash."
      },
      {
        "patterns": ["../../../*"],
        "message": "Use absolute imports via path aliases."
      }
    ]
  }
}
```

Docs: https://eslint.org/docs/latest/rules/no-restricted-imports

### `no-restricted-globals` — ban global access

```json
{
  "rules": {
    "no-restricted-globals": [
      "error",
      { "name": "event", "message": "Use the local event parameter instead." },
      { "name": "console", "message": "Use the project logger instead." }
    ]
  }
}
```

Docs: https://eslint.org/docs/latest/rules/no-restricted-globals

### `no-restricted-properties` — ban property access

```json
{
  "rules": {
    "no-restricted-properties": [
      "error",
      {
        "object": "Math",
        "property": "pow",
        "message": "Use the exponentiation operator (**) instead."
      }
    ]
  }
}
```

Docs: https://eslint.org/docs/latest/rules/no-restricted-properties

### `@typescript-eslint` rules

```json
{
  "rules": {
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/no-floating-promises": "error",
    "@typescript-eslint/no-unsafe-assignment": "error"
  }
}
```

Docs: https://typescript-eslint.io/rules/

---

## TypeScript Compiler Options

TypeScript `tsconfig.json` enforces many behavioral constraints at compile time.

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

Docs (parent page): https://www.typescriptlang.org/tsconfig

---

## Ruff (Python)

```toml
# pyproject.toml or ruff.toml
[tool.ruff.lint]
select = ["E", "F", "B", "I"]

# Ban specific patterns via per-file-ignores or custom rules
[tool.ruff.lint.flake8-bugbear]
extend-immutable-calls = ["fastapi.Depends"]
```

Specific rule bans:
```toml
[tool.ruff.lint]
extend-select = ["TRY"]     # tryceratops: ban broad try-except
ignore = []
```

Docs: https://docs.astral.sh/ruff/rules/

---

## Pre-commit Hooks

Pre-commit hooks can enforce file naming, pattern bans, and custom checks.

```yaml
# .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      - id: no-console-log
        name: Ban console.log
        language: pygrep
        entry: 'console\.log\('
        types: [javascript, typescript]
        exclude: '^tests/'
      - id: filename-kebab-case
        name: Enforce kebab-case filenames
        language: script
        entry: scripts/check-filenames.sh
```

Docs: https://pre-commit.com/hooks.html

---

## Verification

After writing the linter config, verify enforcement actually works:

1. Write a code snippet that violates the rule.
2. Run the linter against that snippet.
3. Confirm an error is produced at the expected line.

```bash
# ESLint verification example
echo "try { foo() } catch(e) {}" > /tmp/test-lint.ts
npx eslint /tmp/test-lint.ts --rule '{"no-restricted-syntax": ["error", {"selector": "TryStatement"}]}'
# Expected: error on line 1
```

If no error is produced, the config is incorrect — do not graduate the directive.
