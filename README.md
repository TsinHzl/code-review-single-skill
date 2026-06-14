# Code Review Skills

*Read this in [简体中文](README_zh.md).*

A Claude Code Code Review skill for single-repo deep review scenarios.

## Skills Overview

| Skill | Description | Trigger Keywords |
|---|---|---|
| `code-review-single` | Single-repo deep review | code review, 代码审查, review, 代码质量检查 |

---

## code-review-single

Single-repo Code Review Expert. Extracts the current branch diff via Git, performs deep review of logic bugs, boundary gaps, readability, performance, and security issues.

### Features

- **Four diff modes** — commit hash, time range, commit count, or source branch comparison (priority: hash > time > count > source)
- **Time range support** — "today", "last 3 days", "last week", "last 2 hours" etc.
- **Component path scoping** — `@src/auth/` to limit review to a subdirectory
- **Deep review** — 6-dimension scan (security, crash/exception, logic errors, performance, code style, refactoring)
- **Structured Markdown report** — written to file via `references/report-template.md`

### Usage

```
/code-review-single                    # Source branch (default)
/code-review-single abc1234            # Commit hash → HEAD
/code-review-single today              # Today's commits
/code-review-single last 3 days        # Last 3 days
/code-review-single @src/auth/         # Scoped to component
```

### Output

```
<working-directory>/<TARGET_COMPONENT>-code-review.md
```

---

## Directory Structure

```
code-review-single-skill/
├── README.md
├── README_zh.md
└── code-review-single/
    ├── SKILL.md
    ├── references/
    │   └── report-template.md
    └── scripts/
        └── code-review-single-env.sh
```

## Report Structure

| Section | Description |
|---|---|
| 📝 Change Summary | Concise overview of what changed |
| 🚫 Critical Issues | Crashes, security holes, data corruption, build failures |
| ⚠️ Improvement Suggestions | Logic errors, boundary gaps, performance, code style |
| 💡 Elegant Refactoring | Concrete refactoring proposals with before/after diffs |
| 🏁 Summary | Score 1–10 + key risk |

## Block Conditions

| Condition | Skill |
|---|---|
| Not a Git repository | single |
| Commit hash not found | single |
| No valid changes in range | single |
| Source branch not found / same as current | single |
| Unparseable time range | single |

## Constraints

- Report output language: **Chinese**
- Auto-generated files (e.g. `package-lock.json`) are skipped
- Reports written to file, not printed to chat
- Each finding is independent — no merging or splitting

## License

[MIT](LICENSE)
