---
title: Run 索引 · <项目短名>
tags: [orchestrator/run-index, wiki]
run_id: <YYYY-MM-DD-项目短名>
task_level: <B | C>
---

# Run 索引 · <项目全名>

> 这是一次 **<级别> 级 Run** 的过程档案索引。
> 给"想回看这个 Run 怎么跑出来的"人看：从用户一句话需求 → 最终成品，中间每一步都留在这里。
>
> **只想看成品** → [04-execute/final.md](04-execute/final.md)
> **想看整体复盘** → [review.md](review.md)
> **想看当前状态机** → [state.md](state.md)
>
> 配套阅读：[[../../docs/user-guide]] · [[../../docs/architecture-tuning]] · [[../../docs/deep-optimization]]

---

## 一句话回顾

用户需求："<原话或一句话摘要>"。
判级 **<级别> 级**（<判级理由>），跑 <实际跑了哪些阶段>，<N> 个原子任务（Supervisor 自任 <X> 个 + Worker 派工 <Y> 个），产出 <成品形态与体量>。

**特殊点**：<跳过 / 豁免 / 不同寻常的决策；如全部走默认，写"全程走默认流程，无豁免">

---

## 文件结构地图

```
<YYYY-MM-DD-项目短名>/
├── index.md            ← 本文件（索引）
├── state.md            ★ 状态机（机器读，Supervisor 维护）
├── 00-input.md         用户原始需求
├── 01-frame.md         Phase 1: 已知/未知/假设 三栏
├── 02-anchor.md        Phase 2: Anchor 状态（C 级才有）
├── 03-decompose.md     Phase 3: 原子任务清单（C 级才有）
├── 04-execute/         Phase 4: 执行产出目录
│   ├── supervisor-*.md   Supervisor 自任串行任务
│   ├── worker-*.md       Worker 派工并行任务
│   └── final.md        ★ 整合后的最终成品
└── review.md           rubric 自查或 anchor 差异分析
```

---

## 各文件中文说明（按生命周期顺序）

### 🟢 入口与状态

| 文件 | 一句话 | 何时产生 | 谁维护 |
|------|--------|----------|--------|
| [state.md](state.md) | **状态机**。记录跑到哪、下一步做什么、所有豁免/风险/调参快照。**用户不要手改** | 每次状态变化点 | Supervisor |
| [00-input.md](00-input.md) | 用户原始需求一字未改地留档 | Run 开始 | Supervisor |

### 🟡 Phase 1-3：把模糊需求结构化

| 文件 | 一句话 | 关键产出 |
|------|--------|---------|
| [01-frame.md](01-frame.md) | **Phase 1 Frame**。三栏：已知 / 未知 / 假设 | 项目边界备忘 |
| [02-anchor.md](02-anchor.md) | **Phase 2 Anchor**（状态：<SELECTED 样本名 / SKIPPED 理由>） | <对标样本 / 替代 rubric> |
| [03-decompose.md](03-decompose.md) | **Phase 3 Decompose**。<N> 个原子任务，按耦合性分档 | 任务清单 + DAG + 验收信号 |

### 🔵 Phase 4：执行（04-execute/）

> **命名规则**：`supervisor-N.M.md` = Supervisor 自任串行任务；`worker-N.M.md` = 派给无全局上下文子智能体的并行任务。

| 文件 | 任务 | 类型 | 干了什么 |
|------|------|------|---------|
| [04-execute/<file>](04-execute/<file>) | <编号 任务标题> | <tight/medium/loose> | <一句话> |
| ... | ... | ... | ... |
| **[04-execute/final.md](04-execute/final.md)** ★ | 整合 | tight | **给用户的最终成品** |

### 🟣 Phase 4 之后：验收与复盘

| 文件 | 一句话 |
|------|--------|
| [review.md](review.md) | <差异分析 / rubric 自查> 报告 |

---

## 这个 Run 的关键决策与豁免

> 任何偏离默认流程的决定都在 [state.md](state.md) 留痕。这里只汇总要点。
> 如全程默认无豁免，写"无"。

| 时间 | 决策 | 理由 | 影响 |
|------|------|------|------|
| <phase> | <做了什么决策> | <为什么> | <对成品/可信度的影响> |

---

## 给后来人的提示

1. **要复盘这次跑得怎么样** → 读 [review.md](review.md) + [state.md](state.md) 的 `skipped_reason` / `risk_note` 段
2. **要把这次成果入样本库** → 用户确认成品质量后，把 [04-execute/final.md](04-execute/final.md) 拷贝到 [anchors/positive/](../../anchors/positive/)，并在 [anchors/index.md](../../anchors/index.md) 追加元数据
3. **要复用任务拆解** → [03-decompose.md](03-decompose.md) 的任务结构可作为同类项目的 project-type 预设
4. **基于本 Run 触发的系统改造** → 见 [log/tuning-history.md](../../log/tuning-history.md) 对应日期条目

---

**版本**：v1.0 · 索引建立于 <YYYY-MM-DD> · 由 Supervisor 自动生成（依据 [CLAUDE.md §1](../../CLAUDE.md) 铁律 7）
