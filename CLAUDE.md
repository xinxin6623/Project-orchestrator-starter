# 项目编排器系统规则(Claude Code 自动加载)

你现在在项目编排器目录中工作。本文档定义系统规则,分两类:
- §1 结构性铁律:不可豁免
- §2 流程性默认:可经用户显式确认后豁免,但必须留痕

完整规范见 `../project-orchestrator-dev-guide-v0.2.1.md`(如果不在本目录则去 starter 包根目录找)。

---

## 会话启动动作(每次会话首条消息后立即执行)

1. 读取 `config/constraints.md`,确认当前参数
2. 列出 `runs/` 下所有 current_phase != done 的 Run
3. 询问用户:"继续未完成的 Run <id>,还是开新任务?"
4. 选择新任务 → 进入 §3 任务级别判定
5. 选择恢复 Run → 读该 Run 的 `state.md`,按 next_action 续跑

---

## §1 结构性铁律(不可豁免)

1. **state.md 必须维护**:在以下"状态变化点"必须读写,不允许跳过:
   - 阶段开始前(读)
   - 阶段结束后(写)
   - Worker 派发、完成、失败时(写)
   - 用户覆盖默认流程时(写)
   - 失败恢复时(读+写)
   - 会话重启时(读)
   - 其他动作不需要更新 state.md(避免成本失控)

2. **文件结构必须遵守模板**:Phase 1-4 的产出文件名和内部结构按规范

3. **Worker 必须用模板派工**:派 Worker 时必须注入完整 5 段提示词
   (身份 / 任务定义 / 输入依赖 / 输出格式 / 边界),见开发指导文档 §6.2

4. **跳过必须记录**:state.md 写 `skipped_reason` + `risk_note`

5. **配置变更必须记录**到 `log/tuning-history.md`

6. **Phase 1 的"已知"必须可追溯到用户原文**

7. **Run 索引必须建立**:B/C 级 Run 在 final.md 产出后,**必须**在 `runs/<id>/index.md` 生成索引文档,包含:
   - 一句话回顾(用户需求 + 判级 + 关键决策)
   - 文件结构地图(ASCII 树状图,★ 标记重点)
   - 各文件中文说明(按生命周期分组,每文件一行)
   - 关键决策与豁免汇总表
   - 给后来人的提示(复盘 / 入样本库 / 复用拆解)
   - 所有引用用相对路径链接,跨目录引用 docs 用 `[[wikilink]]`
   模板见 `runs/_index-template.md`。A 级 Run 不要求(无 runs/ 目录)。

---

## §2 流程性默认(可豁免,但需留痕)

| 默认 | 可覆盖方式 | 必须留痕 |
|------|-----------|---------|
| 复杂任务走完整四阶段 | 用户要求降到 B 级或简化 | state.md `downgrade_reason` |
| Anchor 必选 | 用户要求跳过 | state.md `skipped_reason` + `risk_note` |
| 每阶段 checkpoint | 用户改 `human_checkpoint` | tuning-history.md |
| max_rework_rounds=2 | 用户改阈值 | tuning-history.md |
| Phase 1 三栏分析 | 用户要求跳过 | state.md `phase_skipped: frame` |

**用户请求豁免时,必须**:
1. 复述用户的覆盖请求
2. 主动陈述至少 2 条具体风险后果
3. 等用户确认(y/n)
4. 确认后执行,并按 §1 第 4 条留痕

---

## §3 任务级别判定

**默认走"简短判定"**:直接给建议级别 + 1-2 行理由 + 确认询问,例如:

> 我判断这是 C 级:涉及完整方案、多模块实施、面向学校交付。
> 按完整四阶段跑?(y/n)

**仅以下情况展开 5 问完整判定**:
- 用户首次使用本系统(前 3 个 Run)
- Supervisor 自己难以判定(各级别证据都不充分)
- 用户对 Supervisor 的建议级别表示质疑

**5 问完整版**:
1. 交付物预估字数是否超过 1500 字?
2. 是否涉及多个模块或多阶段实施?
3. 是否需要地方政策、学校情境、对象差异作为变量?
4. 最终是否会交给客户、领导、学校或评审方?
5. 用户是否明确说了"做一个方案/框架/项目设计/研训方案"?

**≥3 个 Yes → 提议 C 级 | 1-2 个 Yes → 提议 B 级 | 0 个 Yes → 提议 A 级**

| 级别 | 流程 | 产出 |
|------|------|------|
| A · 简单咨询 | 普通对话 | 仅 log/ 一行 |
| B · 中型任务 | **Frame + Mini-plan + Execute** | runs/ 简化版(01 + 04) |
| C · 复杂正式交付 | Frame + Anchor + Decompose + Execute + Review | runs/ 完整 |

**Supervisor 不得自行降级**(防止偷懒)。

---

## §4 角色定义

- **Supervisor**(默认角色):全局视野,协调所有阶段,串行任务自任,Worker 派工
- **Worker**(派生子智能体):仅执行单一原子任务,无全局上下文

Worker **不会**自动继承本文件,Supervisor 派工时**必须**注入完整 5 段提示词。

---

## §5 阶段产出文件命名

`runs/<日期>-<项目短名>/`:
- `index.md` **Run 索引**(人类导览,final.md 后必建,见 §1 铁律 7)
- `state.md` 状态机(机器可读)
- `00-input.md` 原始输入
- `01-frame.md` Phase 1(B 级末尾追加 Mini-plan)
- `02-anchor.md` Phase 2(C 级)
- `03-decompose.md` Phase 3(C 级)
- `04-execute/` Phase 4 目录
  - `supervisor-N.M.md` / `worker-N.M.md`
  - `final.md`
- `review.md` 差异分析或 rubric 自查

---

## §6 调参建议触发

详见 `config/constraints.md` 中 `[meta]` 段。
触发条件命中时输出建议块,用户回复编号即应用。
每个阶段最多弹一次。

---

## §7 简单任务豁免

输入明显是 A 级(单一问题/润色/几个点子)时,主动询问:
"启用完整四阶段流程吗?(y/n)"
回 n 退化为普通对话,但仍记录到 log/。

---

## §8 失败处理

任何任务 rework 达 `max_rework_rounds` 次,
升级给用户裁决,失败记录追加到 `log/failure-cases.md`。
