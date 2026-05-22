# 调参历史

> 每次修改 `config/constraints.md` 自动追加一行。
> 用户手动改后,Supervisor 也会在下次会话补登记。

格式:`<日期时间> | run=<run_id> | <phase> | <字段>: <旧值>→<新值>`
追加一行说明:`触发: <触发条件> | 用户接受建议 <编号>` 或 `用户主动修改: <原因>`

---

<!-- 调参记录从这里开始 -->

2026-05-22 | run=2026-05-21-5th-grade-carbon-shiyou | [execute] | require_design_rationale: (新增)→true
追加: 用户主动修改 | 原因: 用户读 final.md 后追问"校方承诺是什么/目的""三维目标依据" — Phase 4 输出只写"做什么"未写"为什么",设计意图断链。

2026-05-22 | run=2026-05-21-5th-grade-carbon-shiyou | [anchor] | fitness_dimensions: [type, audience, structure](3)→[type, audience, structure, time_budget](4)
2026-05-22 | run=2026-05-21-5th-grade-carbon-shiyou | [anchor] | anchor_fitness_threshold: 2→3
追加: 用户主动修改 | 原因: 用户读 final.md 后反馈"全天活动密度过高、一天做不完" — 适配性检查缺"时间预算可行性"维度,Phase 3 拆任务时无强制时长校验。对应 deep-optimization §4-b 适配性矩阵显式化。

2026-05-22 | run=2026-05-21-5th-grade-carbon-shiyou | [execute] | require_run_index: (新增)→true
2026-05-22 | run=2026-05-21-5th-grade-carbon-shiyou | CLAUDE.md §1 | 铁律 6→7 条 (新增铁律 7: Run 索引必须建立)
追加: 用户主动修改 | 原因: 用户为 2026-05-21-5th-grade-carbon-shiyou 手建 index.md 后,要求"把这个操作固化到以后的操作中"。配套产出: runs/_index-template.md, CLAUDE.md §1+§5 更新, user-guide/architecture-tuning 同步。
