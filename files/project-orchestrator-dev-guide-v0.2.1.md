# 项目编排器(Project Orchestrator)开发指导文档

> **版本**:v0.2.1
> **适用场景**:教育咨询类复杂方案设计(课程研发、低碳校园、教研项目、研训方案等)
> **运行平台**:Claude Code 终端智能体
> **v0.2 核心改动**:不增加任何新功能,只补 v0.1 的运行闭环——状态机、Anchor 库、Worker 模板、跳过记录、恢复语义、Subagent 注入机制
> **v0.2.1 微调**:实测前修 3 处会污染实测信号的小缺陷(state.md 更新频率、B 级 Mini-plan、任务级别简短输出),不动主体结构

---

## 0. 如何阅读本文档

本文档同时面向两类读者:

- **开发者(你或协作 AI)**:按 §3 → §9 → §11 → §14 顺序读,可直接照搬模板搭出可运行版本
- **使用者(你日常用时)**:读 §1 + §4 + §8 + §13 即可,其他章节查阅式使用

**关键约定**:
- "Supervisor" = 主智能体(Claude Code 主会话)
- "Worker" = 派生的 Subagent
- "用户" = 你本人
- "Run" = 一次具体项目的执行实例,对应 `runs/` 下一个子目录

**v0.2 相对 v0.1 的主要变化**:见 §15 变更说明。如果你已熟悉 v0.1,先读 §15 再回头查阅。

---

## 1. 项目定位

### 1.1 一句话定义

> 一套**强制 Claude Code 在动手做复杂教育方案前先经历"框定—锚定—拆解—执行"四阶段**的本地协议系统,所有规则用 markdown 存储,**用户在使用中可随时调整规则本身**。

### 1.2 解决的真问题

教育咨询场景下,直接让 LLM 出方案普遍存在三种"软性失败":

| 失败模式 | 表现 | 根因 |
|---------|------|------|
| **套话堆砌** | "立德树人""五育融合"等口号铺满文档 | 高频表达过拟合 |
| **政策原文搬运** | 大段引用文件原文当方案 | 用"权威感"替代"原创性" |
| **回避地方差异** | 不区分北京/西部、城市/县域,通用废话 | 缺乏边界约束走最大公约数 |

本系统通过**结构强制**对抗这三种偷懒倾向,而不是靠提示词哄。

### 1.3 不解决什么(边界声明)

- ❌ 不是通用项目管理工具
- ❌ 不替代专业判断
- ❌ 不处理代码类任务
- ❌ 不做实时协作

### 1.4 与"任务级别"的关系

并非所有任务都需要走完整四阶段。详见 §4 任务级别判定。

---

## 2. 设计哲学:四个核心约束

### 2.1 约束一:拒绝"LLM 自评闭环"

**解法:对标样本法(Anchor-based Review)**

强制选定物理参照样本。验收时要求 Supervisor 回答**"与对标样本相比差异是什么、为何差异"**,差异由你裁决,而不是让 Supervisor 自己说"是否达标"。

### 2.2 约束二:拒绝"无差别原子化拆解"

**解法:耦合性扫描前置**

Phase 3 拆解前,先分类:

| 类型 | 处理方式 |
|------|---------|
| **loose(松耦合)** | 派 Subagents 并发 |
| **tight(强耦合)** | Supervisor 自任,禁止派 |
| **medium(半耦合)** | 串行执行但共享上下文 |

### 2.3 约束三:拒绝"仪式化澄清"

**解法:三栏分析(已知/假设/未知)+ 动态触发**

Phase 1 先做三栏分析,只对"未知"栏追问,默认上限 3 个。

### 2.4 约束四(v0.2 新增):拒绝"协议层完美、运行层崩溃"

**解法:每个 Run 维护机器可读的 `state.md` 状态机**

详见 §9。这一条约束的存在是为了承认一个事实:**Claude Code 会话会被中断、上下文会被压缩、Subagents 会失败**。如果系统设计假设"一气呵成跑完",就会在真实使用第二天崩掉。

---

## 3. 系统架构

### 3.1 运行环境要求

最小依赖:

- **Claude Code** 终端版
- **Git**
- **任意文本编辑器**

显式不依赖:Docker、Python、向量库、任何 Web 服务。

### 3.2 目录结构(v0.2 更新)

```
project-orchestrator/
├── CLAUDE.md                      # 系统铁律(自动加载)
├── README.md                      # 给你看的快速上手
│
├── config/
│   ├── constraints.md             # 全局可调参数
│   ├── rubrics.md                 # 评价模板(备用,仅 review_mode != anchor_based 时使用)
│   └── project-types/
│       ├── _template.md
│       ├── curriculum-design.md
│       ├── carbon-school.md
│       └── teacher-training.md
│
├── anchors/                       # ★ v0.2 新增:对标样本库
│   ├── index.md                   # 样本索引(适用场景、优缺点、可参照维度)
│   ├── positive/                  # 正向样本
│   │   ├── curriculum-good-01.md
│   │   └── ...
│   └── negative/                  # 反向样本(警示用)
│       └── generic-policy-bad-01.md
│
├── runs/
│   └── 2026-05-21-9th-grade-carbon-labor/
│       ├── state.md               # ★ v0.2 新增:机器可读状态机
│       ├── 00-input.md
│       ├── 01-frame.md
│       ├── 02-anchor.md
│       ├── 03-decompose.md
│       ├── 04-execute/
│       │   ├── worker-2.1.md
│       │   ├── worker-2.2.md
│       │   └── final.md
│       └── review.md
│
└── log/
    ├── tuning-history.md          # 调参变更日志
    └── failure-cases.md           # 失败案例归档
```

### 3.3 数据流

```
用户输入需求
    ↓
Claude Code 加载 CLAUDE.md(系统铁律)
    ↓
CLAUDE.md 显式指示读取 config/constraints.md
    ↓
Supervisor 做任务级别判定(§4)
    ├─ A 级简单任务 → 普通对话,仅写入 log/
    ├─ B 级中型任务 → Frame + Execute(无 Anchor、无 Decompose)
    └─ C 级复杂任务 → 完整四阶段
    ↓
创建 runs/<日期-名称>/,写入 state.md 初始状态
    ↓
按 state.md 中 current_phase 推进
    ↓
每阶段结束:更新 state.md → 触发条件命中则弹调参建议
    ↓
最终方案落在 runs/<日期-名称>/04-execute/final.md
```

---

## 4. 任务级别与入口判定(v0.2 新增)

### 4.1 三级定义

| 级别 | 适用 | 流程 | 产出 |
|------|------|------|------|
| **A · 简单咨询** | 单一问题、润色、几个点子 | 普通对话,无 Run 目录 | 仅 log/ 一行 |
| **B · 中型任务** | 单模块设计、初稿、内部讨论稿 | Frame + Mini-plan + Execute | runs/ 简化版(01 + 04) |
| **C · 复杂正式交付** | 完整方案、给客户/学校/评审 | Frame + Anchor + Decompose + Execute + Review | runs/ 完整 |

### 4.2 判定流程(谁来决定)

**Supervisor 提议 → 用户确认**,不让 Supervisor 自行升降级。

**默认走"简短判定"**(v0.2.1):Supervisor 直接给建议级别 + 1-2 行理由 + 确认询问,例如:

> 我判断这是 C 级:涉及完整方案、多模块实施、面向学校交付。
> 按完整四阶段跑?(y/n)

**仅以下情况展开 5 问完整判定**:
- 用户首次使用本系统(前 3 个 Run)
- Supervisor 自己难以判定(各级别证据都不充分)
- 用户对 Supervisor 的建议级别表示质疑

**5 问完整版**(展开时使用):

1. 交付物预估字数是否超过 1500 字?
2. 是否涉及多个模块或多阶段实施?
3. 是否需要地方政策、学校情境、对象差异作为变量?
4. 最终是否会交给客户、领导、学校或评审方?
5. 用户是否明确说了"做一个方案/框架/项目设计/研训方案"?

**≥3 个 Yes → 提议 C 级**
**1-2 个 Yes → 提议 B 级**
**0 个 Yes → 提议 A 级**

用户可强制覆盖,覆盖原因写入 state.md(C 级)或 log(A/B 级)。

### 4.3 升级与降级

Run 进行中允许:

- **A → B/C 升级**:发现任务比预想复杂,Supervisor 主动提议升级,用户确认后创建 runs/ 目录,把已有对话作为 00-input.md
- **C → B 降级**:用户要求"别那么重",但必须明示降级理由,state.md 记录 `downgrade_reason`

不允许:Supervisor 单方面降级(防止 LLM 偷懒)。

---

## 5. 四阶段工作流详解

### 5.1 Phase 1 · Frame(框定)

**目标**:把模糊需求转成一份"项目边界备忘"。

**输出**:`runs/<项目>/01-frame.md`:

```markdown
# 项目边界备忘

## 已知(从输入直接提取)
- [事实 1,引用原文位置]

## 假设(Supervisor 自行补充,需用户确认)
- [假设 1] —— 依据:[为何这么假设]

## 高风险假设(v0.2 新增)
如果判断错了会导致方案塌房的关键假设,与普通假设分开:
- [假设] —— 错了的话:[后果描述]
  例:假设学校已有 PBL 经验。错了的话,活动复杂度需要全部下调,
      框架需要从"探究式"改为"指导式"。

## 未知(需要追问)
仅当 brainstorm.mode != off 时输出:
- [关键变量]:[追问的具体问题]

## 不可变量
- 交付物形式:
- 截止时间:
- 验收方:
```

**Supervisor 行为协议**:
1. 严禁先输出方案任何片段
2. "已知"必须可追溯到用户原文
3. "假设"与"高风险假设"必须显式分开
4. 追问问题不超过 `max_questions`

**修正协议(v0.2 新增)**:用户对 Phase 1 输出提出修正时:
- 修改"假设"或"高风险假设":Supervisor 直接 str_replace 更新 01-frame.md,无需重跑
- 修改"已知"(说明 Supervisor 误读):重跑 Phase 1
- 增补"未知"答案:合并进"已知",删去对应"未知"条目

### 5.1.5 B 级任务的 Mini-plan(v0.2.1 新增)

B 级任务不走完整 Decompose,但 **Frame 后直接 Execute 仍可能让模型乱写**。解法是在 01-frame.md 末尾追加一段轻量执行计划(不独立成文件)。

**Mini-plan 模板**(挂在 01-frame.md 末尾):

```markdown
## 轻量执行计划(B 级专用)

### 关键产出顺序
- 先 [...]
- 再 [...]
- 最后 [...]

### 必要内嵌依赖
- 执行时必须先确定的 2-3 个输入

### 可操作性自检点
- 写完后必须自查的 2-3 项(例:具体数字是否给出?对象是否明确?)
```

**示例**(一份"给某校教师培训的破冰活动设计"):

```markdown
## 轻量执行计划(B 级专用)

### 关键产出顺序
- 先定活动结构(3 段式)
- 再补每段的具体步骤、时长、所需道具
- 最后做可操作性检查

### 必要内嵌依赖
- 教师人数(影响分组)
- 现场是否有投影(影响活动 2 设计)

### 可操作性自检点
- 每个步骤是否给出具体时长?
- 是否标注主持人话术?
- 是否考虑了内向教师的参与方式?
```

**协议要点**:
- Mini-plan **不超过 15 行**,超出说明该升 C 级
- Supervisor 必须按 Mini-plan 顺序写 Execute,不得乱序
- Execute 完成后,**必须按自检点逐项核对**,核对结果写在 04-execute/final.md 顶部

### 5.2 Phase 2 · Anchor(锚定)

**目标**:选定本项目的对标样本。

**v0.2 改动**:优先从 `anchors/` 库查找,而非每次临时找。

**Supervisor 行为协议**:
1. 先读 `anchors/index.md`,按项目类型筛出候选
2. 若库内有匹配:展示 1-3 个候选,用户选定
3. 若库内无匹配:询问用户是否提供外部样本,或允许 Supervisor 从公开源建议
4. **对每个候选执行 Anchor 适配性检查**(v0.2 新增):
   - 项目结构维度是否可比?
   - 受众规模是否相当?
   - 主题方向是否同类?
   - 三项中至少 2 项可比才允许选用,否则标记"参考价值有限"
5. 选用后写入 02-anchor.md,并把样本路径写入 state.md 的 `anchor_path`

**输出**:`runs/<项目>/02-anchor.md`:

```markdown
# 对标样本

## 选定样本
- **样本名**:[北京某校 2024 年劳动课程实施方案]
- **来源**:anchors/positive/curriculum-good-01.md
- **适配性检查**:
  - [✓] 结构维度可比
  - [✓] 受众规模相当(均为单校单年级)
  - [✗] 主题方向不完全同类(对标为综合实践,本项目为低碳主题)
- **参考价值评级**:中(2/3 可比)

## 反向样本(可选)
- **样本名**:[某区教研室通用模板]
- **不像理由**:过于宏观、缺少操作步骤

## 关键参照维度(本次验收时使用)
- [ ] 结构组织方式
- [ ] 颗粒度深度
- [ ] 评价机制设计
- [ ] 地方政策融入方式
```

**Anchor 跳过路径(v0.2 新增)**:
若 `require_anchor: false` 且用户明确要求跳过:
- state.md 中 `anchor_status: skipped`,`skipped_reason` 必填
- state.md 中 `risk_note` 必填(Supervisor 主动陈述风险)
- Phase 4 不再做"对标差异分析",改为执行 `rubrics.md` 检查
- review.md 改用替代模板(见 §5.4)

### 5.3 Phase 3 · Decompose(拆解)

**目标**:产出带耦合性标注和输入依赖的任务清单。

**输出**:`runs/<项目>/03-decompose.md`:

```markdown
# 任务拆解清单

## 耦合性扫描结论
本项目共识别 N 个任务,其中:
- tight(串行,Supervisor 自任):X 个
- loose(并行):Y 个
- medium(串行但共享上下文):Z 个

## 任务表

| # | 任务 | 类型 | 时长 | 输入依赖 | 验收信号 | 执行者 |
|---|------|------|------|---------|---------|--------|
| 1.1 | 制定方案整体框架与价值主张 | tight | 20min | - | 与对标样本结构对应可解释 | Supervisor |
| 2.1 | 收集北京低碳学校 EMS 监测标准 | loose | 15min | - | 列出 3+ 项硬性指标 | Worker |
| 2.2 | 检索 9 年级劳动课程现有案例 | loose | 15min | - | 至少 2 个可引用案例 | Worker |
| 3.1 | 写模块 1:碳足迹测量活动 | medium | 25min | 1.1, 2.1, 2.2 | 含具体步骤/材料/安全提示 | Supervisor |
| 3.2 | 写模块 2:校园低碳行为改造 | medium | 25min | 1.1, 2.1, 3.1 | 同上 | Supervisor |
| 9.1 | 整合各模块为完整方案 | tight | 30min | 1.1, 3.1, 3.2, ... | 通读无断裂感、价值线一致 | Supervisor |

## 派工计划
- 第一批并发:2.1, 2.2
- 第二批串行:1.1 → 3.1 → 3.2 → ... → 9.1
```

**Supervisor 行为协议**:
1. 严禁产出"宏观大纲"作为任务
2. 每个任务必须有可观察的验收信号
3. **每个任务必须填"输入依赖"列**(v0.2 新增):指明该任务执行前需要哪些前序产出
4. **"整合"必须作为独立 tight 任务列出**(v0.2 新增):不能假装整合是免费的
5. 时长按 `atomic_duration_minutes` 范围
6. 耦合性扫描结果必须显式陈述理由

**升级到用户的情况**:任务难以归类时,Supervisor 必须问你,而不是自行决定。

### 5.4 Phase 4 · Execute & Cross-check(执行与对标)

**执行流程**:

```
1. 读 state.md,确认上次执行到哪一步
   ↓
2. 派发第一批 loose 任务(数量 ≤ max_parallel_workers)
   - 每派一个 Worker,在 state.md 的 workers_in_flight 写入
   - Worker 提示词必须注入(见 §6)
   ↓
3. Worker 完成 → 写 04-execute/worker-N.N.md
   - state.md 的 workers_in_flight 移到 workers_completed
   ↓
4. 全部 loose 完成后,串行执行 tight + medium
   - medium 任务执行时必须先读完所有输入依赖文件
   ↓
5. 执行"整合"任务(独立 tight 任务,见 Phase 3)
   - 产出 04-execute/final.md
   ↓
6. 对标差异分析(若 anchor_status: selected):
   逐项填写 02-anchor.md 中的参照维度差异
   产出 review.md
   ↓
   或:rubrics 检查(若 anchor_status: skipped):
   按 rubrics.md 逐项核验
   产出 review.md(替代模板)
   ↓
7. state.md 标记 current_phase: done
   等待用户裁决
```

**review.md 标准模板(有 anchor)**:

```markdown
# 与对标样本的差异分析

## 维度 1:结构组织方式
- **对标样本**:采用"目标-内容-实施-评价"四段式
- **本方案**:采用"问题情境-探究活动-反思评价"三段式
- **差异判断**:更适合 9 年级 PBL,但实施细节可能不足
- **风险**:建议补充实施部分

[各维度依次填写]

## 用户需要裁决的差异
- [ ] 维度 1 的三段式是否保留?
```

**review.md 替代模板(无 anchor)**:

```markdown
# Rubric 自查报告

⚠️ 本项目缺少 Anchor,验收可信度下降。

## 必含要素核验
- [✓] 明确目标对象
- [✗] 缺少安全风险提示 ← 需补充
- [✓] 评价方式说明
...

## 减分信号扫描
- [✓] 未出现政策原文大段搬运
- [✗] 套话占比约 25%,超过阈值 ← 需修改
...

## 用户需要裁决
- [ ] 是否接受套话占比超标?
```

**Supervisor 行为协议**:
1. 差异分析必须基于 02-anchor.md 的真实内容
2. 不允许"基本一致""略有差异"等模糊判断
3. 拿不准的差异列为"用户需要裁决"
4. **Workers 失败处理**(v0.2 新增):
   - Worker 产出不达验收信号 → state.md 中 `workers_failed[id].rework_count += 1`
   - 重做次数达 `max_rework_rounds` → 升级给用户,记录到 failure-cases.md
5. **Checkpoint 行为**(v0.2 明确):
   - `human_checkpoint: per_phase` → 每阶段结束 state.md 标 `checkpoint_pending: true`,等待用户继续指令
   - `per_project` → 仅在 Phase 4 完成后停
   - `off` → 不停

---

## 6. Worker 协议与输出模板(v0.2 新增)

### 6.1 为什么需要这一节

Claude Code 的 Subagent(通过 Task 工具派生)**不会自动继承 CLAUDE.md**——它们获得的是 Supervisor 在调用时显式传入的提示词。这意味着:

- Worker 不知道项目规则
- Worker 不知道输出格式要求
- Worker 不知道任务的上下文边界

如果 Supervisor 派工时只丢一句"去搜集 EMS 标准",Worker 会用任意格式输出,Supervisor 整合时要做大量重写——这就是 v0.1 的隐藏成本。

### 6.2 Supervisor 派工时必须注入的内容

Supervisor 派 Worker 时,提示词必须包含以下 5 段(顺序固定):

```markdown
# [段 1] 任务身份
你是项目 <project_name> 的 Worker,任务编号 <task_id>。
你不需要也不应该思考全局,只专注完成本任务。

# [段 2] 完整任务定义
[复制 03-decompose.md 中该任务的整行,包括验收信号]

# [段 3] 输入依赖(若有)
以下是你需要先读的前序产出:
- [文件路径 1]:[一句话说明该文件包含什么]
- ...

# [段 4] 输出格式(强制使用模板)
你必须输出一份 markdown 文件,文件名 worker-<task_id>.md,
路径 runs/<project>/04-execute/。
文件结构必须严格按以下模板:

[完整粘贴 §6.3 的 Worker 输出模板]

# [段 5] 边界
- 严禁尝试完成不属于你的任务
- 严禁修改任何其他文件
- 不确定的事情写进"不确定项",不要瞎猜
- 完成后只输出文件路径,不要总结
```

### 6.3 Worker 输出模板(强制)

```markdown
# Worker 任务产出

## 元信息
- task_id: 2.1
- task_title: 收集北京低碳学校 EMS 监测标准
- assigned_at: 2026-05-21T10:00:00+08:00
- completed_at: 2026-05-21T10:14:00+08:00

## 原始任务
[Supervisor 复制粘贴的任务定义]

## 输入依赖
[列出实际读取的依赖文件;若无,写"无"]

## 关键结论(给 Supervisor 整合用)
- 结论 1
- 结论 2

## 可直接进入方案的素材
- [带格式/位置标注,Supervisor 复制即可用的成块内容]

## 不确定项 / 需要 Supervisor 裁决
- [明确写出哪些点 Worker 拿不准]

## 来源与引用
- [URL/文件路径/页码]

## 与主方案的衔接建议
- [Worker 视角的整合建议,Supervisor 可不接受]
```

### 6.4 Supervisor 整合行为

Worker 提交后,Supervisor 必须:
1. 读 worker-N.N.md
2. 核对验收信号——不达标则启动 rework(见 §5.4)
3. 把"不确定项"逐条处理:能裁决的直接裁决,不能的列入 review.md
4. 整合时严禁直接复制 Worker 的散文段落,**应当复制"可直接进入方案的素材"段**——这是为什么模板要把素材单独成节

---

## 7. 配置系统详解

### 7.1 `config/constraints.md` 完整规范

```markdown
# 项目编排器全局约束
# 修改后,下次会话生效。in-flight 的 Run 不受影响(见 §7.4)。

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
require_input_dependencies: true   # v0.2 新增

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
silent_mode: false
```

### 7.2 `config/project-types/` 类型预设

(模板与 v0.1 相同,见附录 G)

### 7.3 `config/rubrics.md` 评价模板

**用途明确化(v0.2)**:仅在以下两种情况使用:
- `review_mode: rubric_only`(完全不用 anchor)
- `review_mode: hybrid`(anchor + rubric 双轨)
- anchor 被跳过的 Run(替代 anchor 的兜底)

### 7.4 配置变更的"in-flight"语义(v0.2 新增)

**问题**:Run 进行中如果用户改了 constraints.md,影响哪些?

**规则**:
- 改 `[meta]` 段(调参触发):**立即生效**,本会话剩余阶段适用新值
- 改 `[brainstorm]` `[anchor]` `[decompose]`:**仅影响下个 Run**,当前 Run 保持开始时的配置
- 改 `[execute]`:**仅影响下次 Phase 4 调用**,正在执行的 Workers 不受影响

每次 Run 开始时,Supervisor 在 state.md 中快照当前配置(`config_snapshot: <hash>`),作为本次执行的固定参数。

---

## 8. 调参回路

### 8.1 触发条件

Supervisor 仅在以下信号命中时弹建议:

| 信号 | 字段 | 默认值 |
|------|------|--------|
| 阶段耗时超预期 N 倍 | suggest_when_phase_overshoot_ratio | 2.0 |
| 用户打断 N 次 | suggest_when_user_interrupt_count | 2 |
| 任务被打回 N 次 | suggest_when_rework_count | 2 |
| 用户用了特定词 | suggest_when_user_uses_words | 见上 |

每个阶段最多弹一次。

### 8.2 建议块格式

```
─── 阶段完成:Phase 3 · Decompose ───
本次产出:18 个原子任务(5 tight + 11 loose + 2 medium)
执行耗时:6 分 12 秒(预估 3 分,超 2.07 倍)

🔧 调参建议(回复编号以应用,回车跳过):

[1] 任务粒度可能过细(超时主因)
    建议:atomic_duration_minutes: [10,30] → [15,40]
    影响:下次拆解任务数减少约 30%

[2] "单一活动模块细化"可能不适合本类项目并行
    建议:从 parallelizable_tasks 移除该项

[3] 保存本次配置为新预设
    建议:创建 project-types/<推断的类型名>.md

[0] 跳过,保持现状
```

### 8.3 应用动作

用户回 `1`,Supervisor 执行:
1. `str_replace` 修改 `config/constraints.md`
2. 追加到 `log/tuning-history.md`:
   ```
   2026-05-21 09:15 | run=<id> | Phase 3 | atomic_duration_minutes: [10,30]→[15,40]
   触发:阶段耗时 2.07 倍超预期 | 用户接受建议 1
   ```
3. 简短回复:"已更新 constraints.md(下次 Run 生效,本 Run 保持原配置)"

### 8.4 静默条件

- `silent_mode: true`
- 项目首次运行(无历史数据可比)
- 用户上一条消息明示"别打断我"

### 8.5 反向回路:用户主动调

```
用户:把 max_parallel_workers 改成 5
Supervisor:已更新 config/constraints.md(下次 Run 生效)
```

或用户直接 vim 改 + git commit,Supervisor 下次启动读最新值。

---

## 9. 状态管理与恢复(v0.2 新增)

### 9.1 为什么需要 state.md

Claude Code 会话可能在以下情况中断:
- 上下文超限被压缩
- 用户主动关闭终端
- 网络故障
- Subagent 超时

如果没有持久化的状态,恢复时 Supervisor 不知道:
- 跑到哪个阶段了
- 哪些 Worker 跑完了哪些没跑完
- 用户是否做了某些跳过决定
- 适用的配置快照是什么

### 9.2 state.md 规范

每个 Run 目录下必须有一份 state.md,**机器可读**(YAML-like markdown)。

**读写时机(v0.2.1 调整)**:Supervisor 仅在以下"状态变化点"读写 state.md,**不是每个动作**:

| 时机 | 动作 | 备注 |
|------|------|------|
| 阶段开始前 | 读 | 确认 current_phase 和 next_action |
| 阶段结束后 | 写 | 更新 completed_files、current_phase |
| Worker 派发时 | 写 | 追加 workers_in_flight |
| Worker 完成时 | 写 | 移到 workers_completed |
| Worker 失败时 | 写 | 移到 workers_failed,更新 rework_count |
| 用户覆盖流程时 | 写 | 追加 human_overrides |
| 失败恢复时 | 读+写 | 决定恢复策略并记录 |
| 会话重启时 | 读 | 确定从哪个 Run 哪个阶段续跑 |

**不需要更新 state.md 的动作**:读 anchor 文件、读历史 Run 产出、Supervisor 自任的 tight/medium 任务内部步骤(整个任务完成时算一个状态变化点)、调用 web_search 等。

这条规则避免 state.md 维护本身消耗大量 token——v0.2 原版"每个动作前读、每个动作后写"在实测中会失控。

```markdown
# Run State

## 基本信息
project_name: 9 年级低碳学校劳动实践方案
project_id: 2026-05-21-9th-grade-carbon-labor
created_at: 2026-05-21T09:00:00+08:00
last_updated_at: 2026-05-21T10:14:00+08:00
task_level: C
project_type: carbon-school
config_snapshot: 配置文件在 Run 开始时的 git commit hash

## 流程位置
current_phase: execute        # frame | anchor | decompose | execute | review | done
next_action: 等待 Worker 2.2 完成,然后启动 Worker 3.1
checkpoint_pending: false

## 阶段产出
completed_files:
  - 00-input.md
  - 01-frame.md
  - 02-anchor.md
  - 03-decompose.md
next_required_file: 04-execute/worker-2.2.md

## Anchor 状态
anchor_status: selected       # missing | selected | skipped
anchor_path: anchors/positive/curriculum-good-01.md
anchor_fitness: 2/3
skipped_reason:               # 仅 skipped 时填
risk_note:                    # 仅 skipped 时填

## Phase 4 执行状态
workers_in_flight:
  - id: 2.2
    title: 检索 9 年级劳动课程现有案例
    started_at: 2026-05-21T10:10:00+08:00
    status: running
workers_completed:
  - id: 2.1
    title: 收集北京低碳学校 EMS 监测标准
    completed_at: 2026-05-21T10:14:00+08:00
    rework_count: 0
workers_failed: []

## 流程偏离记录
human_overrides:
  - timestamp: 2026-05-21T09:30:00+08:00
    phase: anchor
    decision: 选用 anchors/positive/curriculum-good-01.md 而非用户提议的样本 X
    reason: 适配性检查发现 X 仅 1/3 维度可比
```

### 9.3 恢复语义

会话重启时,Supervisor 第一动作:

1. 读 CLAUDE.md
2. 读 config/constraints.md(注意:可能与 config_snapshot 不同)
3. 列出 runs/ 下所有 current_phase != done 的 Run,询问用户继续哪个
4. 读选定 Run 的 state.md
5. 根据 `current_phase` 和 `next_action` 直接续跑——不重读已完成的阶段产物作为"重新思考",只在需要引用时读

### 9.4 失败恢复

`max_rework_rounds` 命中后:
1. state.md 中标记 `workers_failed[id]`
2. 追加到 `log/failure-cases.md`:
   ```
   2026-05-21 | run=<id> | task=2.3 | rework 达 2 次仍不达标
   验收信号:列出 2+ 个可引用案例
   实际产出:仅找到 1 个案例,且来源可信度低
   决策:升级用户裁决
   ```
3. state.md 中 `current_phase` 不变,但 `next_action` 改为 "等待用户裁决 task 2.3 失败"
4. 等待用户:接受不达标产出 / 降低验收信号 / 跳过该任务 / 整体放弃 Run

---

## 10. Anchors 库(v0.2 新增)

### 10.1 目的

避免 Phase 2 每次临时找样本——把样本当作可积累的资产管理。

### 10.2 anchors/index.md 规范

```markdown
# Anchor 样本索引

## positive/

### curriculum-good-01.md
- 路径: anchors/positive/curriculum-good-01.md
- 类型: 课程设计 / 校本课程
- 适用项目: 9-12 年级 / 学科融合方向
- 优点: 评价机制具体,有可观察行为指标
- 缺点: 课时安排过密,小规模学校难复制
- 可参照维度: 结构 / 评价 / 跨学科衔接
- 来源: [文件 / URL / 用户提供]
- 加入时间: 2026-04-10

### carbon-school-good-01.md
...

## negative/

### generic-policy-bad-01.md
- 路径: anchors/negative/generic-policy-bad-01.md
- 不像的理由: 70% 内容为政策原文搬运;缺少操作步骤
- 警示价值: 用作"绝对不能像这样"的反向锚点
- 加入时间: 2026-04-12
```

### 10.3 维护节奏

- 每完成一个 C 级 Run,Supervisor 主动询问:"本次产出是否值得加入 anchors/positive/?"
- 用户接受 → 复制 final.md 到 anchors/positive/,在 index.md 追加条目
- 用户拒绝 → 不操作
- 定期(每季度)review index.md,删除明显过时或低质量条目

### 10.4 外部样本处理

用户提供 PDF/网页/图片样本:
- Supervisor 先转为 markdown 摘要(若过长,分块摘要)
- 存入 anchors/positive/<日期>-<简短描述>.md
- 摘要顶部必须标注"原始格式: PDF / 来源: ..."

---

## 11. 系统规则:结构性铁律 vs 流程性默认(v0.2 重大重构)

v0.1 的"Iron Laws 不可违抗"过于刚性。v0.2 拆成两类:

### 11.1 结构性铁律(Structural · 真正不可违抗)

这些是关于**产出文件如何组织、如何留痕**的规则,违反会破坏系统的可追溯性和可恢复性。即使用户要求也不得豁免。

1. **state.md 必须维护**:在状态变化点必须读写(具体定义见 §9.2),不允许跳过
2. **文件结构必须遵守**:Phase 1-4 的产出文件名和内部结构按模板
3. **Worker 必须用模板**:派 Worker 时必须注入 §6.2 的 5 段提示词
4. **跳过必须记录**:任何阶段、检查、约束被跳过,state.md 必须有 `skipped_reason` + `risk_note`
5. **配置变更必须记录**:改 constraints.md 必须追加到 tuning-history.md
6. **已知必须可追溯**:Phase 1 的"已知"必须能回溯到用户原文

### 11.2 流程性默认(Procedural · 可显式豁免)

这些是关于**走不走某流程、走多严**的规则,默认强制但用户可显式覆盖。

| 默认 | 可覆盖方式 | 必须留痕 |
|------|-----------|---------|
| 复杂任务走完整四阶段 | 用户要求降到 B 级或简化 | state.md `downgrade_reason` |
| Anchor 必选 | 用户要求 `anchor_status: skipped` | state.md `skipped_reason` + `risk_note` |
| 每阶段 checkpoint | 用户改 `human_checkpoint: per_project` 或 `off` | tuning-history.md |
| max_rework_rounds=2 | 用户改阈值 | tuning-history.md |
| Phase 1 三栏分析 | 用户说"跳过 Phase 1 用我之前的边界" | state.md `phase_skipped: frame` + 引用前序边界文件 |

### 11.3 覆盖请求的处理流程

用户要求覆盖默认时,Supervisor 必须:
1. 复述用户的覆盖请求
2. **主动陈述风险**(至少 2 条具体后果)
3. 询问用户确认
4. 确认后执行,并按 §11.1 第 4 条留痕

例:

```
用户:这次别找对标了,直接写初稿
Supervisor:你要求跳过 Phase 2 Anchor。风险提示:
  1. Phase 4 的验收将退化为 rubric 自查,可信度下降
  2. 本次产出无法用作 anchors/ 库的对照基准
  3. 如果后续要补对标,需要重跑 Phase 4
  确认跳过吗?(y/n)
用户:y
Supervisor:[在 state.md 写入 anchor_status: skipped,
            skipped_reason: 用户要求快速生成初稿,
            risk_note: <上述 3 条>]
          已记录。继续 Phase 3。
```

---

## 12. 实施路径(三周滚动)

### 第 1 周:观察期

- [ ] 创建项目目录,放入附录 A、B、C、D 的初始模板
- [ ] 准备 2-3 份历史方案放入 anchors/positive/,写好 index.md
- [ ] 跑 1-2 个真实项目
- [ ] **手动**修改 constraints.md,记录每次为何修改
- [ ] 不开启自动调参建议(`silent_mode: true`)

### 第 2 周:固化期

- [ ] 根据笔记调整 `suggest_when_*` 阈值
- [ ] 关闭 silent_mode
- [ ] 再跑 2-3 个项目
- [ ] 建议错误率高 → 回调阈值

### 第 3 周起:沉淀期

- [ ] 每个新类型项目完成后,接受"保存为预设"和"加入 anchors"提议
- [ ] 每月 review tuning-history.md
- [ ] 维护 failure-cases.md

---

## 13. 风险、限制与已知问题

### 13.1 残留风险

| 风险 | 严重度 | 缓解 |
|------|-------|------|
| 对标样本本身质量差 | 高 | anchors/ 库 + 适配性检查 |
| Supervisor 在 Phase 1 假设被忽略 | 中 | 高风险假设独立成段 |
| 调参建议噪声 | 中 | 第 2 周校准 |
| Subagents 并发 token 成本 | 中 | max_parallel_workers 限制 |
| 用户长期不用某 project-type 致过时 | 低 | 季度 review |
| state.md 与实际产出文件不同步 | 中 | §11.1 第 1 条强制 |

### 13.2 已知未解决问题(诚实声明,v0.3 议题)

1. **框架不适配的逃生口**:某些项目类型(如紧急响应、纯创意脑暴)四阶段是错的抽象。当前没有规范的"我们不该用这个框架"判定。临时解法:走 A 级或手动跳出。
2. **Subagent token 成本控制**:特别是 Opus,3 个并发 Worker 跑一轮可能很贵。当前只有数量限制,无成本预算。
3. **project-type schema 升级兼容**:如果未来给 project-type 模板加新字段,老的 project-type 文件不会自动迁移。当前需手工补。
4. **跨设备同步**:假设单机使用。多设备使用要靠 git 手动同步。
5. **anchors/ 库的版本管理**:样本本身可能更新(比如对标方案出了新版),当前没有版本字段。

### 13.3 何时应该放弃这套框架

- 极简单的咨询任务(走 A 级)
- 高度即兴的脑暴场景
- 客户在线实时讨论
- 项目类型完全在框架预设之外(参考 13.2 第 1 条)

---

## 14. 附录

### 附录 A:`CLAUDE.md` 初始模板(完整)

```markdown
# 项目编排器系统规则(Claude Code 自动加载)

你现在在项目编排器目录中工作。本文档定义系统规则,分两类:
- §1 结构性铁律:不可豁免
- §2 流程性默认:可经用户显式确认后豁免,但必须留痕

## 会话启动动作(每次会话首条消息后立即执行)

1. 读取 config/constraints.md,确认当前参数
2. 列出 runs/ 下所有 current_phase != done 的 Run
3. 询问用户:"继续未完成的 Run <id>,还是开新任务?"
4. 选择新任务 → 进入 §3 任务级别判定
5. 选择恢复 Run → 读该 Run 的 state.md,按 next_action 续跑

## §1 结构性铁律(不可豁免)

1. state.md 必须维护:任何 Run 进行中,每个动作前读、每个动作后写
2. 文件结构必须遵守模板(见开发指导文档 §5)
3. Worker 必须用模板派工(见开发指导文档 §6)
4. 跳过必须记录:state.md 写 skipped_reason + risk_note
5. 配置变更必须记录到 log/tuning-history.md
6. Phase 1"已知"必须可追溯到用户原文

## §2 流程性默认(可豁免,但需留痕)

按开发指导文档 §11.2 表执行。
用户请求豁免时,必须:
- 复述请求
- 主动陈述至少 2 条风险
- 等用户确认
- 在 state.md 留痕

## §3 任务级别判定

进入新任务时,按开发指导文档 §4.2 的 5 问判定。
Supervisor 提议级别,用户确认。Supervisor 不得自行降级。

## §4 角色定义

- Supervisor(默认角色):全局视野,协调所有阶段,串行任务自任,Worker 派工
- Worker(派生子智能体):仅执行单一原子任务,无全局上下文

## §5 阶段产出文件命名

runs/<日期>-<项目短名>/:
- state.md          状态机
- 00-input.md       原始输入
- 01-frame.md       Phase 1
- 02-anchor.md      Phase 2
- 03-decompose.md   Phase 3
- 04-execute/       Phase 4 目录
  - worker-N.N.md
  - final.md
- review.md         差异分析或 rubric 自查

## §6 调参建议触发

详见 config/constraints.md 中 [meta] 段。
触发时输出建议块,格式见开发指导文档 §8.2。

## §7 简单任务豁免

输入明显是 A 级任务(单一问题/润色/几个点子)时,
主动询问:"启用完整四阶段流程吗?(y/n)"
回 n 退化为普通对话,但仍记录到 log/。

## §8 失败处理

任何任务 rework 达 max_rework_rounds 次,
升级给用户裁决,失败记录追加到 log/failure-cases.md。
```

### 附录 B:`config/constraints.md` 初始模板

(见 §7.1)

### 附录 C:`state.md` 初始模板

```markdown
# Run State

## 基本信息
project_name: <填入>
project_id: <YYYY-MM-DD-short-name>
created_at: <ISO 8601>
last_updated_at: <ISO 8601>
task_level: C
project_type: <匹配的预设,或 untyped>
config_snapshot: <git commit hash>

## 流程位置
current_phase: frame
next_action: 等待用户提供原始需求,然后开始 Phase 1 三栏分析
checkpoint_pending: false

## 阶段产出
completed_files: []
next_required_file: 00-input.md

## Anchor 状态
anchor_status: missing
anchor_path:
anchor_fitness:
skipped_reason:
risk_note:

## Phase 4 执行状态
workers_in_flight: []
workers_completed: []
workers_failed: []

## 流程偏离记录
human_overrides: []
```

### 附录 D:Worker 输出模板

(见 §6.3)

### 附录 E:`README.md` 初始模板

```markdown
# 项目编排器(Project Orchestrator)

本目录是一个基于 Claude Code 的本地教育咨询方案编排系统。

## 快速开始

1. 在本目录打开 Claude Code 终端
2. Claude Code 会自动加载 CLAUDE.md(系统规则)
3. 描述你的项目需求
4. Supervisor 会提议任务级别(A/B/C),你确认后开始

## 我可以改什么

- `config/constraints.md` —— 所有可调参数,改完下次 Run 生效
- `config/project-types/` —— 项目类型预设,可手动 fork
- `anchors/index.md` —— 对标样本索引,新增样本时更新

## 我不应该改什么

- `CLAUDE.md` —— 改前先看开发指导文档 §11(结构性铁律)
- 任何 `runs/<id>/state.md` —— Supervisor 维护,手动改可能破坏恢复

## 文档

- 开发指导文档:project-orchestrator-dev-guide.md
- 调参历史:log/tuning-history.md
- 失败案例:log/failure-cases.md
```

### 附录 F:`anchors/index.md` 初始模板

(见 §10.2)

### 附录 G:`config/project-types/_template.md`

```markdown
# 项目类型:<名称>

## 匹配触发词
- <触发词 1>
- <触发词 2>

## 命中后的默认覆盖
覆盖 constraints.md 中的同名字段:

[brainstorm]
must_ask_about:
  - <本类型必问的关键变量>

[decompose]
atomic_duration_minutes: [<下限>, <上限>]
serial_only_tasks:
  - <本类型典型的 tight 任务>

## 推荐 anchors
- anchors/positive/<样本文件名>
- ...

## 历次教训(自动追加)
```

### 附录 H:词汇表

| 术语 | 含义 |
|------|------|
| Supervisor | 主智能体角色,持全局视野 |
| Worker / Subagent | 子智能体,仅执行单一原子任务 |
| Anchor / 对标样本 | Phase 2 选定的物理参照方案 |
| 耦合性扫描 | Phase 3 中区分 tight/loose/medium 任务的步骤 |
| 调参建议块 | Supervisor 在阶段结束时主动输出的配置调整提议 |
| Run | 一次具体项目的执行实例 |
| state.md | 每个 Run 的机器可读状态机文件 |
| 任务级别 (A/B/C) | 任务复杂度分级,决定走多重的流程 |
| 结构性铁律 | 不可豁免的产出留痕规则 |
| 流程性默认 | 可显式豁免的默认行为 |
| 适配性检查 | Anchor 与本项目可比性的三维评分 |

---

## 15. v0.2 变更说明 + 自检报告

### 15.1 v0.2 处理了哪些 v0.1 缺陷

**来自 OpenAI 反馈的 7 条**:
- [✓] 新增 anchors/ 库 + index.md(§10)
- [✓] 每个 Run 的 state.md(§9)
- [✓] Worker 输出模板(§6.3)
- [✓] 任务级别 A/B/C 入口判定(§4)
- [✓] Iron Laws 拆为结构性 vs 流程性(§11)
- [✓] Phase 1 增加"高风险假设"(§5.1)
- [✓] Phase 3 任务表加"输入依赖"列(§5.3)

**自检发现的 14 条**(v0.1 未覆盖):
- [✓] CLAUDE.md 必须显式指示读 constraints.md(附录 A 会话启动动作)
- [✓] Subagent 不继承 CLAUDE.md,需 prompt 注入机制(§6.2)
- [✓] state.md 需要 `next_action` 而不仅 `next_required_file`(§9.2)
- [✓] Workers in-flight 必须可追踪(§9.2)
- [✓] Phase 4 的"整合"必须作为独立 tight 任务(§5.3, §5.4)
- [✓] Iron Laws 必须区分结构性与流程性,否则要么过硬要么被掏空(§11)
- [✓] constraints.md mid-run 变更语义必须明确(§7.4)
- [✓] Anchor 跳过路径必须有替代 review 模板(§5.4)
- [✓] Phase 1 修正协议(§5.1 末段)
- [✓] Anchor 适配性检查(§5.2)
- [✓] 任务级别由谁决定的协议(§4.2)
- [✓] README 和 failure-cases.md 的实际格式(附录 E, §9.4)
- [✓] max_rework_rounds 命中后的恢复语义(§9.4)
- [✓] rubrics.md 的角色明确化(§7.3)

### 15.2 v0.2.1 处理了哪些 v0.2 实测前缺陷

来自第二轮 OpenAI 反馈,**只挑会污染实测信号的 3 处修复**,其他建议留待 v0.3:

- [✓] **state.md 更新频率降级**(§9.2 + §11.1):从"每个动作前后"改为"状态变化点",避免 token 成本失控污染"框架本身是否好用"的判断
- [✓] **B 级任务加 Mini-plan**(§5.1.5):避免 Frame 后直接 Execute 退化为普通 prompt
- [✓] **任务级别判定默认简短输出**(§4.2):避免每次 5 问的仪式化摩擦污染"分级是否减少过度流程"的判断

**v0.3 待办**(从第二轮反馈中收下,但等实测 3 个 Run 后再决定):
- Anchor 库增加质量评级 + 复核时间 + 优先推荐字段
- Anchor 适配性检查从 3 维扩到 4 维(加"交付用途")
- Anchor 公开源建议加 300-500 字摘要安全阀

### 15.3 v0.2 主动不做的事

为避免过度工程,以下议题已识别但**有意推迟到 v0.3**:

1. **框架不适配的逃生口**:某些任务类型四阶段本身就是错的。当前临时解法是走 A 级。
2. **Subagent 成本预算**:仅有数量限制,无 token 预算。
3. **project-type schema 迁移**:未来 schema 升级时无自动迁移工具。
4. **跨设备同步**:依赖 git 手动同步。
5. **anchors/ 库版本管理**:样本无版本字段。

### 15.4 已知风险:v0.2 自身可能引入的新问题

诚实声明,以下是对 v0.2 设计的自我怀疑,优先在第 1-2 周观察:

1. **state.md 维护成本可能高于收益**(v0.2.1 已部分缓解):频繁更新会消耗 token 且增加错误面。v0.2.1 把频率降到了"状态变化点",但是否够低仍待实测。
2. **任务级别 A/B/C 的判定可能不稳**(v0.2.1 已部分缓解):默认简短判定降低了 LLM 系统偏差的可能,但仍需观察实际分布。
3. **结构性 vs 流程性的拆分可能用户感知不到**:对用户来说差别不大,但内部复杂度增加。如果 1 个月用下来用户从未"豁免"过任何流程性默认,说明这个区分是过度设计,应合并回去。
4. **anchors/ 库可能从未被填充**:如果用户嫌麻烦从不录入,Phase 2 仍然每次临时找,这一节就是死代码。需要观察录入率。
5. **Worker prompt 注入会让派工提示词变长**:每次派工要发 5 段固定内容,可能 token 成本增加 30%+。如果实测成本不可接受,需要把模板做成 Worker 端可引用的文件。

### 15.5 评估时间表

- 跑完 3 个真实 Run 后,review 15.4 的 5 条怀疑,做减法
- 跑完 5 个 Run 后,review 15.2 中 v0.3 待办的 3 项,决定是否进入 v0.3
- 跑完 10 个 Run 后,review 15.3 推迟的 5 条,决定是否进入 v0.3

---

## 文档变更记录

| 版本 | 日期 | 变更 |
|------|------|------|
| v0.1 | 2026-05-21 | 初始设计稿 |
| v0.2 | 2026-05-21 | 补运行闭环:状态机、Anchor 库、Worker 模板、跳过留痕、恢复语义、Subagent 注入;重构 Iron Laws 为结构性/流程性二分;增加任务级别 A/B/C 判定 |
| v0.2.1 | 2026-05-21 | 实测前微调:state.md 频率降级、B 级 Mini-plan、任务级别简短输出 |

---

**下一步建议**:

1. 使用本启动包中的 `project-orchestrator/` 目录(已包含所有附录模板)
2. 在 `anchors/positive/` 放 2-3 份你过往的成功方案,补 `anchors/index.md`
3. 在该目录下 `git init`,然后用 Claude Code 打开
4. 选一个本周确实要做的真实项目跑一遍
5. 第一次跑完后,回头读 §15.4 的 5 条怀疑,看哪些命中
