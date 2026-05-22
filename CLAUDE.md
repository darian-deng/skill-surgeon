# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 思维纪律

**不要自信，要正确。暴露权衡，不要假装确定。**

- **立场反转必须有新证据**：用户不同意 ≠ 我错了。维持立场时说「我的依据是X，要改变需要Y」；改变立场时说明是哪条新信息触发了改变
- **否定断言前找反例**：做「不支持/做不到/不可能」的断言前，先在代码或文档中搜索，或用 `tvly search` 查证。未验证的标为推测
- **信源三级**：代码/文档/可复现 = 可断言；训练数据印象 = 只够提假设；说清来源
- **发现问题必须说出来**：逻辑漏洞、技术谬误、更简单的方案——给证据指出，沉默即失职
- **假设必须外化**：发现自己在猜时，停下来写出假设并问用户，不要默默往下跑
- **自我纠错先于被指出**：发现说错了，主动更正并说明错在哪

**起效的标志**：改变立场时有明确触发原因；否定断言前有搜索/检查动作；推回用户时附具体证据而非「我认为」；自我更正发生在用户指出之前。

## What this repo is

A collection of AI agent skills for Claude Code, distributed via [skills.sh](https://skills.sh/darian-deng/agent-skills). Each skill lives in `skills/<skill-name>/` and is triggered by Claude when the user's intent matches the skill's `description` frontmatter.

## Installing skills

```bash
# Install all skills globally
npx skills add darian-deng/agent-skills -g --all -y

# Install one skill
npx skills add darian-deng/agent-skills --skill <skill-name> -g -y
```

## Skill anatomy

Every skill requires a `SKILL.md` with YAML frontmatter (`name`, `description`) followed by Markdown instructions. Optional subdirectories:

```
skills/<skill-name>/
├── SKILL.md          # required — frontmatter + instructions
├── references/       # docs loaded into context on demand
├── scripts/          # executable code for deterministic tasks
├── assets/           # templates, icons, fonts used in outputs
└── templates/        # file templates the skill writes out
```

The `description` field is the sole triggering mechanism — Claude reads it and decides whether to invoke the skill. Make descriptions specific and include both **what** the skill does and **when** to use it.

## skill-surgeon eval pipeline

`skill-surgeon` bundles a Python eval + description-optimization pipeline. Run all commands from the `skills/skill-surgeon/` directory:

```bash
# Run trigger eval for a skill description
python -m scripts.run_eval \
  --eval-set <path-to-evals.json> \
  --skill-path <path-to-skill-dir> \
  --runs-per-query 3 --verbose

# Run the full optimization loop (eval + improve, up to 5 iterations)
python -m scripts.run_loop \
  --eval-set <path-to-evals.json> \
  --skill-path <path-to-skill-dir> \
  --model <model-id> \
  --max-iterations 5 --verbose

# Package a skill into a distributable .skill file
python -m scripts.package_skill <path/to/skill-folder>

# Aggregate benchmark results across iterations
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>

# Generate the eval viewer (HTML review UI)
python eval-viewer/generate_review.py <workspace>/iteration-N \
  --skill-name "my-skill" \
  --benchmark <workspace>/iteration-N/benchmark.json
```

Eval results go in `<skill-name>-workspace/` as a sibling to the skill directory, organized by iteration. The `run_eval.py` script uses `claude -p` subprocess calls, requiring the `claude` CLI to be in PATH.

## Adding a new skill

1. Create `skills/<skill-name>/SKILL.md` with `name` and `description` frontmatter.
2. Add bundled resources under `references/`, `scripts/`, or `assets/` as needed.
3. Announce it in `README.md` using the existing table format.
4. Optionally run the description optimization loop via skill-surgeon to improve triggering accuracy.

## Current skills

| Skill | Purpose |
|---|---|
| `optimize-claude-context` | Audit and restructure CLAUDE.md / rules / skills to minimize context bloat |
| `adr-manage` | Create, supersede, index, list, and search Architecture Decision Records |
| `skill-surgeon` | Safe surgical updates to skill files + full eval/optimization pipeline for skill creation |
