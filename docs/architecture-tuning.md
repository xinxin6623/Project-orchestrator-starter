---
title: 架构与调参面板
tags: [orchestrator/architecture, wiki]
aliases: [架构, 调参, 架构调参]
---

# 架构与调参面板

> 给"想理解系统结构、想调架构、想加功能"的人看。
>
> 配套：[[user-guide]]（怎么用）· [[deep-optimization]]（深度优化方向）

---

## 架构一图

```
┌─────────────────────────────────────────────────────────┐
│ CLAUDE.md  (系统规则，Claude Code 启动自动加载)          │
│  ├ §1 结构性铁律（不可豁免）                              │
│  └ §2 流程性默认（可豁免，必须留痕）                      │
└────────────┬────────────────────────────────────────────┘
             ▼
    ┌──────────────────┐         读 ┌────────────────────┐
    │   Supervisor     │ ◀──────────│ config/constraints │
    │  （主智能体）     │            └────────────────────┘
    └────────┬─────────┘
             │ 维护
             ▼
    ┌──────────────────┐
    │   state.md       │ ◀──── 状态机（每个 Run 一份）
    │  断点续跑核心     │
    └────────┬─────────┘
             │
             ▼ 派工（注入 5 段提示词）
    ┌──────────────────────────────────┐
    │ Worker 1  Worker 2  Worker 3 ... │ 无全局上下文，单任务
    └──────────────────────────────────┘
             │
             ▼ 验收（对照 anchor 或 rubric）
    ┌──────────────────┐
    │   final.md       │
    └──────────────────┘
```

核心抽象：**Supervisor/Worker 双角色** + **四阶段流水线** + **state.md 状态机** + **anchor 样本库**。详见后文。

---

## 四阶段流水线

| Phase | 名字 | 干什么 | 产出 | 调参入口 |
|-------|------|--------|------|----------|
| 1 | **Frame** | 把模糊需求结构化成"已知/未知/假设"三栏 | `01-frame.md` | [[constraints#brainstorm]] |
| 2 | **Anchor** | 从样本库选一个参照，做适配性检查 | `02-anchor.md` | [[constraints#anchor]] |
| 3 | **Decompose** | 拆成 10-30 分钟的原子任务，标串行/并行 | `03-decompose.md` | [[constraints#decompose]] |
| 4 | **Execute** | 派 Worker 执行 + 验收 + 整合 | `04-execute/*.md` + `final.md` | [[constraints#execute]] |

**B 级**只跑 Phase 1 + Phase 4（中间夹个 Mini-plan），**A 级**根本不进流水线。判级逻辑见后。

---

## 任务级别判定

`CLAUDE.md §3` 定义了 5 问完整版，Supervisor **默认走简短判定**（直接给建议+理由），只有以下情况展开 5 问：

- 用户首次使用前 3 个 Run
- Supervisor 自己难判
- 用户质疑建议

5 问 → Yes 数：
- ≥3 → C 级
- 1-2 → B 级
- 0 → A 级

**Supervisor 不得自降级**（防偷懒），这是 §1 铁律之外的硬约束。

---

## state.md 状态机

**最关键的设计**。每个 Run 一份 `state.md`，机器可读，记录：

- `current_phase` / `next_action` — 跑到哪、下一步做什么
- `completed_files` — 已落地的产出
- `anchor_status` / `skipped_reason` / `risk_note` — 任何豁免都留痕
- `workers_in_flight` / `workers_completed` — Phase 4 派工情况
- `config_snapshot` — Run 开始时的 git HEAD（防止中途改参数污染）

**读写规则**（[[claude-md#§1-结构性铁律|铁律]]）：以下"状态变化点"必须读写，其他动作不写（避免成本失控）：

1. 阶段开始前（读）
2. 阶段结束后（写）
3. Worker 派发/完成/失败（写）
4. 用户覆盖默认流程（写）
5. 失败恢复（读+写）
6. 会话重启（读）

**用户禁忌**：手改 `state.md` 可能破坏断点续跑。要改让 Supervisor 改。

---

## Anchor 机制

Phase 2 的核心。系统假设：**对标一份具体样本，比用抽象 rubric 评成品更准**。

### 数据结构

```
anchors/
├── index.md          # 索引，所有样本的元数据
├── positive/         # 成功样本（要学的）
├── negative/         # 失败样本（要避的）
└── _raw/             # 原始 PDF/docx，不入库（.gitignore）
```

每条索引项包含：**类型 / 适用项目 / 优点 / 缺点 / 可参照维度 / 来源 / 加入时间**。Supervisor 用这些字段做匹配。

### 适配性检查

提议 anchor 后做 3 维度检查（默认阈值 2/3 通过，[[constraints#anchor|anchor_fitness_threshold]]）：

1. 类型匹配
2. 学段/对象匹配
3. 结构可比

不通过 → Supervisor 提议换样本、双 anchor 互补、或退化 rubric_only。

### 跳过路径

库里实在没匹配的（如 `runs/2026-05-21-5th-grade-carbon-shiyou` 那次）：用户可选 `rubric_only`，但 Supervisor 必须先复述 ≥2 条风险后果，确认后写入 `skipped_reason` + `risk_note`。

---

## 调参手柄

**全部在 [[constraints|config/constraints.md]]**，下次 Run 生效，进行中的 Run 不受影响。

### `[brainstorm]` Phase 1 澄清
- `mode`: `always` / `dynamic` / `off`
- `max_questions`: 默认 3
- `must_ask_about`: 默认 [年级, 地方政策约束, 验收方]
- `skip_if_template_match`: 命中 project-type 预设时跳澄清

### `[anchor]` Phase 2
- `require_anchor`: 是否强制
- `allow_supervisor_suggest`: 允许 Supervisor 自己起草样本
- `require_reverse_anchor`: 是否必须挂一个反面样本
- `anchor_fitness_threshold`: 3 维度要过几个（默认 2）

### `[decompose]` Phase 3
- `atomic_duration_minutes`: 默认 [10, 30]
- `max_parallel_workers`: 默认 3
- `serial_only_tasks` / `parallelizable_tasks`: 任务类型分类
- `require_acceptance_signal`: 每个任务必须有验收信号
- `require_input_dependencies`: 必须声明输入依赖

### `[execute]` Phase 4
- `review_mode`: `anchor_based` / `rubric_only` / `hybrid`
- `human_checkpoint`: `per_phase` / `per_project` / `off`
- `max_rework_rounds`: 默认 2

### `[meta]` 调参建议触发
- `suggest_when_phase_overshoot_ratio`: 默认 2.0
- `suggest_when_user_interrupt_count`: 默认 2
- `suggest_when_rework_count`: 默认 2
- `suggest_when_user_uses_words`: ["太细", "太多", "跳过", "重来"]
- `silent_mode`: 第 1 周建议 true，实测后改 false

---

## 角色与边界

- **Supervisor**（默认角色）：全局视野，协调四阶段，可自任串行任务（如"整体框架"、"叙事线"、"最终整合"）
- **Worker**（派生子智能体）：仅执行单一原子任务，**无全局上下文**

派 Worker 必须注入完整 5 段提示词（[[claude-md#§1-结构性铁律|铁律 3]]）：
1. 身份
2. 任务定义
3. 输入依赖
4. 输出格式
5. 边界（不许干什么）

Worker 不自动继承 `CLAUDE.md`，所以这 5 段是它的全部世界。

---

## 当前架构的优化空间

下面是看得见、可以动手的层面。**架构内核层的优化**（认知科学层、系统理论层）见 [[deep-optimization]]。

### 1. project-types 预设系统**未激活**

`config/project-types/_template.md` 摆着，但实际 Run 都是 `untyped`（看 `runs/2026-05-21-5th-grade-carbon-shiyou/state.md` 的 `project_type` 字段）。激活后能：
- 命中类型时跳澄清（`skip_if_template_match: true`）
- 预填 must_ask_about
- 自动绑定推荐 anchor

**改造点**：补 2-3 个真实类型预设（如"小学研学方案"、"教研评估文档"），跑通匹配逻辑。

### 2. anchor 索引是手动维护的

`anchors/index.md` 每加样本要手动追加元数据。改造方向：
- 在样本文件顶部用 YAML frontmatter 写元数据，写个脚本/Worker 自动重建 index
- 或者直接让 Supervisor 在加样本时同步写索引（已部分实现，但靠提醒）

### 3. state.md 是 YAML-in-Markdown，容易手抖写坏

考虑两条路：
- **保守**：加一个 lint 脚本，每次写完跑一遍
- **激进**：抽离成 `state.json`，markdown 只做人类视图

### 4. 调参建议块（meta）触发了但没"学"

`silent_mode: true` 时建议块不弹，但触发记录也没存。改造：把触发事件落到 `log/meta-triggers.md`，定期回顾哪些参数应该改默认值。

### 5. failure-cases 是被动记录，没有 pattern 抽取

`log/failure-cases.md` 只追加，没人回头看。可以加一步：每 N 次 failure 触发一个"模式归纳" Worker，把共性写进 `log/anti-patterns.md`，作为 Supervisor 派工前的检查项。

### 6. anchor 跳过率应该是个监控指标

`runs/*/state.md` 里 `anchor_status: skipped` 多了说明样本库有结构性盲区。当前没有汇总视图。改造：写个 `bin/anchor-coverage.sh` 输出覆盖率。

### 7. Worker 5 段提示词模板没有抽离

现在散在开发指导文档 §6.2 和 Supervisor 的脑子里。建议抽到 `config/worker-prompt-template.md`，Supervisor 派工时 include 它，便于版本化。

---

## 结构性铁律（改之前必读）

`CLAUDE.md §1` 列了 6 条铁律。**调架构时这 6 条不能违反**，否则系统的"可恢复"保证失效：

1. state.md 必须在 6 个状态变化点维护
2. 文件结构必须遵守 Phase 1-4 模板
3. Worker 必须用 5 段模板派工
4. 跳过必须记录 skipped_reason + risk_note
5. 配置变更必须记到 `log/tuning-history.md`
6. Phase 1 的"已知"必须可追溯到用户原文

**为什么是铁律**：每条都对应一个"如果违反，断点续跑/事后追溯/失败归因 至少一个会废"的失效模式。

---

## 延伸

- 怎么用 → [[user-guide]]
- 内核层借鉴的范式、深度优化方向 → [[deep-optimization]]
- 完整规范 → `../../project-orchestrator-dev-guide-v0.2.1.md`（如有）
- 调参历史 → `../log/tuning-history.md`
- 失败案例 → `../log/failure-cases.md`
