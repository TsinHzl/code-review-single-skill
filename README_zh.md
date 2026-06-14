# Code Review Skills

*Read this in [English](README.md).*

一个 Claude Code Code Review 技能，用于单仓库深度审查。

## 技能概览

| 技能 | 说明 | 触发关键词 |
|---|---|---|
| `code-review-single` | 单仓库深度审查 | code review, 代码审查, review, 代码质量检查 |

---

## code-review-single（单仓库 Code Review 专家）

### 功能

- **四种 diff 模式** — commit hash（暂未实现）、时间范围、commit 数量（暂未实现）、源分支对比（优先级：hash > 时间 > 数量 > 源分支）
- **时间范围支持** — "今天内"、"最近三天"、"最近一周"、"最近两小时"等
- **组件路径限定** — `@src/auth/` 限定审查子目录
- **变更统计** — 深度扫描前自动统计文件数、新增/删除/净变更行数、提交数
- **六维深度扫描** — 安全漏洞、崩溃与异常、逻辑错误、性能问题、代码规范、重构机会
- **结构化报告** — 通过 `references/report-template.md` 模板输出

### 使用方式

```
/code-review-single                    # 源分支（默认）
/code-review-single 今天内             # 今天的提交
/code-review-single 最近三天           # 最近 3 天
/code-review-single @src/auth/         # 限定组件路径
```

> **注意：** commit hash 和 commit 数量模式暂未实现，传入时会被静默忽略，等效于源分支全量对比。

### 输出路径

```
<工作目录>/<TARGET_COMPONENT>-code-review.md
```

---

## 目录结构

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

## 报告结构

| 章节 | 说明 |
|---|---|
| 📊 变更统计 | 文件数、新增/删除/净变更行数、提交数、每文件变更类型 |
| 📝 变更概述 | 变更内容简述 |
| 🚨 深度审查 — 🚫 严重问题 | 崩溃、安全漏洞、数据损坏、构建失败 |
| 🚨 深度审查 — ⚠️ 改进建议 | 逻辑错误、边界遗漏、性能问题、代码规范 |
| 🚨 深度审查 — 💡 优雅重构 | 具体重构提案含前后对比 |
| 🏁 总结评述 | 评分公式：`10 − (严重×3) − (改进×0.5) − (重构×0.2)`，最低 1.0 分 |

## 阻断条件

| 条件 | 技能 |
|---|---|
| 非 Git 仓库 | single |
| Commit hash 未找到 | single |
| 时间范围内无有效变更 | single |
| 源分支不存在或与当前分支相同 | single |
| 时间范围无法解析（会向用户提出澄清问题） | single |

## 约束

- 报告语言：**中文**
- 自动生成文件（`package-lock.json` 等）会被跳过
- 报告写入文件，不打印到对话
- 每条发现独立，不合并、不拆分
- **非线性历史说明：** 在含 merge commit 或 cherry-pick 的仓库中，时间范围模式的 diff 结果为最老 in-range commit 的父节点到 HEAD 的累积变更，可能包含少量时间范围外的提交内容。报告会在变更概述中注明此限制。

## 许可证

[MIT](LICENSE)
