# 协作协议状态机 (Collaboration Agreement State Machine)

> 协作协议（Collaboration Agreement, CA）是 SaaS 团队与 AI 团队之间的核心治理工具。
> 本文定义协作协议从创建到归档的完整生命周期，包括状态定义、转换规则、
> 违约检测机制、重新协商流程以及版本追溯体系。

---

## 1. 状态定义

协作协议共有 **7 种状态**，覆盖从初创到终结的全生命周期。每种状态都有明确的含义、
允许的操作集合以及最大停留时限（超时将触发自动提醒或升级）。

### 1.1 状态全景表

| 状态 | 英文标识 | Frontmatter 值 | 含义 | 允许的操作 | 最大停留时限 |
|------|----------|----------------|------|-----------|-------------|
| 草稿 | `Draft` | `status: Draft` | 协议初创阶段，内容正在编写中，尚未提交任何一方评审 | 编辑内容、邀请协作者预览、提交评审 | ≤7 自然日 |
| 评审中 | `UnderReview` | `status: UnderReview` | 协议已提交，等待双方 Owner（saas_owner 和 ai_owner）逐一确认 | 审批通过、驳回并附理由、请求修改特定章节 | ≤5 工作日 |
| 生效 | `Active` | `status: Active` | 双方均已审批通过，协议正式生效，指导日常开发与交付 | 执行交付、发起违约标记、发起修订请求、触发定期复审 | 按 `review_cadence_days` 字段（默认14天触发复审提醒） |
| 违约 | `InViolation` | `status: InViolation` | AI 侧交付未达到协议中定义的验收标准（acceptance_criteria），需要干预 | 发起重新协商、决定升级至研发总监、决定归档放弃 | ≤10 自然日（必须在此期限内做出处置决定） |
| 重新协商 | `Renegotiating` | `status: Renegotiating` | 协议正在修订中，可能涉及验收标准调整、模型更换、降级策略变更等 | 编辑协议内容、重新提交评审、放弃修订（回退到 InViolation） | ≤14 自然日 |
| 已替代 | `Superseded` | `status: Superseded` | 该版本协议已被更新版本替代，保留历史记录但不再指导开发 | 只读查看、查询关联的新版本 | 永久（终态） |
| 已归档 | `Archived` | `status: Archived` | 协议所对应的功能已下线或项目已终止，协议进入归档状态 | 只读查看、作为历史参考 | 永久（终态） |

### 1.2 状态详细说明

**草稿（Draft）**

- 创建方式：任一 Owner 通过 Skill-S 的 `wc doc create --type CA` 命令创建
- 协议文件在 `draft/*` 分支上编辑（如 `draft/CA-{YYYY}-{NNN}-{简短描述}`）
- Git 分支命名规范：`draft/CA-{YYYY}-{NNN}-{简短描述}`
- 草稿阶段允许频繁修改，不触发变更通知
- 超过7天未提交评审的草稿，系统自动提醒创建者

**评审中（UnderReview）**

- 进入评审后，文件合并至 `03_Collaboration_Agreements/` 目录（通过 PR 实现）
- 需要 saas_owner 和 ai_owner 两人均审批通过才能生效
- 任一方驳回时，需要附带驳回理由（记录在 PR comments 中）
- 驳回后状态回退到 Draft，保留驳回记录
- 超过5个工作日未完成评审的，系统通知双方 Owner 及其直属上级

**生效（Active）**

- 协议的正常工作状态，可能持续数周到数月
- 系统根据 `review_cadence_days` 字段定期触发复审提醒
- 复审时 Owner 需要确认协议仍然有效，或发起修订
- 每次复审完成后，`updated_at` 和 `next_review_date` 自动更新

**违约（InViolation）**

- 可由系统自动检测触发（见第3节 违约检测机制），也可由任一 Owner 手动标记
- 进入违约状态后，系统自动发送通知给双方 Owner 和研发总监
- 10天内必须做出处置决定：重新协商、归档、或升级
- 超过10天未处置的，自动升级至研发总监，要求强制干预

**重新协商（Renegotiating）**

- 从 InViolation 状态发起，也可以从 Active 状态主动发起（预防性修订）
- 创建新的 draft 分支进行修改，版本号 +1
- 14天内必须完成修订并提交评审，否则自动通知研发总监
- 如果放弃修订，状态回退到之前的状态（InViolation 或 Active）

**已替代（Superseded）和 已归档（Archived）**

- 这两个是终态（Terminal State），不可再转换为任何其他状态
- Superseded 的协议保留 `superseded_by` 字段指向新版本
- Archived 的协议保留完整的 Git history 和 SQLite 记录
- 终态文档在 Skill-H 仪表盘中以灰色显示，但仍可搜索和查看

---

## 2. 状态转换矩阵

### 2.1 ASCII 状态流转图

```
                  提交评审(T1)          双方审批通过(T2)
  [Draft] ──────────────→ [UnderReview] ──────────────→ [Active]
    ↑  │                     │  │                         │  │
    │  │                     │  │ 驳回(T3)                │  │
    │  │←────────────────────┘  │                         │  │
    │  │                        │ 驳回(T14,原Renegotiating)│  │
    │  │    [Renegotiating]←────┘                         │  │
    │  │         ↑   ↑                                    │  │
    │  │         │   └──── 放弃预防性修订(T15)→[Active]   │  │
    │  │         │                                        │  │
    │  │         │ 修订完成(T11)──→[UnderReview]          │  │
    │  │         │                                        │  │
    │  │  ┌──────┤ 发起重新协商(T8)                       │  │
    │  │  │      │ 主动修订(T5)←────────────────────────────┘ │
    │  │  │      │                                            │
    │  │  │      │ 放弃修订(T12)──→[InViolation]             │
    │  │  │      │                                            │
    │  │  │  验收不达标(T4)                                   │
    │  │  │  (自动/手动)←─────────────────────────────────────┘
    │  │  │      │
    │  │  │      ↓
    │  │  └─ [InViolation] ──→ [Archived](T9,放弃)
    │  │         │
    │  │         │ 超时升级(T10)
    │  │         ↓
    │  │   [InViolation](escalated)
    │  │
    │  └──→ [Archived](T13,草稿超30天)
    │
    └──── [Active] ──→ [Superseded](T6,新版本生效)
              │
              └───────→ [Archived](T7,项目终止)
```

### 2.2 完整状态转换表（15 条转换）

| 序号 | 起始状态 | 目标状态 | 触发条件 | 触发者 | 自动执行的动作 |
|------|---------|---------|---------|--------|--------------|
| T1 | Draft | UnderReview | 提交评审（PR created） | 任一 Owner | 通知对方 Owner 进行评审；在 SQLite `status_history` 表插入记录；PR 自动添加 `agreement-review` 标签 |
| T2 | UnderReview | Active | 必要审批人全部通过（High: 3人含研发经理; Medium/Low: 2人双方Owner） | 双方 Owner | 合并 PR 到 main；更新 SQLite `agreements` 表 status 为 Active；计算并写入 `next_review_date`；通知关联 PRD 的 Owner |
| T3 | UnderReview | Draft | 任一方驳回（PR changes requested） | 任一 Owner | 驳回理由记录在 PR comments；状态回退到 Draft；通知提交方 |
| T4 | Active | InViolation | 验收指标连续2周不达标（自动检测），或任一 Owner 手动标记 | 系统自动 / 任一 Owner | 通知双方 Owner + 研发总监；在 SQLite 插入违约记录（含严重度等级）；关联实验报告标记为 `needs_attention` |
| T5 | Active | Renegotiating | Owner 主动发起修订请求（预防性修订） | 任一 Owner | 创建 `renegotiate/CA-{ID}-v{N+1}` 分支；通知对方 Owner |
| T6 | Active | Superseded | 新版本协议（由 Renegotiating 流程产生）生效 | 系统自动 | 旧版本 `superseded_by` 字段填入新版本 ID；SQLite 更新；保留 Git history |
| T7 | Active | Archived | 项目终止或功能下线 | 研发总监审批 | 更新关联 PRD 中的协议引用；通知相关 Owner；归档关联实验报告 |
| T8 | InViolation | Renegotiating | 发起重新协商 | 任一 Owner | 创建 draft 分支；版本号 +1；记录违约原因作为修订背景 |
| T9 | InViolation | Archived | 决定放弃该 AI 能力 | 研发总监审批 | 更新关联 PRD（移除该功能或标记为纯 SaaS 实现）；通知市场部更新 GTM 物料 |
| T10 | InViolation | InViolation（升级） | 超过10天未处置 | 系统自动 | 升级通知至研发总监的上级；标记为 `escalated` |
| T11 | Renegotiating | UnderReview | 修订完成，提交评审 | 任一 Owner | 版本号 +1 写入 Frontmatter；创建 PR；通知对方 Owner 评审 |
| T12 | Renegotiating | InViolation | 放弃修订（14天超时或主动放弃） | 系统自动 / Owner | 状态回退到 InViolation；记录放弃原因 |
| T13 | Draft | Archived | 草稿超过30天未提交评审 | 系统自动（需 Owner 确认） | 发送归档提案通知；Owner 确认后归档 |
| T14 | UnderReview | Renegotiating | 评审驳回（原状态为 Renegotiating 时） | 任一 Owner | 状态回退到 Renegotiating；驳回理由记录在 PR comments |
| T15 | Renegotiating | Active | 放弃预防性修订（原状态为 Active 时） | 任一 Owner | 状态回退到 Active；记录放弃原因 |

### 2.3 非法转换（明确禁止）

以下状态转换被系统强制禁止：

- **Superseded -> 任何状态**：终态不可逆转，如需修改请基于最新版本操作
- **Archived -> 任何状态**：终态不可逆转，如需恢复请创建新协议并注明历史关联
- **Draft -> Active**：不可跳过评审环节
- **Draft -> InViolation**：草稿不存在违约概念
- **UnderReview -> InViolation**：评审中的协议不存在违约概念
- **InViolation -> Active**：不可在违约状态直接恢复，必须经过重新协商

---

## 3. 违约检测机制

### 3.1 自动检测流程

系统通过定时任务（Scheduled Job）自动检测所有 Active 状态协议的达标情况。

**检测频率**：

- 每日扫描（Daily Scan）：检查所有 Active 协议的关键指标
- 每周汇总（Weekly Summary）：生成违约风险报告，发送给研发总监

**检测逻辑（伪代码）**：

```python
def daily_violation_scan():
    active_agreements = sqlite_query(
        "SELECT * FROM agreements WHERE status = 'Active'"
    )
    for agreement in active_agreements:
        criteria = parse_acceptance_criteria(agreement)
        latest_metrics = get_latest_experiment_metrics(agreement.id)

        for criterion in criteria:
            actual_value = latest_metrics.get(criterion.metric_name)
            if actual_value is None:
                log_warning(f"指标 {criterion.metric_name} 无最新数据")
                continue

            gap = calculate_gap(actual_value, criterion.threshold, criterion.operator)

            if gap > 0.30:  # 偏离超过30%
                record_violation(agreement.id, "Critical", criterion, actual_value)
            elif is_consecutive_weeks_below(agreement.id, criterion, weeks=2):
                record_violation(agreement.id, "Major", criterion, actual_value)
            elif gap > 0:   # 单次未达标
                record_violation(agreement.id, "Minor", criterion, actual_value)
            elif gap > -0.10:  # 距阈值不到10%
                record_warning(agreement.id, "Warning", criterion, actual_value)
```

**数据来源**：

- 实验报告（EXP）中的 `results` 字段
- CI/CD Pipeline 的自动化测试结果（如果已集成）
- 手动录入的评估数据（通过 Skill-A 命令 `wc metric record`）

### 3.2 手动标记流程

任一 Owner 可以通过以下方式手动将协议标记为违约：

```bash
# 通过 Skill-S 命令行
wc agreement violate CA-2026-001 \
  --severity Major \
  --reason "最近两周搜索准确率持续低于80%阈值，实际值为72-75%" \
  --evidence EXP-001-run-15,EXP-001-run-16
```

手动标记时必须提供：
- `--severity`：违约严重度（Warning / Minor / Major / Critical）
- `--reason`：违约原因的自然语言描述
- `--evidence`（可选）：关联的实验报告 Run ID 或外部证据链接

### 3.3 违约严重度分级体系（4 级）

| 等级 | 英文 | 判定标准 | 系统响应 | 处置时限 |
|------|------|---------|---------|---------|
| 预警 | Warning | 指标实际值距阈值差距 <10%（即接近不达标） | 通知协议双方 Owner；在 Skill-H 仪表盘标黄 | 无强制时限，下次复审时关注 |
| 轻微 | Minor | 单次检测未达标，但尚未形成趋势 | 通知 Owner；记录到 `violation_log` 表；仪表盘标橙 | 7天内确认是否为偶发 |
| 严重 | Major | 连续2周（>=2次检测周期）未达标 | **自动触发 InViolation 状态转换**；通知 Owner + 研发总监；仪表盘标红 | 10天内必须做出处置决定 |
| 致命 | Critical | 核心指标严重偏离阈值（>30%），或出现安全/合规相关的违约 | **自动触发 InViolation**；暂停关联 SaaS 功能的新开发（发出建议，需研发总监确认）；通知高管 | 5天内必须做出处置决定 |

### 3.4 违约升级路径

违约升级遵循逐级上报原则：

```
Warning（预警）
  |-- 通知协议双方 Owner（IM 消息 + 邮件）
  |-- 在 Skill-H 仪表盘标记黄色警告图标
  |-- 不改变协议状态

Minor（轻微违约）
  |-- 通知 Owner（IM 消息 + 邮件）
  |-- 在 SQLite violation_log 表记录违约详情
  |-- 在 Skill-H 仪表盘标记橙色
  |-- 不改变协议状态（但连续2次 Minor 自动升级为 Major）

Major（严重违约）
  |-- 自动将协议状态从 Active 转换为 InViolation
  |-- 通知 Owner + 研发总监（IM 消息 + 邮件 + 日历提醒）
  |-- 在 Skill-H 仪表盘标记红色，首页置顶展示
  |-- 自动创建"违约处置"待办事项

Critical（致命违约）
  |-- 自动将协议状态从 Active 转换为 InViolation
  |-- 通知 Owner + 研发总监 + CTO/VP Engineering
  |-- 向关联 SaaS 功能的 PRD Owner 发送风险通知
  |-- 建议暂停关联 SaaS 功能的新开发（需研发总监确认执行）
  |-- 在 Skill-H 仪表盘标记红色闪烁，全局横幅提醒
  |-- 自动安排紧急会议（如已集成日历系统）
```

---

## 4. 重新协商流程

当协议进入 InViolation 状态（或 Owner 从 Active 状态主动发起修订），
需要通过重新协商流程修订协议内容。以下为完整的 Step-by-step 流程。

### 步骤一：发起重新协商

发起方（任一 Owner）执行：

```bash
wc agreement renegotiate CA-2026-001 \
  --reason "BERT 模型在新增数据上 accuracy 下降至 72%，需要调整阈值或更换模型"
```

系统自动执行：
1. 将协议状态从 InViolation（或 Active）变更为 Renegotiating
2. 创建 Git 分支 `renegotiate/CA-2026-001-v2`
3. 在该分支上复制当前协议文件，版本号 +1（`version: 2`）
4. 在 SQLite `status_history` 表插入状态变更记录
5. 通知对方 Owner："CA-2026-001 已进入重新协商阶段，请在14天内完成修订"

### 步骤二：修订协议内容

修订范围可能包括但不限于：
- **降低验收标准**：例如 `accuracy_threshold` 从 0.80 调整为 0.75
- **更换模型/方案**：例如从 BERT 切换到 DistilBERT 以满足延迟要求
- **调整降级策略**：例如增加更完善的 fallback 方案
- **修改接口定义**：例如增减输入/输出字段
- **调整评审周期**：例如将 `review_cadence_days` 从 14 改为 7（加强监控）

修订时必须同步更新以下 Frontmatter 字段：

```yaml
version: 2  # 版本号 +1
updated_at: 2026-05-10  # 修订日期
change_reason: "BERT模型accuracy下降，调整验收标准并增加降级策略"
previous_version: "CA-2026-001-v1"  # 指向前一版本
```

### 步骤三：双方评审

修订完成后，提交评审：

```bash
wc agreement submit-review CA-2026-001-v2
```

系统自动创建 PR，评审流程与新协议相同（参见转换 T2）：
- saas_owner 和 ai_owner 均需审批通过
- 任一方驳回则回退到 Renegotiating 状态继续修改
- 评审期限：5个工作日

### 步骤四：新版本生效

双方审批通过后：
1. PR 合并到 main 分支
2. 新版本协议状态设为 Active
3. 旧版本协议状态自动设为 Superseded，`superseded_by` 填入新版本 ID
4. SQLite 中更新关联关系
5. 通知所有相关方（PRD Owner、实验 Owner 等）

### 步骤五：收尾与同步

- 如果验收标准发生变化，通知关联实验报告的 Owner 更新实验目标
- 如果接口定义发生变化，通知 SaaS 开发团队更新集成代码
- 在 Skill-H 仪表盘中更新协议版本线，展示版本演进历史

---

## 5. 协议版本追溯

### 5.1 版本号规则

每个协作协议的 Frontmatter 中包含 `version` 字段，从 1 开始递增：

```yaml
# 第一版
id: CA-2026-001
version: 1

# 重新协商后的第二版
id: CA-2026-001
version: 2
superseded_by: null
previous_version: "CA-2026-001-v1"
```

版本号仅在重新协商流程中递增，日常的内容更新（如修正错别字）不改变版本号。

### 5.2 Git History 追溯

通过 Git history 可以查看每个版本的详细变更：

```bash
# 查看协议的完整修改历史
git log --follow 03_Collaboration_Agreements/CA-2026-001.md

# 查看两个版本之间的差异
git diff v1.0..v2.0 -- 03_Collaboration_Agreements/CA-2026-001.md
```

### 5.3 SQLite 状态历史表

`status_history` 表记录所有状态变更，结构如下：

```sql
CREATE TABLE status_history (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    document_id TEXT NOT NULL,           -- 文档 ID (如 CA-2026-001)
    document_type TEXT NOT NULL,         -- 文档类型 (如 CA)
    from_status TEXT,                    -- 变更前状态 (首次创建时为 NULL)
    to_status TEXT NOT NULL,             -- 变更后状态
    changed_by TEXT NOT NULL,            -- 操作人
    change_reason TEXT,                  -- 变更原因
    violation_severity TEXT,             -- 违约严重度 (仅 InViolation 时有值)
    metadata_json TEXT,                  -- 额外元数据 (如关联的 PR 号，JSON 格式)
    changed_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    commit_sha TEXT,                     -- 关联的 Git commit SHA
    FOREIGN KEY (document_id) REFERENCES documents(id)
);
```

### 5.4 Skill-H 仪表盘的协议生命线

在 Skill-H 仪表盘中，每个协议都有一个可视化的"生命线"（Timeline），展示：

- 创建时间（Draft）
- 首次生效时间（Active）
- 所有状态变更节点（含时间、操作人、原因）
- 违约事件标记（红色节点）
- 版本升级标记（蓝色节点）
- 当前状态高亮显示
