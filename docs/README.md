---
title: docs 入口
tags: [orchestrator/index]
---

# docs/

人机交互同步渠道。三份 wiki 文档，分层覆盖不同读者：

| 文件 | 给谁 | 一句话 |
|------|------|--------|
| [[user-guide]] | 用这个项目辅助工作的人 | 入口、出口、交互、可控手柄 |
| [[architecture-tuning]] | 想理解结构、想调参、想改架构的人 | 四阶段流水线、state 状态机、调参面板 |
| [[deep-optimization]] | 想从内核原理改造系统的人 | 借鉴的范式、可深度优化的方向 |

## 阅读方式

**Obsidian**：把这个 `docs/` 目录当作 vault 打开（或加为已有 vault 的子库），三份文档之间是 `[[wikilink]]` 互链，反向链接和图谱视图都能用。

**GitHub / 普通编辑器**：直接读。Wikilink 在 GitHub 上不渲染为链接但显示为可读文字，不影响理解；三份文档自身是独立完整的。

## 维护约定

- 三份文档之间 wikilink 用文件 basename（不写路径，不写 `.md`）
- 引用项目内其他文件用相对路径（`../config/constraints.md`）
- 文档与代码同步：改了 `CLAUDE.md` 或 `config/constraints.md` 的字段，对应回来更新这里
