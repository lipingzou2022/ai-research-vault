---
type: system
version: 0.2
created: 2026-09-03
updated: 2026-09-04
---

# YAML字段字典

字段名使用英文小写和下划线，状态值使用固定的英文短语，日期使用 `YYYY-MM-DD`。

## 通用字段

| 字段 | 用途 | 允许值或格式 |
|---|---|---|
| `type` | 笔记类型 | 六类科研笔记使用 `literature`、`concept`、`field-map`、`idea`、`reproduction`、`experiment`；规则与模板说明使用内部类型 `system` |
| `note_status` | 笔记检查状态 | `draft`、`reviewed`、`archived` |
| `created` | 创建日期 | `YYYY-MM-DD`，创建后不改变 |
| `updated` | 最近实质更新日期 | `YYYY-MM-DD` |

## 论文笔记

| 字段 | 用途 | 允许值或格式 |
|---|---|---|
| `citekey` | 可选稳定引用键 | 仅在Better BibTeX等工具已确认提供稳定键时写入；普通导出键明显含糊时省略 |
| `zotero_item_key` | Zotero内部条目标识 | 8位条目键 |
| `title` | 标题 | 以Zotero为准 |
| `authors` | 作者 | YAML列表 |
| `year` | 年份 | 四位年份；缺失时省略 |
| `doi` | DOI | 缺失时省略 |
| `reading_level` | 阅读深度 | `screening`、`standard`、`core` |
| `reading_status` | 阅读进度 | `unread`、`screened`、`reading`、`discussed` |
| `zotero_tags` | Zotero原始标签 | 原样保存的YAML列表 |

## 概念笔记

| 字段 | 用途 | 允许值或格式 |
|---|---|---|
| `maturity` | 理解成熟度 | `seed`、`developing`、`stable`、`needs-review` |
| `aliases` | 同一概念的其他名称 | YAML列表 |
| `concept_kind` | 可选概念类别 | `task`、`method`、`model`、`module`、`representation`、`dataset`、`metric`、`problem` |

## 领域地图

| 字段 | 用途 | 允许值或格式 |
|---|---|---|
| `maturity` | 地图成熟度 | `seed`、`structured`、`comparative`、`research-ready` |
| `parent_map` | 可选上级地图 | 加引号的Obsidian双链 |

## 研究想法

| 字段 | 用途 | 允许值或格式 |
|---|---|---|
| `idea_id` | 稳定编号 | `IDEA-YYYYMMDD-NN` |
| `maturity` | 想法成熟度 | `seed`、`connected`、`hypothesis`、`test-ready` |
| `validation` | 证据状态 | `untested`、`testing`、`inconclusive`、`partially-supported`、`supported`、`contradicted` |
| `decision` | 当前决策 | `active`、`paused`、`adopted`、`rejected` |
| `novelty_status` | 查新状态 | `unchecked`、`searching`、`close-work-found`、`no-close-match-found` |

## 复现项目

| 字段 | 用途 | 允许值或格式 |
|---|---|---|
| `repro_id` | 稳定编号 | `REPRO-citekey` |
| `paper` | 对应论文笔记 | 加引号的Obsidian双链 |
| `repository` | 代码仓库 | URL |
| `branch` | 基准分支 | Git分支名称 |
| `upstream_commit` | 上游精确版本 | Git commit |
| `reproduction_status` | 工程状态 | `planned`、`code-understanding`、`environment-ready`、`baseline-running`、`evaluating`、`paused`、`completed` |
| `reproduction_level` | 已达到的级别 | `L0`至`L5`，加引号 |

## 实验记录

| 字段 | 用途 | 允许值或格式 |
|---|---|---|
| `experiment_id` | 稳定编号 | `EXP-YYYYMMDD-NN` |
| `project` | 对应复现项目 | 加引号的Obsidian双链 |
| `idea` | 可选研究想法 | 加引号的Obsidian双链 |
| `execution_status` | 执行状态 | `planned`、`running`、`completed`、`aborted` |
| `outcome` | 结论状态 | `not-evaluated`、`supported`、`partially-supported`、`contradicted`、`inconclusive`、`invalid` |
| `code_commit` | 实际代码版本 | Git commit |
| `started` | 开始日期 | `YYYY-MM-DD` |
| `completed` | 完成日期 | `YYYY-MM-DD`，未完成时省略 |

## 维护原则

- 不生成没有用途的空字段。
- YAML只保存用于定位、筛选和状态管理的信息。
- 论文解释、证据、局限和判断写在正文。
- 影响科研结论的状态由用户确认后更新。
- 官方来源与Zotero元数据不一致时，在正文记录差异；未经明确要求不反写Zotero。
