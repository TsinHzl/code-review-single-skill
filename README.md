# code-review-single-skill

Single-repo Code Review Expert. Extracts the current branch diff via Git, performs deep review of logic bugs, boundary gaps, readability, performance, and security issues. Generates a complete Code Review report covering critical issues, improvement suggestions, and elegant refactoring proposals.

## Features

- **Three diff modes** — commit hash, commit count, or source branch comparison
- **Deep review** — logic bugs, boundary gaps, readability, performance, security
- **Structured Markdown report** — written to file, not dumped to chat
- **Auto-skips** generated files (e.g. `package-lock.json`)

## Installation

Place the `code-review-single/` folder into your Claude Code skills directory, then reload the session.

```
code-review-single-skill/
└── code-review-single/
    └── SKILL.md
```

## Usage

Trigger the skill by mentioning **code review**, **review**, or **code quality** in your prompt.

### Mode 1 — Source Branch (default)

Compares the current branch against the branch it was created from.

```
code review
```

### Mode 2 — Commit Count

Reviews the last N commits on the current branch.

```
review the last 3 commits
code review latest 2
```

### Mode 3 — Commit Hash

Diffs from a specific commit hash to HEAD. Takes priority over other modes.

```
code review abc1234
review mf525235
```

### Optional: Component Path

Prefix with `@component-name/` to scope the review to a subdirectory.

```
code review @src/auth/
```

## Output

The skill writes a Markdown report to:

```
<working-directory>/<TARGET_COMPONENT>-code-review.md
```

### Report Structure

| Section | Description |
|---|---|
| 📝 Change Summary | Concise overview of what changed |
| 🚫 Critical Issues | Crashes, security holes, severe logic errors |
| ⚠️ Improvement Suggestions | Standards, readability, best practices |
| 💡 Elegant Refactoring | Concrete refactoring proposals with before/after diffs |
| 🏁 Summary | Score 1–10 + key risk |

Each finding includes: precise issue description, potential impact, line location, before/after diff, and fixed code with highlights.

## Block Conditions

The skill aborts with a `BLOCK:` message when:

| Condition |
|---|
| Not a Git repository |
| Commit hash not found |
| Requested count exceeds total commits |
| Source branch not found |
| Current branch == source branch |
| No code changes in range |

## Constraints

- Report output language: **English**
- Auto-generated files are skipped
- Report is written to file, not printed to chat

## License

[MIT](LICENSE)

---



# code-review-single-skill（中文说明）

单仓库 Code Review 专家。通过 Git 提取当前分支的 diff，对逻辑缺陷、边界问题、可读性、性能和安全漏洞进行深度审查，生成包含严重问题、改进建议和优雅重构提案的完整 Code Review 报告。

## 功能特性

- **三种 diff 模式** —— 提交哈希、提交数量或源分支对比
- **深度审查** —— 逻辑缺陷、边界问题、可读性、性能、安全漏洞
- **结构化 Markdown 报告** —— 写入文件，而非直接输出到对话
- **自动跳过**自动生成的文件（如 `package-lock.json`）

## 安装

将 `code-review-single/` 文件夹放入 Claude Code 的 skills 目录，然后重新加载会话。

```
code-review-single-skill/
└── code-review-single/
    └── SKILL.md
```

## 使用方法

在提示词中提及 **code review**、**review** 或 **code quality** 即可触发本 Skill。

### 模式一：源分支（默认）

将当前分支与其创建来源的分支进行对比。

```
code review
```

### 模式二：提交数量

审查当前分支最近 N 个提交。

```
review the last 3 commits
code review latest 2
```

### 模式三：提交哈希

从指定提交哈希到 HEAD 的 diff，优先级最高。

```
code review abc1234
review mf525235
```

### 可选：组件路径

使用 `@component-name/` 前缀将审查范围限定到子目录。

```
code review @src/auth/
```

## 输出

Skill 将 Markdown 报告写入：

```
<工作目录>/<TARGET_COMPONENT>-code-review.md
```

### 报告结构

| 章节 | 说明 |
|---|---|
| 📝 变更摘要 | 变更内容简述 |
| 🚫 严重问题 | 崩溃、安全漏洞、严重逻辑错误 |
| ⚠️ 改进建议 | 规范、可读性、最佳实践 |
| 💡 优雅重构 | 具体重构提案含前后对比 |
| 🏁 总结 | 1–10 评分 + 关键风险 |

每条发现包含：精确问题描述、潜在影响、代码行位置、修改前后对比、修复代码及要点说明。

## 中止条件

以下情况 Skill 会输出 `BLOCK:` 并中止：

| 条件 |
|---|
| 当前目录不是 Git 仓库 |
| 指定的提交哈希不存在 |
| 请求数量超过总提交数 |
| 找不到源分支 |
| 当前分支与源分支相同 |
| 指定范围内无代码变更 |

## 约束

- 报告输出语言：**英文**
- 自动生成的文件会被跳过
- 报告写入文件，不打印到对话

## 许可证

[MIT](LICENSE)
