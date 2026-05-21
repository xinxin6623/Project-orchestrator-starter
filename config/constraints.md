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
anchor_fitness_threshold: 2   # 适配性检查需通过的维度数(/3)

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
