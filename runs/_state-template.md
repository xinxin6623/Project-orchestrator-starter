# Run State
# 这是 state.md 模板。新建 Run 时,Supervisor 会从这里复制并填入实际值。
# 用户通常不需要手动编辑本文件;手动编辑可能破坏恢复机制。

## 基本信息
project_name: <填入>
project_id: <YYYY-MM-DD-short-name,与目录名一致>
created_at: <ISO 8601>
last_updated_at: <ISO 8601>
task_level: <A | B | C>
project_type: <匹配的预设名,或 untyped>
config_snapshot: <Run 开始时 config/ 的 git commit hash>

## 流程位置
current_phase: frame        # frame | anchor | decompose | execute | review | done
next_action: <自然语言描述下一步要做什么>
checkpoint_pending: false

## 阶段产出
completed_files: []
next_required_file: 00-input.md

## Anchor 状态
anchor_status: missing       # missing | selected | skipped
anchor_path:                 # 选定时填路径
anchor_fitness:              # 选定时填 N/3
skipped_reason:              # 仅 skipped 时填
risk_note:                   # 仅 skipped 时填

## Phase 4 执行状态
workers_in_flight: []
workers_completed: []
workers_failed: []

## 流程偏离记录
human_overrides: []
# 格式:
# - timestamp: <ISO 8601>
#   phase: <frame | anchor | ...>
#   decision: <用户的覆盖决定>
#   reason: <理由>
