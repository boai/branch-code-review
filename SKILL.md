---
name: branch-code-review
description: Use when the user asks to review code changes between a feature branch and a base branch (main/master) before merging, generate a structured Markdown code review document with severity-tagged findings (severe/high/medium) saved to CodeReviews/<repo>.md, or audit a stack of commits prior to PR
---

# Branch Code Review

## Overview

Generate a structured Markdown code review for the diff between a **target** (feature) branch and a **base** (main/master) branch. Output is a single `.md` file with YAML frontmatter, severity-tagged findings, and `file:line` citations grounded in the actual diff.

Core principle: review **what the diff actually shows**, never fabricate citations, and keep every list item on its own line.

## When to Use

- User asks "review my branch", "审一下这个分支", "code review", "评审分支" or similar
- Before opening or merging a PR
- After a stack of commits and you want a structured assessment
- When the user mentions producing a `.md` review file

**Don't use when:**
- The user wants inline feedback on a single uncommitted file → just review directly, no document
- The repo has no `main`/`master` and the user can't name a base branch → ask, don't guess

## Process

Walk through the steps in order. Ask the user **only** when something is genuinely ambiguous (multiple plausible base branches, unclear output path).

### 1. Locate the repo

```bash
git rev-parse --is-inside-work-tree
```

If not inside a repo, ask the user for the absolute path and `cd` there.

### 2. Resolve target & base branches

```bash
# target = current branch
git rev-parse --abbrev-ref HEAD

# base detection, in order:
git symbolic-ref refs/remotes/origin/HEAD --short 2>/dev/null   # e.g. "origin/main"
# strip "origin/" prefix → base
# fallback: try main, then master
git rev-parse --verify --quiet main
git rev-parse --verify --quiet master
```

Announce your detection to the user before proceeding:

> 检测到 target=`<branch>`, base=`<branch>`，输出到 `./CodeReviews/<repo>.md`，确认开始？

If detection fails entirely, ask the user to specify base.

### 3. Collect diff + stats + commits

Use this canonical exclude list (mirrors the noise filter from the original tool):

```bash
git diff <base>...<target> -- . \
  ':(exclude,glob)*.lock' \
  ':(exclude,glob)Podfile.lock' \
  ':(exclude,glob)package-lock.json' \
  ':(exclude,glob)yarn.lock' \
  ':(exclude,glob)pnpm-lock.yaml' \
  ':(exclude,glob)Cargo.lock' \
  ':(exclude,glob)Gemfile.lock' \
  ':(exclude,glob)composer.lock' \
  ':(exclude,glob)*.pbxproj' \
  ':(exclude,glob)*.xcworkspacedata' \
  ':(exclude,glob)*.xcuserstate' \
  ':(exclude,glob)xcuserdata/**' \
  ':(exclude,glob)*.min.js' \
  ':(exclude,glob)*.min.css' \
  ':(exclude,glob)Pods/**' \
  ':(exclude,glob)node_modules/**' \
  ':(exclude,glob).build/**' \
  ':(exclude,glob)build/**' \
  ':(exclude,glob)*.svg' \
  ':(exclude,glob)*.png' \
  ':(exclude,glob)*.jpg' \
  ':(exclude,glob)*.jpeg' \
  ':(exclude,glob)*.gif' \
  ':(exclude,glob)*.ico' \
  ':(exclude,glob)*.pdf' \
  ':(exclude,glob)*.zip' \
  ':(exclude,glob)*.mp3' \
  ':(exclude,glob)*.mp4' \
  ':(exclude,glob)*.mov' \
  ':(exclude,glob)*.ttf' \
  ':(exclude,glob)*.otf' \
  ':(exclude,glob)*.woff' \
  ':(exclude,glob)*.woff2' \
  ':(exclude,glob)*.snap'
```

Then:
```bash
git diff --shortstat <base>...<target>
git log <base>..<target> -200 --pretty=format:'%h|%s'
```

**Empty diff** → stop, tell the user there is nothing to review (e.g., branches identical or all changes filtered).

**Diff > 30,000 chars** → truncate to the first 30,000 chars for analysis; record the original length and add a truncation warning to the output header.

### 4. Analyze the diff yourself

You are the reviewer. Apply this severity rubric:

| Level | Examples |
|-------|----------|
| 严重 (severe) | Crashes, data loss/corruption, leaks, security holes, plaintext secrets, dangling pointers |
| 高 (high) | Clear bugs, retain cycles, missing debounce, sensitive data in logs, missing error handling on critical paths |
| 中 (medium) | Readability, naming, duplication, magic numbers/hardcoding, style |

**Rules — non-negotiable:**
- Each finding is **one sentence**.
- Every `file:line` citation must be findable in the diff body. Do not fabricate.
- Don't restate what the diff already shows; explain the *problem*.
- Empty severity bucket → write `_暂无_` under that section, never delete the section.
- Every list item occupies its own line, prefixed with `- ` (or as a table row). **Never compress multiple points into one line or one blockquote.**

### 5. Compose the Markdown

Use exactly this structure. Sections must appear in this order:

```markdown
---
title: "<repo_name> 分支代码评审记录"
tags: ["code-review", "<repo_name>", "<target_branch>", "<YYYY-MM>"]
created: <ISO-8601 with local timezone offset, e.g. 2026-05-20T14:32:18+08:00>
updated: <same as created>
sources: []
links: []
category: session-log
confidence: medium
schemaVersion: 1
---

# <repo_name> 分支代码评审记录

**评审日期**: <YYYY-MM-DD>
**仓库**: <repo_name>
**分支**: <target> vs <base>
**规模**: <N> 个文件变更 | +<insertions> / -<deletions> 行 | <commit_count> 次提交

> ⚠️ 完整 diff 共 <total_chars> 字符，已按前 30000 字符送审；评审结论可能未覆盖被截断部分。
（仅截断时输出上一行；否则省略）

## 概览

- 要点 1（2-4 条，每条独占一行）
- 要点 2
- 要点 3

## 严重问题（必须修复）

| # | 问题 | 文件:行号 |
|---|------|-----------|
| 1 | … | path/to/file.swift:42 |

（如无则替换整张表为 `_暂无_`）

## 高优先级问题（应当修复）

| # | 问题 | 文件 |
|---|------|------|
| 1 | … | path/to/file.swift |

（如无则 `_暂无_`）

## 中等优先级问题（建议修复）

- 一句话描述
- 一句话描述

（如无则 `_暂无_`）

## 正面评价

- 一句话肯定
- 一句话肯定

## 结论

> 一句话总评。
```

Generate the timestamp with local timezone offset:
```bash
date '+%Y-%m-%dT%H:%M:%S%z' | sed -E 's/([+-][0-9]{2})([0-9]{2})$/\1:\2/'
# → 2026-05-20T14:32:18+08:00
```

### 6. Write the file

```bash
mkdir -p CodeReviews
```

Use the `Write` tool to save to `CodeReviews/<repo_name>.md` (where `<repo_name>` is the basename of the working tree, e.g. via `basename "$(git rev-parse --show-toplevel)"`).

If the file already exists, ask the user before overwriting.

### 7. Report

Tell the user:
- Absolute path of the written file
- Diff stats (files / +ins / -del / commits)
- Any warnings (truncation, missing base, files skipped)

## Quick Reference

| Step | Command |
|------|---------|
| Verify repo | `git rev-parse --is-inside-work-tree` |
| Target branch | `git rev-parse --abbrev-ref HEAD` |
| Default base | `git symbolic-ref refs/remotes/origin/HEAD --short` |
| Filtered diff | `git diff base...target -- . ':(exclude,glob)…'` (see step 3) |
| Stat | `git diff --shortstat base...target` |
| Commits | `git log base..target -200 --pretty=format:'%h\|%s'` |
| Timestamp | `date '+%Y-%m-%dT%H:%M:%S%z' \| sed -E 's/([+-][0-9]{2})([0-9]{2})$/\1:\2/'` |
| Write | `Write` tool → `CodeReviews/<repo>.md` |

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Compressing 多个要点 into one `> ` blockquote line | Use `- ` bullets, each on its own line |
| Fabricating `file:line` not present in the diff | Cite only what the diff body actually contains |
| Writing UTC timestamps in frontmatter | Use `date '+...%z'`, insert the colon (`+08:00`) |
| Reviewing >30K-char diff without truncation | Cap at 30K, add the truncation warning line |
| Skipping the empty-diff check | If filtered diff is empty, stop and tell the user |
| Forgetting `mkdir -p CodeReviews` | Directory may not exist; create it first |
| Deleting empty severity sections | Sections are fixed; write `_暂无_` instead |

## Verification

After writing, sanity check:
```bash
head -20 CodeReviews/<repo>.md      # frontmatter looks right, timezone offset present
grep -c '^## ' CodeReviews/<repo>.md # should be 6 (概览/严重/高/中/正面/结论)
```
