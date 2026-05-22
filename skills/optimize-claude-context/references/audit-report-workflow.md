# Audit-Report Workflow

Pure diagnostic: explore the project, generate a health card and proposal list.
**Does not write any files.** Output is the report only.

## What it does

Run optimize-workflow.md Phases 1–4, then stop.

1. **Phase 1** — Bootstrap check (see optimize-workflow.md § Phase 1)
2. **Phase 2** — Deep exploration (see optimize-workflow.md § Phase 2)
3. **Phase 3** — Health card (see optimize-workflow.md § Phase 3)
4. **Phase 4** — Proposals (see optimize-workflow.md § Phase 4)
5. **Stop.** Present the health card and all proposals to the user.
   Do NOT proceed to Phase 5 (diff preview) or Phase 6 (apply).

Read [optimize-workflow.md](optimize-workflow.md) for the full procedure
for each phase.

## Output format

```
## Audit Report — <project name / CLAUDE.md path>

### Health Card
[health card output from Phase 3]

### Proposals
[proposals from Phase 4, grouped by action type: graduate / drop / rewrite /
migrate / add / keep (protected)]

### Suggested next steps
- Run optimize to apply all or selected proposals
- Run update-rule to make a specific targeted change
- Accept the report as-is — no action needed
```

The report is informational only. No files are modified.
