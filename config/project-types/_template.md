# 项目类型:<填写名称,例如:校本课程设计>

## 匹配触发词
- <触发词 1,例如:课程>
- <触发词 2,例如:教学方案>
- <触发词 3>

## 命中后的默认覆盖
命中本类型时,以下字段覆盖 constraints.md 中的同名字段:

[brainstorm]
must_ask_about:
  - <本类型必问的关键变量,例如:学段>
  - <例如:课时数>

[decompose]
atomic_duration_minutes: [<下限>, <上限>]
serial_only_tasks:
  - <本类型典型的 tight 任务>

## 推荐 Anchors
- anchors/positive/<样本文件名>
- anchors/positive/<样本文件名>

## 历次教训(自动追加)

<!-- Supervisor 在每次 Run 完成后,如果本类型命中且有教训,追加在此 -->
