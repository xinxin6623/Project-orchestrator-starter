---
title: Run 索引 · 5 年级低碳校园研学方案
tags: [orchestrator/run-index, wiki]
run_id: 2026-05-21-5th-grade-carbon-shiyou
task_level: C
---

# Run 索引 · 北京市石油附属实验小学 5 年级低碳校园主题研学方案

> 这是一次 **C 级 Run**（完整四阶段交付）的过程档案索引。
> 给"想回看这个 Run 怎么跑出来的"人看：从用户一句话需求 → 最终 33KB 教师手册，中间每一步都留在这里。
>
> **只想看成品** → [04-execute/final.md](04-execute/final.md)
> **想看整体复盘** → [review.md](review.md)
> **想看当前状态机** → [state.md](state.md)
>
> 配套阅读：[[../../docs/user-guide]] · [[../../docs/architecture-tuning]] · [[../../docs/deep-optimization]]

---

## 一句话回顾

用户需求："给北京市石油附属实验小学做一份 5 年级低碳校园主题的研学方案"。
判级 **C 级**（完整方案、对外交付、多模块），跑 Frame → Anchor（跳过）→ Decompose → Execute 四阶段，10 个原子任务（Supervisor 自任 6 个 + Worker 派工 4 个），产出 33KB 教师内部参考手册 v1.0。

**特殊点**：Phase 2 Anchor **跳过**（库内仅动物饲养类样本，主题不重叠），改走 `rubric_only` 验收。这是触发 [[../../docs/architecture-tuning#Anchor 机制]] 跳过路径的真实案例。

---

## 文件结构地图

```
2026-05-21-5th-grade-carbon-shiyou/
├── index.md            ← 本文件（索引）
├── state.md            ★ 状态机（机器读，Supervisor 维护，断点续跑核心）
├── 00-input.md         用户原始需求 + 5 问判级记录
├── 01-frame.md         Phase 1: 已知/未知/假设 三栏
├── 02-anchor.md        Phase 2: Anchor 状态（本次=SKIPPED）+ rubric 替代模板
├── 03-decompose.md     Phase 3: 10 个原子任务（耦合性扫描 + DAG）
├── 04-execute/         Phase 4: 执行产出目录
│   ├── supervisor-1.1.md   ┐
│   ├── worker-2.1.md       │
│   ├── worker-2.2.md       │ 10 个任务产出
│   ├── worker-2.3.md       │ supervisor-* = Supervisor 自任串行任务
│   ├── worker-2.4.md       │ worker-*     = 派给 Worker 的并行任务
│   ├── supervisor-3.1.md   │
│   ├── supervisor-4.1.md   │
│   ├── supervisor-4.2.md   ┘
│   └── final.md        ★ 整合后的最终方案（给用户的成品）
└── review.md           rubric 自查报告（Anchor 跳过时的替代验收）
```

---

## 各文件中文说明（按生命周期顺序）

### 🟢 入口与状态

| 文件 | 一句话 | 何时产生 | 谁维护 |
|------|--------|----------|--------|
| [state.md](state.md) | **状态机**。记录跑到哪、下一步做什么、所有豁免/风险/调参快照。**用户不要手改** | 每次状态变化点 | Supervisor |
| [00-input.md](00-input.md) | 用户原始需求一字未改地留档，外加 5 问判级答案。Phase 1 的"已知"必须能追溯回这里（[[../../docs/architecture-tuning#结构性铁律 改之前必读|铁律 6]]） | Run 开始 | Supervisor |

### 🟡 Phase 1-3：把模糊需求结构化

| 文件 | 一句话 | 关键产出 |
|------|--------|---------|
| [01-frame.md](01-frame.md) | **Phase 1 Frame**。把"做个低碳研学方案"拆成三栏：已知（来自用户原文）/ 未知（待澄清）/ 假设（Supervisor 拟定，需用户确认） | 输出"项目边界备忘" |
| [02-anchor.md](02-anchor.md) | **Phase 2 Anchor**（本次状态：**SKIPPED**）。理由：[anchors/](../../anchors/) 库内只有动物饲养类样本，与低碳主题不重叠。改走 rubric_only 验收 + 反 anchor 检查清单 | `skipped_reason` + `risk_note` + 替代 rubric |
| [03-decompose.md](03-decompose.md) | **Phase 3 Decompose**。10 个原子任务，按耦合性分三档：tight（4 个串行，Supervisor 自任）/ medium（2 个串行 +共享上下文）/ loose（4 个并行，可派 Worker） | 任务清单 + DAG + 验收信号 |

### 🔵 Phase 4：执行（04-execute/）

> **命名规则**：`supervisor-N.M.md` = Supervisor 自任的串行任务；`worker-N.M.md` = 派给无全局上下文子智能体的并行任务。`N` 是阶段号，`M` 是阶段内任务号。

| 文件 | 任务 | 类型 | 干了什么 |
|------|------|------|---------|
| [04-execute/supervisor-1.1.md](04-execute/supervisor-1.1.md) | 1.1 制定方案整体框架与价值主张 | tight | 定下"看见-测量-行动"三步叙事、设计哲学、边界声明 |
| [04-execute/worker-2.1.md](04-execute/worker-2.1.md) | 2.1 政策与课标对接 | loose | 检索北京市低碳教育政策 + 科学/劳动课标对接点 |
| [04-execute/worker-2.2.md](04-execute/worker-2.2.md) | 2.2 学情适配 | loose | 5 年级（10-11 岁）认知发展特点 + 低碳主题教学适配建议 |
| [04-execute/worker-2.3.md](04-execute/worker-2.3.md) | 2.3 同类案例参考 | loose | 国内小学低碳研学案例 3-5 个，提取共性与差异 |
| [04-execute/worker-2.4.md](04-execute/worker-2.4.md) | 2.4 季节性资源 | loose | 北京秋季校园低碳相关资源（落叶、食物、能源场景） |
| [04-execute/supervisor-3.1.md](04-execute/supervisor-3.1.md) | 3.1 活动主线设计 | tight | 5 个研学环节按"问题→探究→产出"递进串成日程 |
| [04-execute/supervisor-4.1.md](04-execute/supervisor-4.1.md) | 4.1 教师手册详案 | medium | 每个环节四段式细案：活动简述/组织实施/引导要点/注意事项 |
| [04-execute/supervisor-4.2.md](04-execute/supervisor-4.2.md) | 4.2 安全 / 资源 / 评价 | medium | 安全伦理清单 + 物料场地人员 + 评价方案 + 学习单模板 |
| **[04-execute/final.md](04-execute/final.md)** ★ | 整合 | tight | **给用户的最终成品**：v1.0 教师内部参考手册 11 章 + 3 附录 |

### 🟣 Phase 4 之后：验收与复盘

| 文件 | 一句话 |
|------|--------|
| [review.md](review.md) | **rubric 自查报告**。因 Anchor 跳过，按 [config/rubrics.md](../../config/rubrics.md) 逐项自查，记录达标项 / 待改进项 / 风险残留 |

---

## 这个 Run 的关键决策与豁免

> 任何偏离默认流程的决定都在 [state.md](state.md) 留痕。这里只汇总要点：

| 时间 | 决策 | 理由 | 影响 |
|------|------|------|------|
| Phase 2 入口 | **跳过 Anchor**（`anchor_status: skipped`） | 库内仅动物饲养类，主题不重叠 | 改 `rubric_only` 验收，可信度下降；用户已确认 |
| Phase 3 | 10 任务中 6 个由 Supervisor 自任 | 涉及叙事一致性 / 价值主张 / 跨模块衔接 / 最终整合，全是 tight 耦合 | Worker 派工数偏少，但保证叙事不撕裂 |
| Phase 4 整合 | 把"半天紧凑版"做成附录 C 而非另起 Run | 用户原始需求未明确单日/半天，单 Run 内提供双方案选择 | 增加 ~3KB 篇幅 |

---

## 给后来人的提示

1. **要复盘这次跑得怎么样** → 读 [review.md](review.md) + [state.md](state.md) 的 `skipped_reason` / `risk_note` 段
2. **要把这次成果入样本库** → 用户确认成品质量后，把 [04-execute/final.md](04-execute/final.md) 拷贝到 [anchors/positive/](../../anchors/positive/)，并在 [anchors/index.md](../../anchors/index.md) 追加元数据。这是当前样本库**首个低碳主题**的候选 positive anchor
3. **要复用任务拆解** → [03-decompose.md](03-decompose.md) 的 10 任务结构对"小学主题研学方案"类项目可直接照搬，建议抽离到 [config/project-types/](../../config/project-types/) 做预设
4. **基于本 Run 的反馈触发的系统改造** → 见 [log/tuning-history.md](../../log/tuning-history.md) 2026-05-22 条目（新增 `require_design_rationale` + Anchor 适配性扩到 4 维 + `time_budget_feasibility`）

---

**版本**：v1.0 · 索引建立于 2026-05-22 · 由 Supervisor 回填生成
