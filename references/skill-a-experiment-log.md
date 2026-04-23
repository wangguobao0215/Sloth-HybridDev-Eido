# Skill-A: AI 实验记录员 (WisdomCore-ExperimentLog)

> 本文档为 Skill-A 的完整参考手册，涵盖实验报告模板、一键归档、知识链接、
> 谱系追踪、Bad Case 管理及 ADR 触发机制。

---

## 1. Skill 概况

| 属性 | 内容 |
|------|------|
| 名称 | **WisdomCore-ExperimentLog** |
| Skill ID | `wisdomcore-a` |
| 目标用户 | AI/ML 工程师、算法研究员、数据科学家 |
| 核心价值 | 让每一次实验都有迹可循，把"我试过了但忘记参数了"变成结构化资产 |
| 输入 | 实验数据（手动输入 / 从 Notebook 日志提取）、Bad Case 数据 |
| 输出 | 标准化 `EXP-YYYY-NNN-{slug}.md` 文件、Bad Case Registry 记录、可选 ADR 触发 |
| 依赖 | 父 Skill 目录结构 (`02_AI_Exploration/`)、SQLite 索引、Bad Case Registry 表 |
| 部署阶段 | Phase 1（首批上线，作为数据生产者） |

**设计哲学**：AI 研发的核心困境是实验过程不可复现、结论难以传承。Skill-A 的使命
是将每一次实验——无论成功还是失败——都转化为团队的可检索知识资产。特别地，
Bad Case 被视为**一等公民**（First-Class Citizen），因为 Bad Case 往往比成功案例
更有价值。

---

## 2. 实验报告模板（Experiment Report Template）

模板覆盖实验全生命周期，以下是完整结构及字段说明。

### 2.1 Frontmatter

```yaml
---
id: EXP-YYYY-NNN
title: "{实验标题，简洁概括实验目的}"
type: Experiment
status: Running | Completed | Failed | Superseded | Archived
created_at: YYYY-MM-DD
updated_at: YYYY-MM-DD
owner: "{实验负责人}"
linked_agreements: [CA-YYYY-NNN]
predecessor: EXP-YYYY-NNN | null   # 前序实验（谱系追踪）
successor: EXP-YYYY-NNN | null     # 后续实验
tags: [模型类型, 任务类型, 数据领域]
key_metrics:
  metric_name_1: value_1
  metric_name_2: value_2
conclusion: Accept | Reject | Inconclusive
---
```

### 2.2 正文章节

#### 第 1 章：实验背景

- **1.1 问题定义** — 用 1-3 段话描述当前面临的问题来源（用户反馈/线上数据/协议要求）
- **1.2 业务上下文** — 关联协作协议 CA-ID、当前基线水平、业务影响

#### 第 2 章：假设

- **核心假设** — 一句话描述假设及理由
- **支撑依据** — 论文/技术博客引用、行业 Benchmark、内部前序实验结论
- **风险因素** — 可能导致假设失败的因素

#### 第 3 章：实验设计

- **3.1 数据集** — 训练集/验证集/测试集（描述、规模、来源、时间范围、标注方式）
  - 数据质量说明：标注一致性（Inter-Annotator Agreement）、已知偏差、清洗策略
- **3.2 模型/算法** — 基线 Baseline vs 候选 Candidate（名称、版本、关键特征、核心改动）
- **3.3 评估指标** — Primary / Secondary / Efficiency 分级，含目标值与优先级

#### 第 4 章：实验过程

- **4.1 环境配置** — 硬件、框架版本、关键依赖、Random Seed、Docker Image
- **4.2 超参数** — 基线值 vs 实验值及调整理由
- **4.3 训练日志** — 开始/结束时间、总耗时、最佳 Epoch、收敛情况、异常事件

#### 第 5 章：实验结果

- **5.1 定量结果** — 基线 vs 实验组 vs 目标值 vs 提升幅度 vs 达标判定
- **5.2 定性分析** — 表现优秀场景、表现不佳场景、意外发现
- **5.3 Bad Case 分析（核心章节）** — 至少 5 个典型 Bad Case，含 Case ID / 输入 / 期望 / 实际 / 失败分类 / 严重程度 / 改进方向
- **5.4 统计显著性检验** — 方法、p-value、置信区间、结论

#### 第 6 章：结论与下一步

- **6.1 实验结论** — Accept / Reject / Inconclusive + 核心发现
- **6.2 与协作协议对比** — CA 指标 vs 实验结果 vs 达标判定
- **6.3 下一步行动** — 具体行动 + 时间 + 负责人

#### 第 7 章：决策建议

- 是否需要创建 ADR？若是，建议决策主题

---

## 3. 一键归档流程（One-Click Archive Flow）

### 3.1 输入方式

**方式 1：对话式手动输入**
- Agent 按模板章节逐步引导填写
- 支持跳过非必填字段，但 Bad Case 章节强制至少 3 条

**方式 2：从实验日志/笔记提取**
- 用户粘贴实验日志或 Notebook 内容
- Agent 自动解析关键信息：
  - 超参数：正则匹配 `key=value` 模式
  - 指标数值：匹配 `metric: value` 模式
  - 训练时间：匹配时间戳模式
- 提取结果让用户确认和补充

### 3.2 归档执行流程

```
用户输入 -> 信息解析 -> 模板填充 -> ID 生成 (EXP-YYYY-NNN)
  -> Frontmatter 渲染 -> Markdown 渲染
  -> 写入 02_AI_Exploration/experiments/EXP-YYYY-NNN-{slug}.md
  -> SQLite INSERT (documents 表)
  -> SQLite INSERT (bad_cases 表，逐条写入)
  -> 关联协议扫描 -> 输出关联提示
```

### 3.3 SQLite 操作

```sql
-- 写入文档索引
INSERT INTO documents (id, title, type, status, created_at, updated_at, owner, tags)
VALUES ('EXP-2026-005', '语义搜索模型 v2 实验', 'Experiment', 'Completed',
        '2026-04-23', '2026-04-23', '李华', '["搜索","NLP","BERT"]');

-- 写入关联关系
INSERT INTO dependencies (source_id, target_id, relation_type)
VALUES ('EXP-2026-005', 'CA-2026-001', 'validates');

-- 写入 Bad Cases
INSERT INTO bad_cases (case_id, title, severity, category,
                       description, root_cause, related_experiment_id,
                       reported_by, status)
VALUES ('BCS-2026-005-001', '否定语义理解失败-坚果零食', 'Major', 'Accuracy',
        '输入"不含坚果的零食"，期望排除坚果类商品，实际返回坚果零食',
        'model_limitation', 'EXP-2026-005',
        '李华', 'Open');
```

> **sqlite-utils 简化提示**：上述 SQL 操作可通过 sqlite-utils CLI 一行完成：
> ```bash
> # 查询某实验的全部 Bad Case
> sqlite-utils query .wisdomcore.db "SELECT * FROM bad_cases WHERE experiment_id='EXP-2026-005'" --json
>
> # 插入新实验记录
> echo '{"id":"EXP-2026-006","type":"Experiment","title":"..."}' | sqlite-utils insert .wisdomcore.db documents -
> ```

---

## 4. 知识链接引擎（Knowledge Linking Engine）

实验报告归档时，Skill-A 自动执行知识链接扫描。

### 4.1 扫描逻辑

1. 查询所有 `status='Active'` 的协作协议
2. 解析每个协议的验收标准（Acceptance Criteria）
3. 将实验的 `key_metrics` 与协议验收标准逐一比对
4. 输出匹配结果：`meets`（达标）或 `below`（未达标）

### 4.2 输出示例

```
知识链接扫描结果：

1. EXP-2026-005 的 MRR@10=0.82 -> 达标 CA-2026-001 的验收标准（要求 >= 0.80）
   实验结果满足协议要求

2. EXP-2026-005 的 P99 Latency=280ms -> 达标 CA-2026-001 的延迟要求（要求 <= 300ms）
   余量仅 20ms，建议关注后续优化

3. 未找到与 CA-2026-003（智能推荐）的直接关联
```

### 4.3 违约预警

如果实验结果低于协议要求，Agent 主动告警：
- 与 SaaS Owner 沟通指标差距
- 考虑调整协议目标值或追加实验
- 检查降级策略是否需要更新

---

## 5. 实验谱系追踪（Experiment Lineage Tracking）

AI 实验通常呈链式演进关系，Skill-A 通过 `predecessor`/`successor` 字段维护谱系。

### 5.1 谱系维护规则

1. 创建新实验时，Agent 询问是否基于之前的某个实验
2. 若是，自动填入 `predecessor` 字段，并更新前序实验的 `successor` 字段
3. 实验被标记为 Superseded 时，自动检查并更新整条链

### 5.2 谱系查询 SQL

```sql
WITH RECURSIVE lineage AS (
    SELECT d.id, d.title, d.status, 1 as depth
    FROM documents d WHERE d.id = 'EXP-2026-005'
    UNION ALL
    SELECT d.id, d.title, d.status, l.depth + 1
    FROM lineage l
    JOIN dependencies dep ON dep.source_id = l.id AND dep.relation_type = 'supersedes'
    JOIN documents d ON d.id = dep.target_id
)
SELECT * FROM lineage ORDER BY depth DESC;
```

### 5.3 谱系可视化输出

```
EXP-2026-001 (Superseded, MRR=0.65)
  -> EXP-2026-003 (Superseded, MRR=0.72)
       -> EXP-2026-005 (Completed, MRR=0.82) <- 当前最优
            -> EXP-2026-008 (Running) <- 进行中
```

---

## 6. Bad Case 管理（Bad Case Registry）

Bad Case 在 WisdomCore 体系中被视为**一等公民**，Skill-A 维护跨实验的持久化 Registry。

### 6.1 数据模型

```sql
CREATE TABLE bad_cases (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    case_id         TEXT NOT NULL UNIQUE,       -- BCS-YYYY-NNN
    title           TEXT NOT NULL,
    severity        TEXT NOT NULL,              -- Critical | Major | Minor
    category        TEXT NOT NULL,              -- Accuracy | Latency | Safety | UX | Integration
    description     TEXT NOT NULL,
    root_cause      TEXT,
    related_experiment_id TEXT,
    related_agreement_id  TEXT,
    reported_by     TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'Open', -- Open | Investigating | Resolved | WontFix
    resolution      TEXT,
    created_at      TEXT NOT NULL DEFAULT (datetime('now')),
    resolved_at     TEXT,
    team            TEXT DEFAULT 'default',
    product_line    TEXT DEFAULT 'default',
    CONSTRAINT valid_severity CHECK (severity IN ('Critical', 'Major', 'Minor')),
    CONSTRAINT valid_bc_status CHECK (
        status IN ('Open', 'Investigating', 'Resolved', 'WontFix')
    )
);
```

### 6.2 失败分类体系（Failure Taxonomy）

| 分类 | 英文标识 | 描述 | 典型处理方式 |
|------|---------|------|-------------|
| 数据质量问题 | `data_quality` | 训练数据标注错误、数据不一致 | 修正数据、重新标注 |
| 模型能力不足 | `model_limitation` | 模型架构或容量无法处理此类问题 | 更换模型、增加特征 |
| 边缘场景 | `edge_case` | 罕见但合理的输入场景 | 增加训练样本、添加规则 |
| 标注错误 | `annotation_error` | 测试集标注本身有误 | 修正标注 |
| 基础设施问题 | `infrastructure` | 超时、OOM 等非模型问题 | 优化部署、扩容 |
| 能力边界 | `out_of_scope` | 超出当前系统设计范围 | 评估是否需要扩展能力范围 |

### 6.3 跨实验追踪

- 每个新实验完成后，自动检查是否解决了 Registry 中任何 `open` 状态的 Bad Case
- 解决逻辑：对 `open` Bad Case 重新用新模型预测，若结果正确则标记为 `resolved`
- 输出示例："本次实验解决了 3 个历史 Bad Case：BCS-001, BCS-004, BCS-007；仍有 12 个未解决"

### 6.4 与生产事故关联

- 线上 AI 能力不足导致的事故可通过 `production_incident_id` 关联
- 关联后自动升级为 `high` 严重程度
- 在相关实验 Frontmatter 中自动添加 tag: `production_incident`

---

## 7. ADR 触发机制（Architecture Decision Record Trigger）

当实验结论涉及重要技术决策时，Skill-A 自动检测并建议创建 ADR。

### 7.1 触发条件

1. 实验结论为 `Reject`（原方案被否定，需要方向性调整）
2. 实验结论为 `Accept` 且包含重大架构变更（如更换基础模型框架）
3. 连续 3 个实验在同一方向上 `Reject`（暗示需要根本性转变）
4. 实验引入的延迟或资源消耗超过原方案 200%
5. 用户手动触发

### 7.2 触发流程

```
实验归档完成
  -> 检测触发条件
  -> 若命中：
     Agent 提示："这个实验的结论可能构成重要的技术决策，是否创建 ADR？"
  -> 用户确认后：
     -> 预填充 ADR 模板：
       - 决策背景：从实验背景章节提取
       - 被评估的选项：从基线和候选方案提取
       - 决策依据：从实验结果章节提取
       - 相关实验：自动填入当前实验 ID
     -> 写入 02_AI_Exploration/decisions/ADR-YYYY-NNN.md
     -> 更新 SQLite 索引
```

---

## 8. Agent 交互记忆 (memweave 架构启发)

Skill-A 支持将 AI Agent 的交互上下文持久化到 `.meta/memory/` 目录，采用 memweave 的 "Markdown 日记 + SQLite 索引" 模式。

### 8.1 记忆文件格式

```markdown
---
date: 2026-04-23
session_type: experiment_review
documents_touched: [EXP-2026-005, CA-2026-001]
key_decisions:
  - "决定采用 ONNX 格式部署语义搜索模型"
  - "CA-2026-001 需要更新 P99 延迟指标从 500ms 到 300ms"
follow_up_actions:
  - "创建 ADR-004 记录 ONNX vs TensorRT 选型决策"
---

## 会话摘要

用户要求分析 EXP-2026-005（语义搜索 A/B 测试）的结果...
```

### 8.2 记忆索引

Agent 交互日志的 Frontmatter 同样通过 sqlite-utils 索引到 `.wisdomcore.db` 的 `agent_sessions` 表（可选扩展），支持按日期、涉及文档、决策关键词检索历史上下文。

> **设计原则**：记忆层是可选增强，不影响核心功能。不启用时 Skill-A 完全正常工作。

---

## 9. 与其他 Skill 的交互

| 交互方向 | 交互内容 | 触发时机 | 数据流 |
|---------|---------|---------|--------|
| A -> 父 Skill | 写入实验文件、更新 SQLite | 实验归档时 | EXP 文件 + SQL INSERT |
| A -> Skill-S | 反向通知 SaaS Owner 实验进展 | 实验达标/不达标时 | 输出关联提示信息 |
| A -> Skill-H | 实验数据自动进入仪表盘 | 实时（通过 SQLite） | SQLite 查询 |
| A <- Skill-S | 获取协议验收标准用于比对 | 知识链接扫描时 | 读取 CA 文件 |
| A -> Skill-G | Bad Case 数据供 GTM 生成"已知限制" | Skill-G 主动拉取 | SQLite 查询 bad_cases 表 |
| A <- 用户 | 实验数据输入 | 用户主动触发 | 对话 / 粘贴内容 |
