# 变更管理与影响分析 (Change Management & Impact Analysis)

> 在 SaaS + AI 混合研发中，任何一份文档的变更都可能像多米诺骨牌一样
> 影响到上下游的多个文档和团队。本参考文档建立系统化的变更管理机制，
> 确保每一次变更都经过影响分析、正确审批，并通知到所有受影响方。

---

## 1. 变更类型分类

WisdomCore 将所有文档变更分为五个类型，每种类型有不同的影响范围和审批要求。

### 1.1 变更类型定义表

| 变更类型 | 英文标识 | 影响范围 | 审批要求 | 典型示例 |
|---------|----------|---------|---------|---------|
| 内容更新 | `ContentUpdate` | 仅本文档 | 无需审批（Owner 自主决定） | 修正 PRD 中的错别字；补充实验报告的某个细节；更新 GTM 文档的市场数据 |
| 接口变更 | `InterfaceChange` | 关联文档（直接依赖） | 对端 Owner 审批 | 修改协作协议中的 API 输入/输出字段定义；更改数据格式 |
| 标准变更 | `CriteriaChange` | 关联文档（直接+间接依赖） | 研发总监审批 | 调整协作协议的验收标准阈值（如 accuracy 从 0.80 改为 0.75） |
| 架构变更 | `ArchitecturalChange` | 全局影响 | 架构评审会集体审批 | 新增一个协作协议；拆分或合并现有协议；引入新的文档类型 |
| 紧急修复 | `EmergencyFix` | 关联文档 | 事后补审批（24小时内） | 线上事故导致的协议紧急调整；安全漏洞相关的接口变更 |

### 1.2 变更类型自动判定规则

当 Owner 提交变更时，系统根据以下规则自动判定变更类型：

```python
def classify_change(diff, document):
    changed_fields = extract_changed_frontmatter_fields(diff)
    changed_body_sections = extract_changed_sections(diff)

    # 架构变更：新增或删除文档
    if is_new_document(diff) or is_document_deletion(diff):
        return "ArchitecturalChange"

    # 标准变更：修改了验收标准相关字段
    if any(f in changed_fields for f in [
        "acceptance_criteria", "accuracy_threshold",
        "latency_p99", "performance_target"
    ]):
        return "CriteriaChange"

    # 接口变更：修改了接口定义相关字段或章节
    if any(f in changed_fields for f in [
        "interface_contract", "input_schema", "output_schema",
        "api_endpoint"
    ]) or "接口定义" in changed_body_sections:
        return "InterfaceChange"

    # 其余为内容更新
    return "ContentUpdate"
```

Owner 也可以手动指定变更类型（覆盖自动判定），例如标记为 `EmergencyFix`。

---

## 2. 依赖链追踪 (Dependency Chain Tracking)

### 2.1 依赖关系存储

文档间的依赖关系同时存储在两个位置（双写 + 一致性校验）：

**Frontmatter 中的依赖声明**（Source of Truth）：

```yaml
# PRD 文档示例
related_agreements: ["CA-2026-001", "CA-2026-002"]
related_experiments: []

# 协作协议示例
related_prd: ["PRD-2026-001"]
related_experiments: ["EXP-001", "EXP-002"]
related_adrs: ["ADR-001"]
```

**SQLite `dependencies` 表**（Index，用于快速查询）：

```sql
CREATE TABLE dependencies (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    source_id TEXT NOT NULL,        -- 依赖方文档 ID
    target_id TEXT NOT NULL,        -- 被依赖方文档 ID
    relation_type TEXT NOT NULL,    -- 依赖类型
    description TEXT,               -- 依赖说明
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(source_id, target_id, relation_type)
);
```

### 2.2 十种依赖类型定义

| 依赖类型 | 英文标识 | 含义 | 方向示例 |
|---------|----------|------|---------|
| 依赖关系 | `depends_on` | A 依赖 B（B 是 A 的前置条件） | PRD -> PRD：PRD-B 是 PRD-A 的前置需求 |
| 实现关系 | `implements` | AI 能力实现某个 SaaS 需求 | CA -> PRD：CA 实现了 PRD 中的某个功能 |
| 约束关系 | `governed_by` | A 受 B 约束 | PRD -> CA：需求受协作协议约束 |
| 替代关系 | `supersedes` | 新版本替代旧版本 | CA-v2 -> CA-v1：新版本替代旧版本 |
| 引用关系 | `references` | A 引用了 B（一般性引用） | ADR -> EXP：ADR 引用了实验报告数据 |
| 阻塞关系 | `blocks` | A 阻塞 B | EXP -> PRD：实验未完成阻塞需求交付 |
| 指导关系 | `informs` | 技术决策指导协议的技术选择 | ADR -> CA：ADR 的结论影响了协议的技术方案 |
| 需求依赖 | `requires` | SaaS 需求依赖 AI 能力交付 | PRD -> CA：PRD 需要 CA 定义的 AI 能力 |
| 验证关系 | `validates` | 实验验证协议的验收标准 | EXP -> CA：实验结果验证了协作协议的标准 |
| 派生关系 | `derives` | GTM 材料基于协议和需求派生 | GTM -> CA + PRD：GTM 内容基于两者 |

### 2.3 依赖关系可视化

通过 Skill-H 仪表盘可以查看完整的依赖关系图：

```
PRD-2026-001 (用户中心重构)
  |
  +-- requires --> CA-2026-001 (智能搜索协作协议)
  |                  |
  |                  +-- validates <-- EXP-001 (BERT微调实验)
  |                  +-- validates <-- EXP-002 (DistilBERT实验)
  |                  +-- informs  <-- ADR-001 (模型选型决策)
  |                  +-- derives  --> GTM-BattleCard-001
  |
  +-- requires --> CA-2026-002 (智能推荐协作协议)
  |                  |
  |                  +-- validates <-- EXP-003 (协同过滤实验)
  |
  +-- derives  --> GTM-LaunchPlan-001
```

### 2.4 循环依赖检测

系统在每次文档关联关系变更时，自动检测是否存在循环依赖：

```sql
-- 使用 SQLite 递归 CTE 检测循环依赖
WITH RECURSIVE dep_chain(source_id, target_id, path, depth) AS (
    SELECT source_id, target_id,
           source_id || ' -> ' || target_id,
           1
    FROM dependencies
    WHERE source_id = :changed_doc_id

    UNION ALL

    SELECT d.source_id, d.target_id,
           dc.path || ' -> ' || d.target_id,
           dc.depth + 1
    FROM dependencies d
    JOIN dep_chain dc ON d.source_id = dc.target_id
    WHERE dc.depth < 10
      AND d.target_id != dc.source_id
)
SELECT * FROM dep_chain
WHERE target_id = :changed_doc_id;
```

如果检测到循环依赖，系统会：

1. 阻止该依赖关系的建立
2. 向操作者展示依赖链路径
3. 建议使用 `informs`（弱依赖）替代 `requires`（强依赖）

---

## 3. 变更影响分析报告

### 3.1 报告自动生成逻辑

当文档发生 InterfaceChange 或 CriteriaChange 时，系统自动生成影响分析报告。生成步骤：

1. **查询直接依赖**：通过 SQLite `dependencies` 表查询与变更文档存在直接关联的所有文档
2. **查询间接依赖（二级关联）**：对每个直接依赖文档，继续查询其关联文档（排除变更文档自身）
3. **收集通知人员**：从所有受影响文档中提取 Owner、SaaS Owner、AI Owner
4. **生成报告**：根据以上信息渲染影响分析报告模板

### 3.2 影响分析报告模板

```markdown
## 变更影响分析报告
生成时间: {timestamp}

### 变更概要
- **变更文档**: {doc_id} ({doc_title})
- **变更类型**: {change_type_cn} ({change_type_en})
- **变更内容摘要**:
  - {field_1}: {old_value} -> {new_value}
  - {field_2}: {old_value} -> {new_value}
- **变更原因**: {reason}

### 直接影响 (Direct Impact)
| 文档 | 类型 | 影响说明 | 需要的操作 | Owner |
|------|------|---------|-----------|-------|
| {doc_id} ({title}) | {type} | {impact} | {action_required} | {owner} |

### 间接影响 (Indirect Impact)
| 文档 | 类型 | 影响说明 | 需要的操作 | Owner |
|------|------|---------|-----------|-------|
| {doc_id} ({title}) | {type} | {impact} | {action_required} | {owner} |

### 无影响确认 (No Impact)
| 文档 | 类型 | 确认无影响的原因 |
|------|------|----------------|
| {doc_id} ({title}) | {type} | {reason} |

### 通知清单
- {person} ({role}) -- {action_required}

### 审批要求
- 本次变更为"{change_type_cn}"，需要{approver}审批
- 审批截止时间: {deadline}
```

---

## 4. 变更审批流程

### 4.1 内容更新 (ContentUpdate) 审批流程

```
Owner 修改文档内容
  +-> 创建 PR（标签: content-update，auto-merge 启用，无需人工审批）
  +-> Pre-commit Hook 自动同步 Frontmatter 到 SQLite
  +-> 更新 updated_at 和 next_review_date
  +-> 无需通知（除非修改了 Frontmatter 关键字段）
```

### 4.2 接口变更 (InterfaceChange) 审批流程

```
Owner 修改接口相关内容
  +-> 创建 PR，标签: interface-change
  +-> 系统自动生成影响分析报告
  +-> 系统自动添加对端 Owner 为 PR Reviewer
  +-> 对端 Owner 评审（<=3个工作日）
      +-> 审批通过 -> 合并 PR -> 通知关联文档 Owner
      +-> 驳回 -> 附带理由 -> 提交者修改后重新提交
```

### 4.3 标准变更 (CriteriaChange) 审批流程

```
Owner 修改验收标准或关键指标
  +-> 创建 PR，标签: criteria-change
  +-> 系统自动生成影响分析报告（含间接影响）
  +-> 系统自动添加以下 Reviewer:
      +-> 对端 Owner（必选）
      +-> 研发总监（必选）
      +-> 间接影响文档的 Owner（可选，CC 通知）
  +-> 研发总监审批（<=5个工作日）
      +-> 审批通过 -> 合并 PR -> 通知所有受影响方 -> 更新关联文档的状态标记
      +-> 驳回 -> 附带理由和替代建议 -> 发起讨论会议
```

### 4.4 架构变更 (ArchitecturalChange) 审批流程

```
提案人提交架构变更提案
  +-> 创建 PR，标签: architectural-change
  +-> 系统自动生成全局影响分析报告
  +-> 安排架构评审会议（线上或线下，<=10个工作日内召开）
  +-> 参会人: 研发总监 + 所有受影响文档的 Owner + Tech Lead
  +-> 评审会议决议:
      +-> 通过 -> 合并 PR -> 按计划执行 -> 更新受影响的所有文档
      +-> 有条件通过 -> 修改后重新提交 -> 再次评审
      +-> 否决 -> 记录否决理由 -> 关闭 PR
```

### 4.5 审批超时处理

| 变更类型 | 审批时限 | 超时后的处理 |
|---------|---------|------------|
| InterfaceChange | 3个工作日 | 通知对端 Owner 的上级；再过2天自动升级至研发总监 |
| CriteriaChange | 5个工作日 | 通知研发总监催促；再过3天自动列入下一次团队会议 |
| ArchitecturalChange | 10个工作日 | 通知研发总监安排评审会；再过5天升级至 CTO |
| EmergencyFix | 事后24小时 | 未补审批的紧急变更自动列入下一次 Sprint Review 复盘 |

---

## 5. 紧急变更绿色通道 (Emergency Green Channel)

### 5.1 适用场景

紧急变更绿色通道仅适用于以下场景：

1. **线上事故**：生产环境出现影响用户的故障，需要紧急调整协议或接口
2. **安全漏洞**：发现安全相关问题，需要紧急修改接口定义或数据处理方式
3. **合规紧急要求**：监管部门紧急要求的合规变更

### 5.2 绿色通道流程

```
紧急情况发生
  +-> 任一 Owner 或值班工程师发起紧急变更
      +-> wc doc emergency-change CA-2026-001 \
              --reason "线上搜索服务P99延迟飙升至500ms，需紧急调整阈值" \
              --incident-id "INC-20260423-001"
  +-> 系统标记该变更为 EmergencyFix
  +-> 跳过审批流程，直接修改并合并到 main
  +-> 自动通知: 研发总监 + 双方 Owner + 关联文档 Owner
  +-> 开始24小时倒计时
```

### 5.3 事后补审批要求

紧急变更执行后，必须在 **24小时内** 完成以下步骤：

1. **填写紧急变更记录**：

```yaml
emergency_change:
  incident_id: "INC-20260423-001"
  change_time: "2026-04-23T14:30:00+08:00"
  changed_by: "LiSi"
  change_content: "latency_p99 阈值从 200ms 调整为 300ms"
  reason: "线上搜索服务因流量突增导致 P99 延迟飙升"
  rollback_plan: "流量恢复后将阈值回调至 200ms"
  approved_by: null  # 待补审批
```

2. **创建补审批 PR**：包含紧急变更的 diff 和变更记录
3. **研发总监审批**：确认紧急变更合理性
4. **评估后续影响**：是否需要正式的标准变更或接口变更流程

### 5.4 Sprint Review 复盘要求

所有紧急变更必须在下一次 Sprint Review 中进行复盘，复盘内容包括：

- 事故根因分析（Root Cause Analysis）
- 紧急变更的合理性评估
- 是否需要将临时变更转为正式变更
- 类似问题的预防措施
- 流程改进建议（是否需要调整监控阈值、告警规则等）

---

## 快速参考：变更类型选择决策树

```
文档发生变更
  |
  +-- 是否为新增/删除文档？
  |     +-- 是 --> ArchitecturalChange（架构变更）
  |     +-- 否 --> 继续判断
  |
  +-- 是否为线上事故/安全漏洞/合规紧急？
  |     +-- 是 --> EmergencyFix（紧急修复）
  |     +-- 否 --> 继续判断
  |
  +-- 是否修改了验收标准或关键指标？
  |     +-- 是 --> CriteriaChange（标准变更）
  |     +-- 否 --> 继续判断
  |
  +-- 是否修改了接口定义或数据格式？
  |     +-- 是 --> InterfaceChange（接口变更）
  |     +-- 否 --> ContentUpdate（内容更新）
```
