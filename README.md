# 项目编排器(Project Orchestrator)

本目录是一个基于 Claude Code 的本地教育咨询方案编排系统。

## 快速开始

1. 在本目录打开 Claude Code 终端
2. Claude Code 会自动加载 `CLAUDE.md`(系统规则)
3. 描述你的项目需求
4. Supervisor 会提议任务级别(A/B/C),你确认后开始

## 第一次使用前建议

- 在 `anchors/positive/` 放 2-3 份你过往的成功方案
- 在 `anchors/index.md` 追加对应条目(类型、优点、缺点、可参照维度)
- 没有 Anchor 也能跑,但 Phase 2 会退化为兜底 rubric 模式

## 我可以改什么

- `config/constraints.md` — 所有可调参数,改完下次 Run 生效
- `config/project-types/` — 项目类型预设,可手动 fork `_template.md`
- `anchors/index.md` 和 `anchors/positive/` / `anchors/negative/` — 样本库

## 我不应该改什么

- `CLAUDE.md` — 改前先看开发指导文档 §11(结构性铁律)
- 任何 `runs/<id>/state.md` — Supervisor 维护,手动改可能破坏恢复

## 目录速查

```
.
├── CLAUDE.md                       # 系统规则(自动加载)
├── config/
│   ├── constraints.md              # 可调参数
│   ├── rubrics.md                  # 评价模板(备用)
│   └── project-types/              # 项目类型预设
├── anchors/                        # 对标样本库
│   ├── index.md
│   ├── positive/
│   └── negative/
├── runs/                           # 每个项目一个子目录
│   └── _state-template.md          # state.md 模板
└── log/
    ├── tuning-history.md           # 调参历史
    └── failure-cases.md            # 失败案例
```

## 文档

- 完整规范:`../project-orchestrator-dev-guide-v0.2.1.md`
- 版本变更:`../CHANGELOG.md`
- 调参历史:`log/tuning-history.md`
- 失败案例:`log/failure-cases.md`
