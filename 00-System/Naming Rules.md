---
type: system
version: 0.2
created: 2026-09-03
updated: 2026-09-04
---

# 文件命名规则

## 基本原则

- 名称应当对人可读，同时包含稳定身份。
- 避免使用 `\ / : * ? " < > |`。
- 状态写入YAML，不写入文件名。
- 新建前先检查稳定编号、文件名和aliases。

## 六类文件

```text
论文（有稳定citation key）：citationKey - Short Title.md
论文（没有稳定citation key）：ZOTERO-itemKey - Short Title.md
概念：Canonical Concept Name.md
领域地图：研究范围 + 领域地图.md
研究想法：IDEA-YYYYMMDD-NN - 简短陈述.md
复现项目：REPRO-citationKey - Short Title.md
实验：EXP-YYYYMMDD-NN - 实验目的.md
```

## 例子

```text
lambourne_brepnet_2021 - BRepNet.md
ZOTERO-QB28SJ6R - 电子鼻技术及其应用研究进展.md
B-rep.md
AI for CAD 领域地图.md
IDEA-20260903-01 - 在CAD序列解码中加入局部拓扑约束.md
REPRO-lambourne_brepnet_2021 - BRepNet.md
EXP-20260903-01 - 官方权重推理测试.md
```

## 重命名规则

- `idea_id`、`repro_id`、`experiment_id` 创建后不改变。
- 论文使用的citation key进入知识库后应保持稳定；尚未确认稳定键时，先使用Zotero item key。
- 概念改名时保留旧名称为alias，并检查已有链接。
- 大批量重命名必须先报告影响范围并等待确认。
