# 子 Skill-S 协作协议生成器 (WisdomCore-AgreementForge)

> 协作协议是混合研发体系的"接口契约"。Skill-S 的使命不仅是生成文档，
> 更重要的是迫使 SaaS 侧在提需求时就想清楚降级策略。
> 没有降级策略的 AI 需求等于没有安全网的高空走钢丝——
> Skill-S 将降级策略设为强制必填项，从源头杜绝"AI 挂了整个功能就挂了"的灾难。

---

## 1. Skill 概况

| 属性 | 内容 |
|------|------|
| 名称 | **WisdomCore-AgreementForge** |
| Skill ID | `wisdomcore-s` |
| 目标用户 | SaaS 产品经理、业务分析师、技术产品负责人 |
| 核心价值 | 将模糊的"我要智能搜索"转化为精确、可验收、含降级策略的协作协议文档 |
| 输入 | 自然语言描述的 AI 功能需求（对话式引导采集） |
| 输出 | 标准化的 `CA-YYYY-NNN-{slug}.md` 文件（含完整 YAML Frontmatter + Markdown Body） |
| 依赖 | 父 Skill 目录结构 (`03_Collaboration_Agreements/`)、SQLite 索引 (`wisdomcore.db`)、降级策略模板库 |
| 部署阶段 | Phase 1（首批上线，作为数据生产者） |

---

## 2. 六步引导式需求结构化 (Interactive Guided Flow)

Skill-S 采用六步引导式对话流程，将非结构化的 AI 功能需求逐步转化为结构化协议。每一步都有校验逻辑和智能提示。

### Step 1: 能力描述 (Capability Description)

> Agent 提示："你想要什么 AI 能力？请用一两句话描述。"

- 接受自由文本输入，无格式限制
- Agent 会对输入进行初步解析，提取关键实体：
  - 能力类型（分类/推荐/搜索/生成/检测/预测）
  - 领域关键词
  - 隐含的性能期望
- 若描述过于模糊（如"加个 AI 功能"），Agent 追问："能否具体说明 AI 在这个场景中要做什么？比如——根据用户历史行为推荐商品？自动检测异常交易？"
- 示例合格输入："在商品搜索结果中加入语义理解能力，让用户搜'适合跑步的鞋子'能返回相关运动鞋，而不是只匹配关键词"

### Step 2: 产品关联 (Product Context Binding)

> Agent 提示："这个能力用在哪个产品或功能模块上？"

- 自动扫描 `01_SaaS_Requirements/` 目录下已有 PRD 文档
- 展示匹配的 PRD 列表供用户选择（基于关键词相似度匹配）
- 若找到匹配 PRD，自动填入 `linked_prd` 字段
- 若未找到匹配，提示用户："未找到关联 PRD。建议先创建产品需求文档，或输入产品模块名称手动关联"
- 支持手动输入产品线/模块名称（不强制必须有 PRD）

### Step 3: 验收标准 (Acceptance Criteria)

> Agent 提示："对 AI 的最低要求是什么？我来帮你逐项确认。"

Agent 根据 Step 1 识别的能力类型，动态推荐适用的验收指标：

| 能力类型 | 默认推荐指标 | 说明 |
|---------|-------------|------|
| 搜索/推荐 | MRR@K, NDCG@K, Recall@K | K 值需用户指定（通常 K=5 或 K=10） |
| 分类/检测 | Precision, Recall, F1-Score | 需区分整体指标和各类别指标 |
| 生成 | BLEU, ROUGE, 人工评测通过率 | 生成类任务强烈建议包含人工评测 |
| 预测 | MAE, RMSE, MAPE | 需明确预测窗口 |
| 通用 | 准确率 (Accuracy) | 仅适用于均衡数据集 |

每项指标需填写：
- **指标名称**（如 Precision@10）
- **目标值**（如 >= 0.85）
- **测量方法**（如 "在标注测试集上计算" 或 "A/B 测试对比基线"）
- **测量频率**（如 "每次模型更新后" 或 "每周自动回归"）

延迟要求（必填）：P50 / P95 / P99 延迟目标

可用性要求（必填）：目标可用率、可接受的最大连续不可用时间

吞吐量要求（按需）：峰值 QPS 估算、并发用户数估算

### Step 4: 降级策略 (Fallback Strategy) -- 强制必填

> Agent 提示："如果 AI 达不到要求，你的 Plan B 是什么？这是必填项——没有降级策略的协议不允许生成。"

此步骤是 Skill-S 的核心差异化设计。Agent 引导用户完成三个层次的降级规划：

**(a) 降级方式选择**（可多选组合）：

| 降级方式 | 适用场景 | 实现复杂度 | 用户体验影响 |
|---------|---------|-----------|-------------|
| 规则引擎兜底 | AI 推理失败时用预定义规则替代 | 中 | 低（用户通常无感知） |
| 缓存兜底 | 返回最近一次有效的 AI 结果 | 低 | 中（数据可能不够新） |
| 人工介入 | 转人工处理，异步返回结果 | 低 | 高（响应延迟增加） |
| 功能降级 | 隐藏 AI 功能入口，显示基础版 | 低 | 中（功能缺失） |
| 默认值兜底 | 返回统计学上最优的默认值 | 低 | 中（千人一面） |
| 混合降级 | 按严重程度分级使用以上多种策略 | 高 | 低（精细化控制） |

**(b) 降级触发条件**：

- 置信度阈值：当 AI 返回置信度 < X 时触发（如 < 0.6）
- 延迟阈值：当响应时间 > Xms 时触发（如 > 500ms）
- 错误率阈值：当滑动窗口错误率 > X% 时触发（如 > 10%）
- 服务不可用：AI 服务无响应或返回错误码
- 熔断条件：连续 N 次失败后触发熔断（如连续 5 次）

**(c) 用户感知级别**：

| 级别 | 描述 | 示例 |
|------|------|------|
| 无感知 | 用户完全不知道正在使用降级方案 | 规则引擎结果与 AI 结果展示方式完全相同 |
| 轻度提示 | 小字提示或细微 UI 变化 | 搜索结果旁标注"基础模式" |
| 明确告知 | 明确告知用户当前为降级模式 | 弹窗："智能推荐暂时不可用，已切换为热门推荐" |

### Step 5: 风险评级 (Risk Assessment)

> Agent 提示："这个需求的风险等级是什么？"

| 等级 | 判断标准 | 审批要求 |
|------|---------|---------|
| High | 涉及营收核心功能、用户数据安全、监管合规 | 需技术总监 + 产品总监双签 |
| Medium | 影响用户体验但有可靠降级、非核心链路 | 需产品负责人审批 |
| Low | 辅助性功能、有完善降级策略、影响范围小 | 产品经理自主决定 |

Agent 会基于以下因素自动建议风险等级（用户可覆盖）：
- 关联功能的用户覆盖面
- 降级策略的完善程度
- 是否涉及数据隐私
- 是否在关键业务链路上

### Step 6: 时间线 (Timeline)

> Agent 提示："预计什么时候需要这个能力？"

- 目标上线日期
- 是否有硬性 Deadline（如客户承诺、合同约束）
- 期望的评审节奏（每周 / 双周 / 每月）
- 首次评审日期（自动设为创建后 2 周）

---

## 3. 降级策略模板库 (Fallback Strategy Templates)

Skill-S 内置六种经过验证的降级策略模板，用户可直接选用或组合定制。

### 模板 1: 规则引擎兜底

```yaml
fallback_name: rule_engine_fallback
description: 用预定义业务规则替代 AI 推理结果
trigger:
  - condition: confidence < 0.6
  - condition: response_time > 500ms
  - condition: service_unavailable
implementation:
  engine: rule_based_matching
  rules_source: "config/fallback_rules/{feature_name}.yaml"
  warm_up: 预加载规则集到内存
user_perception: 无感知
recovery:
  auto_recovery: true
  health_check_interval: 30s
  recovery_condition: "连续 3 次 AI 服务正常响应"
```

### 模板 2: 缓存兜底

```yaml
fallback_name: cache_fallback
description: 返回最近一次有效的 AI 推理结果
trigger:
  - condition: service_unavailable
  - condition: response_time > 1000ms
implementation:
  cache_backend: Redis / 本地内存
  cache_ttl: 3600s
  cache_key_strategy: "user_id + query_hash"
  stale_threshold: 7200s  # 超过此时间的缓存标记为 stale
user_perception: 轻度提示（标注"结果可能非最新"）
limitations:
  - 新用户无缓存可用，需配合其他降级策略
  - 缓存数据不反映最新变化
```

### 模板 3: 人工介入

```yaml
fallback_name: human_in_the_loop
description: 将请求转入人工处理队列，异步返回结果
trigger:
  - condition: confidence < 0.4  # 极低置信度
  - condition: high_risk_decision  # 高风险决策场景
implementation:
  queue: "human_review_queue"
  sla: 4h  # 人工处理 SLA
  escalation: "超过 SLA 自动升级至主管"
user_perception: 明确告知（"您的请求正在人工复核中，预计 X 小时内返回结果"）
limitations:
  - 不适用于实时性要求高的场景
  - 需要人工团队支持
```

### 模板 4: 功能降级

```yaml
fallback_name: feature_degradation
description: 隐藏 AI 功能入口，展示基础版本功能
trigger:
  - condition: circuit_breaker_open
  - condition: error_rate > 20%
implementation:
  feature_flag: "ai_{feature_name}_enabled"
  degraded_ui: "显示基础版搜索/列表/排序"
  switch_mechanism: "Feature Flag 服务 / 本地配置"
user_perception: 中度（功能入口消失或变为基础版）
recovery:
  manual_recovery: true  # 需人工确认后恢复
```

### 模板 5: 默认值兜底

```yaml
fallback_name: default_value_fallback
description: 返回基于历史数据统计的最优默认值
trigger:
  - condition: service_unavailable
  - condition: confidence < 0.3
implementation:
  default_source: "统计分析离线计算"
  update_frequency: daily
  strategy: "返回全局热门 / 类别热门 / 随机采样"
user_perception: 无感知（但结果千人一面）
limitations:
  - 完全丧失个性化能力
  - 需定期更新默认值集合
```

### 模板 6: 混合降级（推荐）

```yaml
fallback_name: hybrid_fallback
description: 按严重程度分级使用多种降级策略
levels:
  - level: L1_soft_degradation
    trigger: "confidence < 0.6 OR response_time > 300ms"
    strategy: rule_engine_fallback
    user_perception: 无感知
  - level: L2_moderate_degradation
    trigger: "error_rate > 10% OR response_time > 1000ms"
    strategy: cache_fallback + default_value_fallback
    user_perception: 轻度提示
  - level: L3_severe_degradation
    trigger: "circuit_breaker_open OR service_unavailable > 5min"
    strategy: feature_degradation
    user_perception: 明确告知
  - level: L4_full_fallback
    trigger: "连续 30 分钟不可用"
    strategy: human_in_the_loop + feature_degradation
    user_perception: 明确告知 + 人工接管
```

---

## 4. 自动依赖检查 (Dependency Pre-Check)

在生成协议文档之前，Skill-S 执行四项自动化检查，确保协议不会成为"孤岛"。

### 检查项 1: PRD 关联性

扫描 `01_SaaS_Requirements/` 目录下的 PRD 文档，通过关键词模糊匹配查找与当前能力描述相关的 PRD。若未找到匹配，提示用户先创建产品需求文档。

### 检查项 2: 已有实验

查询 SQLite 中状态不为 Archived/Superseded 的实验记录，通过关键词模糊匹配查找相关实验。若未找到，提示 AI 团队可能尚未开始探索该方向；若找到，展示相关实验列表。

### 检查项 3: 重复协议检测

查询 SQLite 中状态为 Active 或 Draft 的已有协议，通过语义相似度（threshold=0.8）检测是否存在同类协议。若存在，提示用户确认是补充还是替代。

### 检查项 4: Owner 有效性

从 `.meta/team-roster.yaml` 加载团队配置，校验填写的 SaaS Owner 和 AI Owner 是否在团队成员列表中。

所有检查结果以汇总形式展示给用户，用户确认后才进入生成阶段。

---

## 5. 协作协议文档生成规格 (CA Document Generation Spec)

### 5.1 ID 生成规则

- 格式：`CA-YYYY-NNN`
- YYYY：当前年份
- NNN：当年内的三位序号，从 001 开始自增
- 查询 SQLite 获取当年最大序号并 +1

### 5.2 文件写入流程

1. 渲染 YAML Frontmatter + Markdown Body
2. 写入 `03_Collaboration_Agreements/CA-YYYY-NNN-{slug}.md`
3. 执行 `INSERT INTO documents (...)` 更新 SQLite 索引
4. 若配置了 AI Owner 通知，输出提示："已创建协议 CA-YYYY-NNN，请通知 AI Owner {name} 查看"

### 5.3 与其他 Skill 的交互

| 交互方向 | 交互内容 | 触发时机 |
|---------|---------|---------|
| S -> 父 Skill | 写入 CA 文件到 `03_Collaboration_Agreements/` | 文档生成完成时 |
| S -> 父 Skill | 执行 SQLite `INSERT/UPDATE` 操作 | 文档生成或更新时 |
| S <- 父 Skill | 读取目录结构和配置信息 | Skill 启动时 |
| S <- 父 Skill | 调用 `assign_id()` 获取新文档 ID | 文档创建时 |
| S -> Skill-A | 新协议创建后，通知 AI Owner 查看验收标准并评估可行性 | 协议创建后 |
| S <- Skill-A | 关联实验进展可在下次打开协议时显示 | 协议查看时 |
| S -> Skill-H | 新建/更新的协议自动出现在全景仪表盘（通过 SQLite 索引实时反映） | 文档变更时 |
| S -> Skill-G | 当协议 Active 且关联实验达标时，Skill-G 可读取协议内容生成 GTM 素材 | 协议状态变更时 |

---

## 6. 完整 CA 文档生成示例

以下是 Skill-S 生成的一份完整协作协议文档范例：

```yaml
---
id: CA-2026-001
title: 商品搜索语义理解能力
type: CollaborationAgreement
status: Draft
created_at: 2026-04-23
updated_at: 2026-04-23
saas_owner: 张明（产品经理）
ai_owner: 李华（算法工程师）
team: default
product_line: default
related_requirements: [PRD-2026-012]
related_experiments: []
risk_level: Medium
risk_reason: "搜索是核心功能但降级策略完善，影响可控"
acceptance_criteria:
  - metric: MRR@10
    operator: ">="
    threshold: 0.80
  - metric: P99_latency_ms
    operator: "<="
    threshold: 300
fallback_strategy: "混合降级方案：L1置信度<0.6时规则引擎兜底；L2错误率>10%时缓存+关键词；L3熔断时回退关键词搜索"
review_cadence_days: 14
target_release: v2.1.0
next_review_date: 2026-05-07
tags: [搜索, NLP, 语义理解, 核心功能]
---
```

```markdown
# CA-2026-001: 商品搜索语义理解能力

## 1. 能力概述

在现有关键词搜索基础上增加语义理解能力，使搜索系统能够理解用户的意图
而非仅匹配关键词。

用户场景示例：
- 用户搜"适合跑步的鞋子" -> 应返回"跑鞋"分类下的商品，
  而非仅包含"跑步"和"鞋子"关键词的商品
- 用户搜"便宜的生日礼物" -> 应理解"便宜"为价格筛选 + "生日礼物"为场景需求
- 用户搜"不含坚果的零食" -> 应理解否定语义，排除含坚果的商品

## 2. 接口定义

### 输入
{
  "query": "string, 用户搜索文本, 最大 256 字符",
  "user_id": "string, optional, 用于个性化",
  "context": {
    "category": "string, optional, 当前浏览品类",
    "price_range": "object, optional, {min, max}",
    "page": "int, 分页参数",
    "page_size": "int, 每页条数, 默认 20"
  }
}

### 输出
{
  "results": [
    {
      "product_id": "string",
      "score": "float, 0-1, 综合相关性得分",
      "match_type": "semantic | keyword | hybrid",
      "explanation": "string, optional, 匹配原因（调试用）"
    }
  ],
  "query_understanding": {
    "intent": "string, 识别到的搜索意图",
    "entities": ["抽取的实体列表"],
    "rewritten_query": "string, 改写后的查询"
  },
  "metadata": {
    "model_version": "string",
    "confidence": "float, 0-1",
    "latency_ms": "int",
    "fallback_used": "boolean"
  }
}

## 3. 验收标准

| 指标 | 目标值 | 测量方法 | 测量频率 |
|------|--------|---------|---------|
| MRR@10 | >= 0.80 | 标注测试集（500 条） | 每次模型更新 |
| NDCG@10 | >= 0.75 | 标注测试集（500 条） | 每次模型更新 |
| 语义理解准确率 | >= 0.85 | 意图分类 + 实体抽取准确率 | 每周回归 |
| P50 延迟 | <= 50ms | 线上监控 | 实时 |
| P95 延迟 | <= 150ms | 线上监控 | 实时 |
| P99 延迟 | <= 300ms | 线上监控 | 实时 |
| 可用性 | >= 99.9% | 拨测 + 线上监控 | 实时 |

## 4. 降级策略

采用混合降级方案（Hybrid Fallback）：

L1 软降级（置信度 < 0.6 或延迟 > 150ms）：
- 使用同义词扩展 + TF-IDF 规则引擎替代语义搜索
- 用户无感知

L2 中度降级（错误率 > 10% 或延迟 > 500ms）：
- 返回缓存的热门搜索结果 + 关键词匹配
- 搜索结果旁小字标注"基础搜索模式"

L3 严重降级（服务熔断）：
- 完全回退到关键词搜索
- 隐藏"智能搜索"标签
- 搜索框提示语变为"搜索商品名称"

降级恢复条件：连续 5 次正常响应且延迟 < 100ms，自动恢复上一级别。

## 5. 数据依赖

| 数据 | 来源 | 更新频率 | 数据量预估 |
|------|------|---------|-----------|
| 商品标题与描述 | 商品数据库 | 实时同步 | ~100 万条 |
| 搜索日志 | 搜索服务 | T+1 离线 | ~50 万条/天 |
| 用户点击行为 | 埋点系统 | T+1 离线 | ~200 万条/天 |
| 人工标注数据 | 标注平台 | 按批次 | 500 条测试集 |

## 6. 监控与告警

| 监控项 | 告警阈值 | 告警方式 | 响应要求 |
|-------|---------|---------|---------|
| MRR@10 周均值 | < 0.75 | 企业微信通知 | 24h 内排查 |
| P99 延迟 | > 500ms 持续 5 分钟 | 短信 + 企业微信 | 30 分钟内响应 |
| 可用率 | < 99.5% (1h 窗口) | 电话 + 短信 | 15 分钟内响应 |
| 降级触发率 | > 5% (1h 窗口) | 企业微信通知 | 4h 内排查 |

## 7. 评审计划

- 首次评审：2026-05-07（创建后 2 周）
- 评审节奏：每两周一次
- 评审内容：指标达标情况、降级触发频率、Bad Case 分析、下阶段计划
- 参与人员：SaaS Owner（张明）、AI Owner（李华）、技术主管
```
