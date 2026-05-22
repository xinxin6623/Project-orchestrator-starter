# 项目编排器全局约束
# 修改后,下次 Run 生效。in-flight 的 Run 不受影响。
# 字段说明详见开发指导文档 §7.1。

## [brainstorm] Phase 1 澄清环节
mode: dynamic              # always | dynamic | off
max_questions: 3
skip_if_template_match: true
must_ask_about:
  - 年级
  - 地方政策约束
  - 验收方

## [anchor] Phase 2 对标样本
require_anchor: true       # false 时允许跳过(但 Supervisor 必须警告)
allow_supervisor_suggest: true
require_reverse_anchor: false
fitness_dimensions:           # 适配性检查的维度
  - type_match                # 项目类型匹配
  - audience_match            # 学段/对象匹配
  - structure_comparable      # 结构可比
  - time_budget_feasibility   # 时间预算可行性: Σ(每环节预估时长)+课间+缓冲 ≤ 总时长×0.85
anchor_fitness_threshold: 3   # 适配性检查需通过的维度数(/4)

## [decompose] Phase 3 任务拆解
atomic_duration_minutes: [10, 30]
max_parallel_workers: 3
serial_only_tasks:
  - 整体框架与价值主张
  - 叙事线设计
  - 跨模块衔接逻辑
  - 评价体系总框架
  - 最终整合
parallelizable_tasks:
  - 政策与文献检索
  - 案例收集与摘要
  - 数据/标准提取
  - 单一活动模块细化
require_acceptance_signal: true
require_input_dependencies: true

## [execute] Phase 4 执行与验收
review_mode: anchor_based       # anchor_based | rubric_only | hybrid
human_checkpoint: per_phase     # per_phase | per_project | off
max_rework_rounds: 2
require_design_rationale: true  # 每个环节/模块产出必附【设计意图】段:解决什么前概念或风险?依据哪条原理/课标/政策?可否删除、删了代价是什么?
                                # Worker 5 段提示词的"输出格式"段须 include 此要求
require_run_index: true         # final.md 产出后必须在 runs/<id>/index.md 生成 Run 索引(中文)
                                # 内容: 一句话回顾 + 文件结构地图 + 各文件中文说明 + 关键决策表 + 后来人提示
                                # 模板: runs/_index-template.md  规则: CLAUDE.md §1 铁律 7

## [meta] 调参建议块的触发
suggest_when_phase_overshoot_ratio: 2.0
suggest_when_user_interrupt_count: 2
suggest_when_rework_count: 2
suggest_when_user_uses_words:
  - 太细
  - 太多
  - 跳过
  - 重来
silent_mode: true               # 第 1 周建议设 true,实测后改 false
