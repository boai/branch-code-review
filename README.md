# branch-code-review

A Claude Code skill that generates a structured Markdown code review for the diff between a feature branch and a base branch (main/master) — saved to `CodeReviews/<repo>.md` with severity-tagged findings (严重 / 高 / 中) and `file:line` citations grounded in the actual diff.

## What it does

- Auto-detects current branch as `target` and `origin/HEAD` (or `main`/`master`) as `base`
- Collects diff, shortstat, and commit log between the two branches
- Filters out noisy files (lockfiles, binaries, build artifacts, etc.)
- Truncates oversized diffs (>30,000 chars) and records the truncation
- Produces a fixed-structure Markdown report:
  - YAML frontmatter (title, tags, ISO-8601 timestamp with timezone offset)
  - 概览 / 严重问题 / 高优先级 / 中优先级 / 正面评价 / 结论
- Writes to `CodeReviews/<repo>.md`, asking before overwriting

## When to invoke

Trigger phrases:
- "review my branch", "审一下这个分支", "code review", "评审分支"
- Before opening or merging a PR
- After a stack of commits, when you want a structured assessment

## Install

This is a Claude Code [skill](https://docs.claude.com/en/docs/claude-code/skills). Drop the directory into your skills folder:

```bash
mkdir -p ~/.claude/skills/branch-code-review
cp SKILL.md ~/.claude/skills/branch-code-review/SKILL.md
```

Restart Claude Code (or reload skills) and the skill becomes available.

## Files

- `SKILL.md` — the skill definition Claude Code loads
- `LICENSE` — MIT

## License

MIT
