---
title: 内核原理与深度优化
tags: [orchestrator/kernel, wiki]
aliases: [内核, 深度优化, 原理]
---

# 内核原理与深度优化

> 给"想从原理层改造系统"的人看。
>
> 前置阅读：[[user-guide]]（用法）· [[architecture-tuning]]（表层架构与可调参数）

---

## 这个项目到底在借鉴什么

不是"AI 写方案工具"，是把几条**成熟范式**叠在一起跑：

| 借鉴的范式 | 在本项目对应什么 | 关键论断 |
|------------|------------------|----------|
| **编译器流水线** | Frame → Anchor → Decompose → Execute | 阶段间靠"中间表示"传递，不靠对话记忆 |
| **状态机 / Event Sourcing** | [state.md（模板）](../runs/_state-template.md) | 进度=状态，断点续跑靠重放状态 |
| **Map-Reduce** | Supervisor 派 Worker + 整合 | 并行执行 + 串行整合，原子任务无副作用 |
| **Case-Based Reasoning（CBR）** | [anchors/ 样本库](../anchors/index.md) | 用具体案例比用抽象规则更准 |
| **Rubric + 范例对照评估** | Phase 4 验收（anchor_based） | 双信号验收：结构 + 内容 |
| **Constraint propagation** | [constraints.md](../config/constraints.md) + [project-types/](../config/project-types/) | 约束前置注入，避免事后清洗 |
| **PDCA / 失败归因** | [log/failure-cases.md](../log/failure-cases.md) + rework 上限 | 失败是数据，不是噪声 |

下面按"项目可深度优化的部分"展开，每条都点到**原理 → 当前实现 → 深化路径**。

---

## 1. 编译器流水线：把"对话"变成"AST"

### 原理
传统对话式 AI 把上下文塞进 token，靠 attention 找信息——上下文越长越脏。**编译器范式**则在每阶段产出**结构化中间表示**（IR），下一阶段只读 IR，不读原始对话。

### 当前实现
四阶段产出（`01-frame.md` / `02-anchor.md` / `03-decompose.md`）就是 IR。但还是**自然语言 IR**，依赖下游 Worker 的语义理解，不严格。

### 深化路径
- **a. IR 形式化**：给 frame、decompose 定义结构化 schema（YAML/JSON），Worker 读结构而不是读文本。例如 `decompose.tasks[].acceptance_signal` 是字段而不是段落
- **b. IR 验证器**：每阶段产出后跑校验（如"已知必须能引回 00-input.md"是 [CLAUDE.md §1](../CLAUDE.md) 铁律 6，目前靠人）
- **c. IR 可视化**：Obsidian 里写一个 dataview 视图，把所有 Run 的 IR 提取出来横向比较

---

## 2. 状态机 / Event Sourcing：进度即状态

### 原理
**事件溯源**的核心：系统状态 = 一串事件的折叠（fold）。任何时刻可以从事件流重建当前状态。崩溃恢复 = 重放事件。

### 当前实现
`state.md` 是**快照式**，不是事件流。Supervisor 在 6 个状态变化点写整份快照（[[architecture-tuning#state.md 状态机]]）。优点：人类可读；缺点：丢中间事件，无法回放过去某个时点。

### 深化路径
- **a. 双写**：保留 `state.md` 给人看，另写 `events.jsonl` 追加事件流（一行一事件），供"时间旅行"和审计
- **b. 事件 schema**：定义事件类型（PhaseStarted / WorkerDispatched / WorkerCompleted / SkipApproved / ConfigChanged...）
- **c. 重建命令**：写个 `bin/replay-state.sh <run-id>`，从 events 重建任意时点的 state，调试断点续跑

**收益**：debug 时能精准回答"是哪一步开始跑歪的"。

---

## 3. Map-Reduce：并行的边界在哪

### 原理
Map 阶段任务必须**无共享状态**才能真并行。一旦任务间有隐式依赖，并行就是假并行（结果靠运气）。

### 当前实现
[constraints.md](../config/constraints.md) 的 `[decompose]` 段已经显式区分：
- `serial_only_tasks`: 整体框架 / 叙事线 / 跨模块衔接 / 评价体系 / 最终整合
- `parallelizable_tasks`: 政策检索 / 案例收集 / 数据提取 / 单一活动模块

但**依赖检查靠 Supervisor 经验**，没有形式化的依赖图。

### 深化路径
- **a. 依赖图显式化**：`03-decompose.md` 里每个任务声明 `depends_on: [task_id]`，画出 DAG（Mermaid），自动判断哪些可并行
- **b. 拓扑排序调度**：Supervisor 派工时按 DAG 调度，而不是按文档顺序
- **c. 并发上限自适应**：`max_parallel_workers` 当前固定 3，可以基于任务平均 token 量动态调整

---

## 4. Case-Based Reasoning：anchor 是这个系统的灵魂

### 原理
CBR 假设：**新问题最好的解法是检索一个相似的旧问题，复用它的解，再做适配**。这比从抽象原则推导更可靠，因为：
- 案例携带"隐性知识"（rubric 抓不住的）
- 适配（adaptation）比推导（derivation）认知负担低

### 当前实现
[anchors/](../anchors/) 系统是 CBR 的简化版（机制详情见 [[architecture-tuning#Anchor 机制]]）：
- 检索：靠 Supervisor 读 [anchors/index.md](../anchors/index.md) 做语义匹配
- 适配性检查：3 维度（类型 / 学段 / 结构可比）
- 跳过路径：rubric_only 兜底

**[5th-grade-carbon-shiyou](../runs/2026-05-21-5th-grade-carbon-shiyou/state.md) 那次**跳过了 Anchor，因为库内仅动物饲养类，主题完全不重叠——这是 CBR 失效的典型场景：**case base coverage gap**。

### 深化路径
- **a. 检索从手动到半自动**：给 [anchors/index.md](../anchors/index.md) 每条加 embedding（或简单的 keyword bag），Supervisor 检索时算相似度排序，而不是肉眼扫
- **b. 适配性矩阵显式化**：当前 3 维度是简化的。可以扩展到 (类型, 学段, 结构, 学科, 验收方, 资源约束) 6 维，让"为什么这个 anchor 不行"变成可解释的雷达图
- **c. Anchor 反向利用**：当前 [anchors/negative/](../anchors/negative/) 几乎没用到。改造为"反 anchor 检查"：成品产出后跑一遍，确认没有踩过相同的坑
- **d. Case base 自演化**：每个 Run 的 `final.md` 经用户确认后，自动建议入 [anchors/positive/](../anchors/positive/)。当前是手动加，所以库长不大
- **e. 覆盖率可视化**：见 [[architecture-tuning#当前架构的优化空间]] 第 6 条，把"哪些主题维度没样本"做成视图，主动指导扩库方向

---

## 5. 评估范式：rubric 与 anchor 不是替代关系

### 原理
教育评估文献里早有结论：**全 rubric** 易导致打分机械、不可比；**全 example-based** 易过拟合范本。两者**互补**才稳。

### 当前实现
`review_mode` 三档：`anchor_based` / `rubric_only` / `hybrid`，定义在 [config/constraints.md](../config/constraints.md) `[execute]` 段。但 hybrid 模式具体怎么跑没文档化。

### 深化路径
- **a. 明确 hybrid 协议**：anchor 比对结构，rubric 评内容。具体每个维度走哪条路，写进 [constraints.md](../config/constraints.md) 注释
- **b. 双信号一致性检查**：anchor 和 rubric 给出不一致结论时，触发用户裁决（而不是 Supervisor 自决）
- **c. rubric 也版本化**：[config/rubrics.md](../config/rubrics.md) 当前是一份。可以按 project-type 分裂，每类项目用专属 rubric

---

## 6. Constraint propagation：把约束前置，不要事后清洗

### 原理
形式化方法（formal methods）的核心洞见：**生成阶段就遵守约束**，比"先生成再检查再返工"成本低一个量级。

### 当前实现
- `must_ask_about` 强制澄清必问项
- `serial_only_tasks` 强制串行
- Worker 5 段提示词里的"边界"段

但还有大量约束是**事后才发现**的（如"这个方案违反了地方政策"），通过 rework 返工。

### 深化路径
- **a. project-types 携带约束**：每个类型预设里写死"必查政策清单"、"必嵌入维度"，Phase 1 自动注入到 frame
- **b. 约束传播链可追溯**：成品里每条政策/标准引用回 frame 里的"已知/未知/假设"哪一栏，[[claude-md#§1-结构性铁律|铁律 6]] 的延伸
- **c. 软约束 vs 硬约束分离**：硬约束（如"严禁人工剥壳"这类伦理铁律）违反必须 rework；软约束（如"建议引用 2 篇文献"）给提示不强制

---

## 7. PDCA：失败是数据，不是噪声

### 原理
PDCA / 持续改进的关键：**失败案例必须被结构化捕获并定期归因**，否则系统不会真正变好。

### 当前实现
- `log/failure-cases.md` 追加日志
- `max_rework_rounds=2` 触发升级
- `log/tuning-history.md` 记录调参

但都是**被动追加**，没有主动归因循环。

### 深化路径
- **a. 周期性 retro**：每 N 次 Run 或固定时段，跑一个"retro Worker"，读 failure-cases + tuning-history，输出"反模式清单" → `log/anti-patterns.md`
- **b. anti-pattern 反哺 Supervisor**：派工前先扫一遍 anti-patterns，发现匹配的踩坑模式提前警告
- **c. 调参建议从触发到决策的闭环**：当前 [[constraints#meta|meta 触发]] 弹建议块，但用户决策不闭环。改造：用户接受建议后自动改 constraints + 写 tuning-history + 影响后续 Run

---

## 8. 一个被忽视的内核：Supervisor 自身的元认知

### 原理
专家与新手的核心差异不是知识量，是**元认知**：知道自己什么时候会判错。本系统目前 Supervisor 没有显式的"自我怀疑"机制。

### 深化路径
- **a. 不确定性标注**：Supervisor 在判级别、选 anchor、拆任务时，主动报置信度（高/中/低）。低置信度自动触发额外用户确认
- **b. 反事实检查**：关键决策（跳过 Anchor、降级 B、跳过 Phase 1）必须先生成"如果不这样做会怎样"的反事实文本，再让用户裁决
- **c. 一致性自检**：Phase 4 整合前，Supervisor 自问"这份 final 和 frame 里的'已知/未知/假设'对得上吗"，对不上回到 Phase 1

5th-grade 那次跳过 Anchor 时 Supervisor 做得对（复述+风险+确认），但这是靠 `CLAUDE.md §2` 硬编码的，**不是 Supervisor 自发的元认知**。把它泛化成"任何高代价决策都走这套协议"，能解锁更多稳定性。

---

## 优化优先级（如果只能做 3 件）

我的判断（你可以推翻）：

1. **[[#4-case-based-reasoning-anchor-是这个系统的灵魂|Anchor 检索半自动化 + 覆盖率视图]]** — 当前最大的瓶颈是样本库小且无指引扩库方向
2. **[[#2-状态机-event-sourcing-进度即状态|事件流双写]]** — 调试和审计能力质变，成本不高
3. **[[#6-constraint-propagation-把约束前置-不要事后清洗|project-types 携带硬约束]]** — 把已经存在但闲置的 project-types 系统激活，立刻减少 Phase 1 澄清开销

---

## 延伸

- 怎么用 → [[user-guide]]
- 表层架构与可调参数 → [[architecture-tuning]]
- 真实失败案例素材 → `../log/failure-cases.md` 与 `../runs/2026-05-21-5th-grade-carbon-shiyou/state.md`
