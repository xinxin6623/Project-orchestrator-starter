---
title: 用户使用手册
tags: [orchestrator/user-facing, wiki]
aliases: [使用手册, 用户指南]
---

# 用户使用手册

> 给"用这个项目辅助自己工作的人"看。读完知道：把什么扔进去、会拿到什么、中途能管什么。
>
> 配套阅读：[[architecture-tuning]]（系统是怎么搭的）· [[deep-optimization]]（可以怎么改造）

---

## 一句话定位

把你脑子里一个模糊的"做个方案"的念头，**结构化**成可交付的成品；过程对你可见、可干预、可回滚。

不是 ChatGPT 那种聊天，是带状态机的**流水线**。

---

## 工作原理（用人话）

```
你的需求 ──▶ [Frame] ──▶ [Anchor] ──▶ [Decompose] ──▶ [Execute] ──▶ 成品
            澄清       对标参照     拆原子任务     派工执行
```

四个阶段串起来。每阶段都落一个 markdown 文件到 `runs/<日期>-<项目短名>/`，[[state-machine]] 文件记录"现在跑到哪、下一步做什么"。

详细机制看 [[architecture-tuning#四阶段流水线]]。

### 三种工作模式（A/B/C 级）

| 级别 | 适用 | 跑什么 | 大约耗时 |
|------|------|--------|----------|
| **A** | 一个问题、润色、几个点子 | 普通对话 | 几分钟 |
| **B** | 中型任务、单一模块 | Frame + Mini-plan + Execute | 半小时左右 |
| **C** | 完整方案、对外交付 | 全四阶段 | 1-3 小时 |

进会话后 Supervisor 会主动判级别问你确认。判级规则见 [[architecture-tuning#任务级别判定]]。

---

## 入口

**唯一入口**：在 `project-orchestrator/` 目录下打开 Claude Code，直接说话。

`CLAUDE.md` 会被自动加载，Supervisor 角色就位。开场会问：

> 继续未完成的 Run `<id>`，还是开新任务？

- **新任务** → 描述你要做什么 → 它判级别 → 你确认 → 开跑
- **续跑** → 它读 [[state-machine|state.md]] 接着上次的位置干

---

## 出口

每个 Run 产出一个文件夹 `runs/<日期>-<项目短名>/`：

```
00-input.md          你的原始需求（追溯用）
01-frame.md          已知 / 未知 / 假设 三栏
02-anchor.md         对标了哪份样本（C 级）
03-decompose.md      拆成了几个原子任务（C 级）
04-execute/
  worker-*.md        每个子任务的产出
  final.md           ★ 这个就是给你的成品
review.md            自查或差异分析
state.md             机器读的状态机（你别动）
```

**你只需要看 `04-execute/final.md` 和 `review.md`**，其他是过程档案，出问题时用来回溯。

---

## 交互过程：你会被问什么、什么时候问

### Phase 1（Frame）阶段
- 会问 1-3 个澄清问题（默认动态模式，没必要不会问）
- **必问项**：年级、地方政策约束、验收方（在 [[constraints#brainstorm]] 可改）
- 你也可以主动补一段长背景，省得它问

### Phase 2（Anchor）阶段
- 会给你提议一份"参照样本"，你可以同意、换一份、或要求跳过
- **跳过有代价**：成品没参照，可能套话偏多。Supervisor 会先警告并要你确认
- 库里没合适样本时会退化为 rubric 兜底，原理见 [[architecture-tuning#anchor-机制]]

### Phase 3（Decompose）阶段
- 给你看任务清单：每个 10-30 分钟一块，标好串行/并行
- 你可以砍掉、合并、调顺序

### Phase 4（Execute）阶段
- 默认 **每阶段结束都停一下让你看**（checkpoint）
- 不满意可以触发 rework（默认最多 2 轮，超了升级给你裁决）
- 想全自动跑完最后看，把 [[constraints#execute|human_checkpoint]] 改成 `per_project`

---

## 你可控的"手柄"

按从轻到重排序，前面是日常用的，后面是改架构的（改架构看 [[architecture-tuning#调参手柄]]）。

### A. 会话内随口说（最轻）

| 你说 | 效果 |
|------|------|
| "降到 B 级" / "走 A 级" | 简化流程 |
| "跳过 Anchor" | 不要对标。Supervisor 会先警告 |
| "跳过 Phase 1" | 直接进执行。慎用 |
| "重做这一段" | 触发 rework |
| "停，让我看看" | 强制 checkpoint |
| "把 max_questions 改成 1" | 改完写进 [[constraints]] 下次生效 |

所有豁免都会被记进 [[state-machine|state.md]] 和 [[tuning-history]]，事后能查为什么这样跑。

### B. 改文件（中等）

- **[[constraints|config/constraints.md]]** — 全局参数面板，所有可调项都在这。改完**下次 Run 生效**，进行中的不动
- **`config/project-types/`** — 项目类型预设。复制 `_template.md` 起一个新的
- **`anchors/index.md` + `anchors/positive/`** — 往样本库里加你过往的成功方案，越多 Phase 2 越准。加入方法见 [[architecture-tuning#anchor-机制]]

### C. 改规则（重，慎用）

- **`CLAUDE.md`** — 系统铁律。改前看 [[architecture-tuning#结构性铁律]]，有些不能动
- **`runs/<id>/state.md`** — Supervisor 维护的状态机，**手动改可能破坏断点续跑**

---

## 注意事项

1. **首次使用前**：在 `anchors/positive/` 放 2-3 份你过往的成功方案（哪怕是别人的），Phase 2 效果立刻不一样。没样本也能跑，但成品会偏"通用"
2. **跳过 Anchor 的代价**：Phase 4 验收没标尺，套话/政策搬运/回避地方差异都更难识别。例子见 `runs/2026-05-21-5th-grade-carbon-shiyou/state.md` 里 `skipped_reason` 段
3. **不要打断 Phase 4 的 Worker**：每个 Worker 是无全局上下文的子智能体，打断了它不知道前因后果，重启需要 Supervisor 重新派工
4. **跨会话续跑**：关掉 Claude Code 不丢进度，下次进来会问你恢复哪个 Run
5. **A 级任务**：不会强行走流程，会问你"启用完整四阶段吗"，回 n 就是普通对话

---

## 我搞砸了怎么办

| 症状 | 处理 |
|------|------|
| 跑到一半想推倒重来 | 直接说"重开"，旧 Run 留档不删 |
| 成品方向错了 | 让它回到 Phase 1 改澄清，后面阶段会自动重跑 |
| state.md 看着像坏了 | 别手改，跟 Supervisor 说"state 有问题"让它修 |
| 同一个错连续 rework 2 次还不行 | 系统会自动升级让你裁决，并写进 `log/failure-cases.md` |
| 想看历史调参 | `log/tuning-history.md` |
| 想看踩过的坑 | `log/failure-cases.md` |

---

## 延伸

- 系统怎么搭的、可以怎么调架构 → [[architecture-tuning]]
- 底层借鉴了哪些范式、能往哪里深挖 → [[deep-optimization]]
- 完整开发规范 → `../../project-orchestrator-dev-guide-v0.2.1.md`（如有）
