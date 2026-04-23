# 父Skill 参考手册 -- 产品知识本体库

> 本文档为 WisdomCore 父Skill 的操作级参考，提炼自设计方案第四章。
> 包含五类文档的完整 Frontmatter 模板、正文模板、标签体系、命名规范和编号系统。

---

## 1. 父Skill 角色定义

### 1.1 什么是父Skill

父Skill **不是代码**，而是一套**规范（Specification）**。它定义了智核系统的"骨架"：

- **目录结构**：文件放在哪里、如何组织
- **文档模板**：每类文档的 Frontmatter 字段和正文结构
- **命名规范**：文件名、ID、slug 的生成规则
- **标签体系**：标签的分类、命名和治理规则
- **状态机**：每类文档的合法状态和转换路径

### 1.2 父Skill 与子Skill 的关系

```
父Skill（产品知识本体库）
  ├── 定义: 目录结构、模板、命名规范、标签体系
  ├── 输出: .meta/ 目录下的所有配置文件
  └── 约束: 子Skill 必须遵守父Skill 定义的规范
      │
      ├── 子Skill-S: 协作协议生成器 (Agreement Forge)
      │   └── 操作: 创建/审批/监控/归档 CollaborationAgreement
      ├── 子Skill-A: 实验记录员 (Experiment Log)
      │   └── 操作: 记录/评估/决策 Experiment + ADR
      ├── 子Skill-H: 混合视图生成器 (Hybrid View)
      │   └── 操作: 查询/统计/预警 全类型文档状态（PRD/EXP/CA/ADR/GTM）；聚合分析、趋势报告、异常预警
      └── 子Skill-G: GTM知识架构师 (GTM Architect)
          └── 操作: 从PRD/实验自动生成 GTM 资产
```

### 1.3 父Skill 的交付物

| 交付物 | 路径 | 说明 |
|-------|------|------|
| 全局配置 | `.meta/config.yaml` | 团队、产品线、默认值 |
| 团队白名单 | `.meta/team-roster.yaml` | 成员列表与角色 |
| 标签体系 | `.meta/tag-taxonomy.yaml` | 二级分类标签定义 |
| PRD Schema | `.meta/schema/prd.schema.json` | PRD Frontmatter 校验 |
| 实验 Schema | `.meta/schema/experiment.schema.json` | 实验报告 Frontmatter 校验 |
| 协议 Schema | `.meta/schema/agreement.schema.json` | 协作协议 Frontmatter 校验 |
| ADR Schema | `.meta/schema/adr.schema.json` | ADR Frontmatter 校验 |
| GTM Schema | `.meta/schema/gtm.schema.json` | GTM资产 Frontmatter 校验 |
| SQLite Schema | `.meta/schema.sql` | 数据库建表语句 |
| 文档模板（7个） | `.meta/templates/*.md` | 各类文档的新建模板 |
| Pre-commit Hook | `.meta/hooks/pre-commit` | 校验 + 同步 |
| Commit-msg Hook | `.meta/hooks/commit-msg` | 提交信息格式校验 |
| 同步脚本 | `.meta/scripts/sync_frontmatter.py` | Frontmatter -> SQLite |
| 校验脚本 | `.meta/scripts/validate_frontmatter.py` | Frontmatter 格式校验 |
| 重建脚本 | `.meta/scripts/rebuild_index.sh` | 全量重建索引 |
| 度量快照脚本 | `.meta/scripts/metrics_snapshot.py` | 每周自动采集 6 大核心指标快照 |
| 仪表盘生成脚本 | `.meta/scripts/generate_dashboard.py` | 生成健康度仪表盘与趋势图表 |

---

## 2. 五类文档 Frontmatter 完整模板

### 2.1 SaaS需求文档 (PRD)

```yaml
---
# === 基础标识 ===
id: PRD-2026-001
type: PRD
title: "用户中心重构"

# === 状态与优先级 ===
status: Draft                    # Draft | InReview | Approved | InDev | Released | Archived
priority: P1                    # P0（紧急）| P1（高）| P2（中）| P3（低）

# === 责任人 ===
owner: "ZhangSan"               # 需求负责人（Product Owner）
tech_lead: "LiSi"               # 技术负责人

# === 组织归属（预留多团队） ===
team: "user-platform"
product_line: "main-app"

# === 发布计划 ===
target_release: "v2.1.0"        # 目标发布版本号
target_date: 2026-06-30          # 目标上线日期

# === AI关联 ===
ai_dependency: true              # 是否依赖AI能力
ai_components:                   # AI组件清单（当 ai_dependency=true 时填写）
  - name: "智能推荐引擎"
    type: "ML Model"
    maturity: "Experiment"       # Experiment | Validated | Production

# === 跨文档关联 ===
related_agreements:              # 关联的协作协议
  - "CA-2026-001"
related_experiments: []          # 关联的AI实验
related_adrs: []                 # 关联的架构决策
jira_epic_key: "PROJ-123"       # Jira Epic 关联（可选）

# === 风险评估 ===
risk_level: Medium               # High | Medium | Low
risk_notes: "依赖推荐引擎实验结果"

# === 标签 ===
tags:
  - "UserCenter"
  - "Refactor"
  - "AI-Enhanced"

# === 时间信息 ===
created_at: 2026-04-20
updated_at: 2026-04-23
next_review_date: 2026-05-07
---
```

**PRD 状态机：**

```
Draft -> InReview -> Approved -> InDev -> Released -> Archived
                  \-> Draft (打回修改)
```

### 2.2 AI实验报告 (Experiment)

```yaml
---
# === 基础标识 ===
id: EXP-001
type: Experiment
title: "LLM辅助代码审查效果评估"

# === 状态 ===
status: InProgress               # Proposed | InProgress | Completed | Validated | Rejected | Archived

# === 责任人 ===
ai_owner: "WangWu"              # AI侧实验负责人
saas_owner: "ZhangSan"          # SaaS侧利益相关人（可选）

# === 组织归属 ===
team: "ai-platform"
product_line: "ai-copilot"

# === 实验设计 ===
hypothesis: "使用LLM进行代码审查可将人工审查时间减少40%以上"
methodology: "A/B Test"          # A/B Test | Benchmark | Prototype | Survey | Pilot
duration_weeks: 4                # 预计实验周期（周）
start_date: 2026-04-01
end_date: 2026-04-28             # 预计或实际结束日期

# === 关键指标 ===
key_metrics:
  - metric: "review_time_reduction"
    description: "人工审查时间减少比例"
    baseline: 0
    target: 40
    actual: null                 # 实验完成后填写
    unit: "%"
  - metric: "false_positive_rate"
    description: "误报率"
    baseline: null
    target: 10
    actual: null
    unit: "%"
  - metric: "developer_satisfaction"
    description: "开发者满意度评分"
    baseline: null
    target: 4.0                  # 1-5分
    actual: null
    unit: "score"

# === 资源需求 ===
resources:
  compute: "GPU A100 x 2"
  data: "近6个月代码审查记录"
  budget_estimate: "¥15,000"

# === 跨文档关联 ===
related_requirements:
  - "PRD-2026-001"
related_agreements: []
output_adr: null                 # 实验结论输出的 ADR（如有）

# === 风险评估 ===
risk_level: Medium
risk_notes: "模型幻觉可能导致误导性审查意见"

# === 标签 ===
tags:
  - "CodeReview"
  - "LLM"
  - "DeveloperTools"

# === 时间信息 ===
created_at: 2026-03-25
updated_at: 2026-04-23
next_review_date: 2026-04-30
---
```

**Experiment 状态机：**

```
Proposed -> InProgress -> Completed -> Validated -> (输出ADR/CA)
                       \-> Rejected -> Archived
                       \-> Archived (中止)
```

### 2.3 协作协议 (CollaborationAgreement)

协作协议是智核系统中最复杂、最核心的文档类型，定义 SaaS 与 AI 两侧的接口约定、质量标准和降级策略。

```yaml
---
# === 基础标识 ===
id: CA-2026-001
type: CollaborationAgreement
title: "AI代码审查服务质量协议"
version: 1                       # 协议版本号，重大修订时递增

# === 状态 ===
status: Active                   # Draft | UnderReview | Active | InViolation |
                                 # Renegotiating | Superseded | Archived

# === 责任人（双方必须指定） ===
saas_owner: "ZhangSan"
ai_owner: "WangWu"
approvers:                       # 审批人列表（Active前必须全部签署）
  - name: "LiSi"
    role: "Engineering Manager"
    approved: true
    approved_at: 2026-04-18
  - name: "ZhaoLiu"
    role: "AI Tech Lead"
    approved: true
    approved_at: 2026-04-19

# === 组织归属 ===
team: "user-platform"
product_line: "ai-copilot"

# === 协议范围 ===
scope: "定义AI代码审查服务的质量标准、响应时间、准确率要求及降级策略"
applicable_services:
  - "code-review-bot"
  - "pr-summary-generator"

# === 验收标准（核心） ===
acceptance_criteria:
  - metric: "accuracy"
    description: "审查建议的准确率"
    operator: ">="
    threshold: 85
    unit: "%"
    measurement_method: "人工抽样验证，每周随机抽取50条"
  - metric: "latency_p99"
    description: "99分位响应延迟"
    operator: "<="
    threshold: 30
    unit: "seconds"
    measurement_method: "APM监控系统自动采集"
  - metric: "availability"
    description: "服务可用率"
    operator: ">="
    threshold: 99.5
    unit: "%"
    measurement_method: "健康检查端点，每分钟探测"
  - metric: "false_positive_rate"
    description: "误报率（无效建议比例）"
    operator: "<="
    threshold: 15
    unit: "%"
    measurement_method: "开发者反馈标记，月度统计"

# === 降级策略（核心） ===
fallback_strategy: |
  当AI代码审查服务不满足验收标准时，执行以下降级策略：
  1. 自动切换为"建议模式"（仅提示，不阻塞合并）
  2. 通知SaaS Owner和AI Owner
  3. AI团队在24小时内提供根因分析
  4. 48小时内出具修复方案或重新协商阈值
fallback_trigger: "连续3次周度指标不达标，或单次Critical级Bad Case"
fallback_contact: "WangWu (AI Owner), ZhangSan (SaaS Owner)"

# === 评审机制 ===
review_cadence_days: 14
next_review_date: 2026-05-07
review_checklist:
  - "验收标准是否仍然合理"
  - "是否有新的Bad Case需要纳入"
  - "降级策略是否需要更新"
  - "服务范围是否需要扩展/收缩"

# === 违约记录 ===
violation_history: []
# 示例:
# - date: 2026-05-01
#   metric: "accuracy"
#   expected: ">= 85%"
#   actual: "78%"
#   resolution: "模型微调，2026-05-05恢复"

# === 跨文档关联 ===
related_requirements:
  - "PRD-2026-001"
related_experiments:
  - "EXP-001"
supersedes: null
superseded_by: null

# === 风险评估 ===
risk_level: High
risk_notes: "核心开发流程依赖，影响全团队代码合并效率"

# === 标签 ===
tags:
  - "CodeReview"
  - "SLA"
  - "AI-Service"
  - "Critical-Path"

# === 时间信息 ===
created_at: 2026-04-15
updated_at: 2026-04-23
---
```

**CollaborationAgreement 状态机：**

```
Draft -> UnderReview -> Active -> InViolation
                           |  -> Renegotiating -> UnderReview (重新评审)
                           |                  \-> InViolation (放弃修订)
                           +-> Superseded -> Archived
                           +-> Archived (直接归档)
```

### 2.4 架构决策记录 (ADR)

```yaml
---
# === 基础标识 ===
id: ADR-001
type: ADR
title: "向量数据库选型：Milvus vs Pinecone vs pgvector"

# === 状态 ===
status: Accepted                 # Proposed | Accepted | Deprecated | Superseded

# === 责任人 ===
proposer: "WangWu"
reviewers:
  - "ZhangSan"
  - "LiSi"

# === 组织归属 ===
team: "ai-platform"
product_line: "ai-copilot"

# === 决策上下文 ===
decision_date: 2026-04-10
decision: "Milvus"               # 最终决策结论（简短）
decision_drivers:
  - "需要支持100M+向量规模"
  - "团队已有Kubernetes运维能力"
  - "开源优先，避免供应商锁定"

# === 候选方案 ===
alternatives_considered:
  - name: "Milvus"
    pros: ["开源", "高性能", "社区活跃"]
    cons: ["运维复杂度较高"]
  - name: "Pinecone"
    pros: ["全托管", "即开即用"]
    cons: ["闭源", "成本较高", "数据出境风险"]
  - name: "pgvector"
    pros: ["复用PostgreSQL", "运维简单"]
    cons: ["大规模性能不足", "功能有限"]

# === 影响范围 ===
impact_scope:
  - "RAG知识库服务"
  - "语义搜索引擎"
  - "推荐系统向量索引"

# === 跨文档关联 ===
related_experiments:
  - "EXP-002"
related_requirements:
  - "PRD-2026-003"
related_agreements: []
supersedes: null
superseded_by: null

# === 风险评估 ===
risk_level: High
risk_notes: "数据库选型影响长期架构，迁移成本高"

# === 标签 ===
tags:
  - "VectorDB"
  - "Infrastructure"
  - "RAG"

# === 时间信息 ===
created_at: 2026-04-05
updated_at: 2026-04-23
next_review_date: 2026-07-10     # 3个月后回顾
---
```

**ADR 状态机：**

```
Proposed -> Accepted -> Deprecated（技术过时）
                     -> Superseded（被新ADR替代）
```

### 2.5 GTM资产

GTM（Go-To-Market）资产包含三个子类型，共享基础 Frontmatter 字段，各有扩展字段。

#### 2.5.1 Release Note

```yaml
---
# === 基础标识 ===
id: GTM-RN-v2.1.0
type: GTM
gtm_subtype: ReleaseNote         # ReleaseNote | BattleCard | FeatureBrief
title: "v2.1.0 版本发布说明"

# === 状态 ===
status: Draft                    # Draft | InReview | Published | Archived

# === 责任人 ===
owner: "ZhaoLiu"
saas_owner: "ZhangSan"

# === 组织归属 ===
team: "user-platform"
product_line: "main-app"

# === Release Note 专有字段 ===
release_version: "v2.1.0"
release_date: 2026-06-30
highlights:
  - "AI代码审查功能正式上线"
  - "用户中心全面重构，性能提升60%"
  - "新增智能通知系统"
breaking_changes: false
target_audience:
  - "终端用户"
  - "系统管理员"

# === 跨文档关联 ===
included_prds:
  - "PRD-2026-001"
  - "PRD-2026-002"
included_experiments:
  - "EXP-001"

# === 分发渠道 ===
distribution_channels:
  - "产品内公告"
  - "邮件通知"
  - "飞书群公告"

# === 标签 ===
tags:
  - "Release"
  - "v2.1.0"

# === 时间信息 ===
created_at: 2026-06-25
updated_at: 2026-06-28
---
```

#### 2.5.2 Battle Card

```yaml
---
# === 基础标识 ===
id: GTM-BC-competitor-alpha
type: GTM
gtm_subtype: BattleCard
title: "竞品对抗卡：Competitor Alpha"

# === 状态 ===
status: Published                # Draft | InReview | Published | Outdated | Archived

# === 责任人 ===
owner: "SunQi"

# === 组织归属 ===
team: "growth"
product_line: "main-app"

# === Battle Card 专有字段 ===
competitor_name: "Competitor Alpha"
competitor_website: "https://competitor-alpha.com"
last_intel_date: 2026-04-15
our_advantages:
  - category: "AI能力"
    points:
      - "深度集成AI代码审查，竞品仅有基础lint"
      - "RAG知识库支持私有化部署"
  - category: "定价"
    points:
      - "同等功能下价格低30%"
our_weaknesses:
  - category: "市场认知"
    points:
      - "品牌知名度不及竞品"
counter_arguments:
  - objection: "竞品市场份额更大"
    response: "市场份额不等于产品力，我们在AI增强领域领先一代"
win_rate: 62
confidence: "Medium"             # High | Medium | Low

# === 标签 ===
tags:
  - "Competitive"
  - "Sales-Enablement"

# === 时间信息 ===
created_at: 2026-03-01
updated_at: 2026-04-23
next_review_date: 2026-05-15
---
```

#### 2.5.3 Feature Brief

```yaml
---
# === 基础标识 ===
id: GTM-FB-ai-code-review
type: GTM
gtm_subtype: FeatureBrief
title: "功能简报：AI代码审查"

# === 状态 ===
status: Published                # Draft | InReview | Published | Archived

# === 责任人 ===
owner: "ZhaoLiu"

# === 组织归属 ===
team: "growth"
product_line: "ai-copilot"

# === Feature Brief 专有字段 ===
feature_name: "AI代码审查"
feature_version: "v1.0"
elevator_pitch: "让每个PR都有一位经验丰富的AI审查员，审查时间缩短40%，代码质量提升25%"
target_personas:
  - persona: "技术负责人"
    pain_point: "代码审查是瓶颈，高级工程师时间被大量占用"
    value_prop: "释放高级工程师时间，同时不降低审查质量"
  - persona: "开发者"
    pain_point: "等待审查时间长，反馈不及时"
    value_prop: "秒级AI预审查，即时获得改进建议"
key_differentiators:
  - "基于团队代码风格定制化训练"
  - "支持私有化部署，代码不出域"
  - "与现有CI/CD流水线无缝集成"
demo_link: "https://demo.example.com/ai-code-review"
assets:
  - type: "演示视频"
    url: "https://assets.example.com/demo-video.mp4"
  - type: "一页纸"
    url: "https://assets.example.com/one-pager.pdf"

# === 跨文档关联 ===
source_prd: "PRD-2026-001"
source_experiment: "EXP-001"
related_agreement: "CA-2026-001"

# === 标签 ===
tags:
  - "FeatureLaunch"
  - "AI-CodeReview"
  - "Sales-Enablement"

# === 时间信息 ===
created_at: 2026-04-20
updated_at: 2026-04-23
---
```

---

## 3. 文档正文模板

### 3.1 PRD 正文模板

```markdown
## 1. 背景与问题
<!-- 描述当前痛点和业务背景 -->

## 2. 目标与非目标
### 目标
-

### 非目标（明确排除的范围）
-

## 3. 用户故事
- 作为 [角色]，我希望 [行为]，以便 [价值]

## 4. 功能需求
### 4.1 功能点一
- 描述:
- 验收标准:
- 优先级:

### 4.2 功能点二
- 描述:
- 验收标准:
- 优先级:

## 5. 非功能需求
- 性能:
- 安全:
- 可用性:

## 6. AI组件依赖（如适用）
| AI组件 | 依赖类型 | 关联协议 | 降级方案 |
|--------|---------|---------|---------|
|        |         |         |         |

## 7. 技术方案概述

## 8. 里程碑与排期
| 里程碑 | 日期 | 交付物 |
|--------|------|--------|
|        |      |        |

## 9. 风险与依赖
| 风险/依赖 | 影响程度 | 缓解措施 |
|-----------|---------|---------|
|           |         |         |

## 10. 附录
```

### 3.2 Experiment 正文模板

```markdown
## 1. 实验背景
<!-- 为什么发起这个实验？解决什么问题？ -->

## 2. 假设 (Hypothesis)
> **假设**: [具体假设内容]

## 3. 实验设计
### 3.1 方法论
<!-- A/B Test / Benchmark / Prototype / Survey / Pilot -->

### 3.2 实验变量
- 自变量:
- 因变量:
- 控制变量:

### 3.3 数据采集方案

### 3.4 样本量与周期
- 样本量:
- 实验周期:

## 4. 实验结果
### 4.1 数据汇总
| 指标 | 基线 | 目标 | 实际 | 达标? |
|------|------|------|------|-------|
|      |      |      |      |       |

### 4.2 分析与解读

### 4.3 Bad Cases 汇总
| # | 描述 | 严重程度 | 根因 |
|---|------|---------|------|
|   |      |         |      |

## 5. 结论与建议
### 5.1 假设验证结果

### 5.2 建议行动
- [ ] 行动项一
- [ ] 行动项二

### 5.3 后续实验（如需要）

## 6. 附录
```

### 3.3 CollaborationAgreement 正文模板

```markdown
## 1. 协议概述
<!-- 用1-2段话描述本协议的目的和范围 -->

## 2. 服务描述
### 2.1 AI服务能力
### 2.2 SaaS侧集成要求

## 3. 接口规范
### 3.1 API 接口
### 3.2 数据格式
### 3.3 错误码定义
| 错误码 | 含义 | SaaS侧处理方式 |
|--------|------|----------------|
|        |      |                |

## 4. 质量标准详述
### 4.1 指标定义
### 4.2 测量方法
### 4.3 报告频率

## 5. 降级策略详述
### 5.1 触发条件
### 5.2 降级执行步骤
### 5.3 恢复流程
### 5.4 通知链

## 6. 运维约定
### 6.1 监控告警
### 6.2 变更管理
### 6.3 事故响应

## 7. 评审与修订
### 7.1 定期评审日程
### 7.2 修订流程
### 7.3 版本管理

## 8. 签署记录
| 角色 | 姓名 | 签署日期 | 备注 |
|------|------|---------|------|
| SaaS Owner | | | |
| AI Owner | | | |
| Engineering Manager | | | |
```

### 3.4 ADR 正文模板

```markdown
## 1. 上下文 (Context)
<!-- 什么情况促使了这个决策？ -->

## 2. 决策驱动因素 (Decision Drivers)
- 因素一:
- 因素二:

## 3. 候选方案 (Considered Options)
### 方案A: [名称]
- **描述**:
- **优点**:
- **缺点**:
- **成本估算**:

### 方案B: [名称]
- **描述**:
- **优点**:
- **缺点**:
- **成本估算**:

### 方案C: [名称]
- **描述**:
- **优点**:
- **缺点**:
- **成本估算**:

## 4. 决策结果 (Decision Outcome)
> **选择**: [方案X]
> **原因**: [一句话总结]

### 4.1 正面影响
### 4.2 负面影响 / 技术债
### 4.3 需要的后续行动

## 5. 验证计划
- 验证指标:
- 验证周期:
- 回退条件:

## 6. 参考资料
```

### 3.5 GTM 正文模板 -- Release Note

```markdown
## 版本亮点
<!-- 3-5 条核心亮点，面向用户的语言 -->

## 新功能
### [功能名称一]
### [功能名称二]

## 改进

## 修复

## Breaking Changes（如有）

## 已知问题

## 升级指南
```

---

## 4. 标签体系 (Tag Taxonomy)

### 4.1 二级分类定义

标签采用二级结构：**分类 (Category)** > **标签 (Tag)**。

```yaml
# .meta/tag-taxonomy.yaml
taxonomy_version: "1.0.0"

categories:
  # === 业务领域 ===
  Domain:
    description: "业务领域分类，标识文档所属的功能域"
    tags:
      UserCenter:       { display_name: "用户中心" }
      Payment:          { display_name: "支付" }
      Notification:     { display_name: "通知系统" }
      DataPlatform:     { display_name: "数据平台" }
      ContentManagement: { display_name: "内容管理" }
      Search:           { display_name: "搜索" }

  # === 技术领域 ===
  Technology:
    description: "技术标签，标识涉及的技术方向"
    tags:
      LLM:              { display_name: "大语言模型" }
      RAG:              { display_name: "RAG" }
      VectorDB:         { display_name: "向量数据库" }
      CodeReview:       { display_name: "代码审查" }
      MLOps:            { display_name: "MLOps" }
      Infrastructure:   { display_name: "基础设施" }
      Refactor:         { display_name: "重构" }
      DeveloperTools:   { display_name: "开发者工具" }

  # === 风险标记 ===
  Risk:
    description: "风险相关标签"
    tags:
      Critical-Path:    { display_name: "关键路径" }
      Data-Privacy:     { display_name: "数据隐私" }
      Security:         { display_name: "安全" }
      Compliance:       { display_name: "合规" }

  # === 阶段标记 ===
  Stage:
    description: "文档所处的生命周期阶段"
    tags:
      AI-Enhanced:      { display_name: "AI增强" }
      FeatureLaunch:    { display_name: "功能上线" }
      Deprecated:       { display_name: "已废弃" }

  # === GTM 标记 ===
  GTM:
    description: "Go-To-Market 相关标签"
    tags:
      Sales-Enablement: { display_name: "销售赋能" }
      Competitive:      { display_name: "竞品分析" }
      Release:          { display_name: "版本发布" }
      SLA:              { display_name: "SLA" }
      AI-Service:       { display_name: "AI服务" }
      AI-CodeReview:    { display_name: "AI代码审查" }
```

### 4.2 标签治理规则

```yaml
governance:
  max_tags_per_document: 8
  min_tags_per_document: 1
  tag_name_pattern: "^[A-Za-z][A-Za-z0-9_-]{1,30}$"
  review_cadence_days: 90
```

1. **注册制**：所有标签必须在 `tag-taxonomy.yaml` 中注册，未注册标签无法通过 Pre-commit 校验
2. **数量限制**：每个文档最少 1 个、最多 8 个标签
3. **命名规范**：首字母大写，英文或英文缩写，使用连字符分隔单词（如 `AI-Enhanced`）
4. **废弃流程**：标签废弃需经过 90 天过渡期，期间标签标记为 `deprecated`
5. **定期评审**：每季度评审标签体系，清理无文档引用的标签，合并含义重叠的标签

---

## 5. 文档命名规范

### 5.1 各类型文档命名模式

| 文档类型 | 文件名模式 | 示例 |
|---------|-----------|------|
| PRD | `PRD-{YYYY}-{NNN}-{slug}.md` | `PRD-2026-001-user-center-refactor.md` |
| Experiment | `EXP-{NNN}-{slug}.md` | `EXP-001-llm-code-review.md` |
| CollaborationAgreement | `CA-{YYYY}-{NNN}-{slug}.md` | `CA-2026-001-ai-code-review-sla.md` |
| ADR | `ADR-{NNN}-{slug}.md` | `ADR-001-vector-db-selection.md` |
| GTM - Release Note | `RN-{version}.md` | `RN-v2.1.0.md` |
| GTM - Battle Card | `BC-{competitor-slug}.md` | `BC-competitor-alpha.md` |
| GTM - Feature Brief | `FB-{feature-slug}.md` | `FB-ai-code-review.md` |

### 5.2 Slug 生成规则

Slug 是文件名中的人类可读描述部分：

1. **使用英文**：中文标题需转换为英文缩写或 pinyin
   - "用户中心重构" -> `user-center-refactor`
   - "AI代码审查服务质量协议" -> `ai-code-review-sla`
   - "向量数据库选型" -> `vector-db-selection`

2. **格式要求**：
   - 全小写
   - 单词间用连字符 `-` 分隔
   - 长度 3-50 字符
   - 仅允许 `[a-z0-9-]`
   - 不以连字符开头或结尾

3. **正则表达式**：`^[a-z0-9][a-z0-9-]{1,48}[a-z0-9]$`

### 5.3 完整文件路径示例

```
WisdomCore-Root/01_SaaS_Requirements/2026-Q2/PRD-2026-001-user-center-refactor.md
WisdomCore-Root/02_AI_Exploration/experiments/EXP-001-llm-code-review.md
WisdomCore-Root/02_AI_Exploration/decisions/ADR-001-vector-db-selection.md
WisdomCore-Root/03_Collaboration_Agreements/CA-2026-001-ai-code-review-sla.md
WisdomCore-Root/04_GTM_Knowledge/release-notes/RN-v2.1.0.md
WisdomCore-Root/04_GTM_Knowledge/battle-cards/BC-competitor-alpha.md
WisdomCore-Root/04_GTM_Knowledge/feature-briefs/FB-ai-code-review.md
```

---

## 6. ID 编号系统

### 6.1 编号格式

| 编号类型 | 格式 | 示例 |
|---------|------|------|
| PRD ID | `PRD-{YYYY}-{NNN}` | `PRD-2026-001` |
| Experiment ID | `EXP-{NNN}` | `EXP-001` |
| Agreement ID | `CA-{YYYY}-{NNN}` | `CA-2026-001` |
| ADR ID | `ADR-{NNN}` | `ADR-001` |
| GTM Release Note ID | `GTM-RN-{version}` | `GTM-RN-v2.1.0` |
| GTM Battle Card ID | `GTM-BC-{slug}` | `GTM-BC-competitor-alpha` |
| GTM Feature Brief ID | `GTM-FB-{slug}` | `GTM-FB-ai-code-review` |
| Bad Case ID | `BCS-{YYYY}-{NNN}` | `BCS-2026-001` |

### 6.2 编号分配规则

- 编号一旦分配，即使文档被删除也**不复用**
- 通过 SQLite 查询当前最大编号来分配下一个：
  ```sql
  SELECT MAX(CAST(SUBSTR(id, -3) AS INTEGER)) FROM documents WHERE type = 'Experiment'
  ```
- 归档文档保留原编号
