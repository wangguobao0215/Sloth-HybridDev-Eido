# 决策记录（ADR）框架 (Architecture Decision Records Framework)

> Architecture Decision Records (ADR) 是 WisdomCore 中的"组织记忆"工具。
> 在 AI + SaaS 混合研发中，技术决策的语境往往比结论本身更有长期价值。
> 本参考文档定义 ADR 的完整框架，包括为什么需要、模板规范、与实验报告的关联，
> 以及 ADR 自身的治理规则。

---

## 1. 为什么需要 ADR

### 1.1 AI 技术选型的特殊挑战

在传统 SaaS 开发中，技术选型决策（如选择 React vs Vue）通常有明确的社区共识和长期稳定的评估标准。但在 AI 研发中，技术选型面临以下独特挑战：

1. **技术迭代速度极快**：6个月前的最优选择可能已经过时（如 GPT-3.5 -> GPT-4 -> GPT-4o）
2. **约束条件高度特定**：同样是"文本分类"任务，在不同的延迟/成本/数据量约束下最优解完全不同
3. **评估标准多维交叉**：不仅要看准确率，还要看延迟、成本、可解释性、数据隐私等
4. **实验结果不可直接复用**：在 A 场景下的实验结论不能直接搬到 B 场景

### 1.2 没有 ADR 时的典型问题

- **重复踩坑**："我们去年试过 GPT-2 做搜索排序，效果不好" -> "效果不好是因为什么？当时的约束条件是什么？" -> 无人记得
- **决策幽灵**："为什么我们用 BERT 不用 GPT？" -> "好像是因为延迟问题" -> "到底是什么级别的延迟要求？" -> 无法考证
- **新人困境**：新加入团队的工程师面对现有架构，无法理解历史决策的逻辑，要么盲目接受要么盲目否定
- **循环讨论**：同一个技术选型问题在不同会议上反复讨论，每次都像是第一次

### 1.3 ADR 的核心价值

**"我们选了 BERT 不选 GPT"——这句话三个月后就没用了。**

**"在 P99 <= 200ms 的延迟约束 + 10万条标注数据 + 2xA100 GPU 的硬件条件下，BERT 微调在性价比上优于 GPT-2 少样本学习"——这句话永远有用。**

ADR 的价值在于捕获决策的 **完整语境（Context）**，而不仅仅是结论。语境包括：
- 当时面临的约束条件（技术、资源、时间）
- 评估了哪些备选方案以及各自的优劣
- 最终选择的理由
- 预期的正面和负面后果
- 何时需要复审这个决策

---

## 2. 完整 ADR 模板

### 2.1 Frontmatter 规范

```yaml
---
# === 基本信息 ===
id: ADR-001                                    # 唯一标识，格式: ADR-{NNN}
type: ADR                                      # 文档类型，固定为 ADR
title: "智能搜索模型选型：选择BERT微调而非GPT-2"  # 决策标题，简洁明了

# === 状态信息 ===
status: Proposed                               # Proposed | Accepted | Rejected | Deprecated | Superseded
decision_date: 2026-04-20                      # 做出决策的日期
superseded_by: null                            # 如被替代，填入新 ADR 的 ID

# === 归属信息 ===
author: "LiSi"                                 # 决策提出者/记录者
team: "ai-search"                              # 所属团队
product_line: "main-app"                       # 所属产品线

# === 关联信息 ===
related_agreements: ["CA-2026-001"]            # 关联的协作协议
related_experiments: ["EXP-001", "EXP-002"]    # 关联的实验报告（决策依据）
related_prds: ["PRD-2026-001"]                 # 关联的 PRD
related_adrs: []                               # 关联的其他 ADR（相关决策）

# === 分类标签 ===
tags: ["Search", "ModelSelection", "BERT", "NLP"]
decision_category: "ModelSelection"            # 决策大类

# === 治理信息 ===
review_cadence_days: 90                        # 复审周期：90天
created_at: 2026-04-20
updated_at: 2026-04-20
next_review_date: 2026-07-20                   # 下次复审日期
---
```

### 2.2 正文模板

ADR 正文包含七个标准章节：

#### 背景 (Context)

描述促使做出这个决策的背景和问题。不要假设读者知道项目的上下文。应交代清楚：哪个项目、什么需求、为什么需要做出这个决策。

#### 约束条件 (Constraints)

列出影响决策的所有硬性和软性约束：

- **硬性约束（不可违反）**：延迟上限、数据隐私合规、可用性 SLA 等
- **软性约束（尽量满足）**：硬件资源、数据量、团队能力、时间窗口等

#### 备选方案 (Options Considered)

逐一列出评估过的方案，每个方案应包含：

- **描述**：方案的技术路线概述
- **优势**：该方案的核心优点
- **劣势**：该方案的核心缺点
- **预估成本**：开发成本、推理/运维成本、维护成本

#### 决策 (Decision)

一句话明确最终选择的方案。

#### 理由 (Rationale)

按优先级排序说明选择该方案的理由，需要与约束条件和备选方案分析对应。建议采用编号列表，每条理由清晰对应一个关键判断依据。

#### 预期后果 (Consequences)

分三个维度描述：

- **正面后果**：选择该方案带来的预期收益
- **负面后果**：选择该方案必须接受的已知代价
- **风险**：未来可能需要重新评估的条件变化

#### 复审计划 (Review Plan)

明确下次复审日期和届时需要评估的具体内容，包括：

- 生产环境实际表现是否符合预期
- 约束条件是否发生变化
- 是否有新的技术/模型可以替代
- 团队能力是否已支持更复杂的方案

---

## 3. ADR 与实验报告的关联规则

### 3.1 关联原则

ADR 与实验报告（EXP）之间存在紧密的因果关系：

1. **实验报告是 ADR 的决策依据**
   - 每个 ADR 的 `related_experiments` 字段列出作为决策依据的实验
   - 在 ADR 正文的"备选方案"部分，应引用具体的实验结果数据
   - 示例："根据 EXP-001 的实验结果，BERT 微调在测试集上 accuracy 达到 83%"

2. **ADR 的决策可能触发新的实验**
   - 当 ADR 指出"需要进一步验证"时，应创建新的实验计划
   - 新实验的 `related_adrs` 字段应回指该 ADR

3. **ADR 复审可能依赖新的实验结果**
   - 复审时如果发现新模型/新方法，应先设计实验验证再决策
   - 避免"拍脑袋"式的决策更新

### 3.2 关联完整性校验

系统自动校验 ADR 与 EXP 的关联完整性，包含三项检查：

1. **实验存在性校验**：ADR `related_experiments` 中引用的实验 ID 是否存在于文档库中
2. **实验完成度校验**：ADR 引用的实验状态是否为 Completed 或 Concluded（基于未完成实验的决策会触发警告）
3. **引用一致性校验**：ADR 正文中提到的实验 ID（如 "EXP-001"）是否均在 Frontmatter `related_experiments` 中声明

### 3.3 决策链可视化

在 Skill-H 仪表盘中可以看到完整的"决策链"——从实验到决策再到协议的因果路径：

```
EXP-001 (BERT微调实验, accuracy=83%)
  |
  +-- validates --> CA-2026-001 (智能搜索协议, threshold=80%)
  |
  +-- informs  --> ADR-001 (模型选型: 选择BERT)
                     |
EXP-002 (GPT-2实验, latency=800ms)  --> (作为反面证据)
                     |
                     +-- informs --> CA-2026-001 (延迟标准设定依据)
```

### 3.4 生命周期独立性

重要原则：**ADR 被 Superseded 时，关联的实验报告不受影响。**

- ADR-001 标记为 Superseded（被 ADR-002 替代），EXP-001 仍保持 Completed 状态
- EXP-001 的实验数据和结论仍然有效，可被新的 ADR-002 引用
- 在决策链可视化中，Superseded 的 ADR 以灰色虚线显示

---

## 4. ADR 治理 (Governance)

### 4.1 什么时候需要写 ADR

**必须写 ADR 的决策（Mandatory）**：

| 决策大类 | `decision_category` 值 | 示例 |
|---------|----------------------|------|
| 模型选型 | `ModelSelection` | 选择 BERT vs GPT vs 传统方法 |
| 架构设计 | `ArchitectureDesign` | 决定 AI 服务的部署架构（微服务 vs 嵌入式） |
| 数据策略 | `DataStrategy` | 选择数据标注方案、数据存储方案 |
| 降级策略 | `FallbackStrategy` | 确定 AI 能力不可用时的降级方案 |
| 基础设施 | `Infrastructure` | GPU 集群选型、推理框架选择 |
| 协议重大修订 | `AgreementMajorRevision` | 涉及验收标准大幅调整的决策 |

**建议写 ADR 的决策（Recommended）**：

| 决策大类 | `decision_category` 值 | 示例 |
|---------|----------------------|------|
| 工具选型 | `ToolSelection` | 选择实验管理平台（MLflow vs W&B） |
| 流程设计 | `ProcessDesign` | 确定 AI 团队的 Sprint 节奏 |
| 评估方法 | `EvaluationMethod` | 选择离线评估 vs 在线 A/B 测试 |

**不需要写 ADR 的决策**：日常代码实现细节、已有明确团队规范的选择、临时的调试配置。

### 4.2 审批流程

```
作者(Author)撰写 ADR
  +-> 创建 PR，标签: adr
  +-> 自动添加 Reviewer:
      +-> 所属团队的 Tech Lead（必选）
      +-> 关联协作协议的对端 Owner（如有关联，可选）
  +-> Tech Lead 评审（<=5个工作日）
      +-> 审批通过
      |    +-> ADR status 设为 Accepted
      |    +-> 合并到 main
      |    +-> 同步到 SQLite
      |    +-> 通知关联文档 Owner
      +-> 要求修改
      |    +-> 作者修改后重新提交
      +-> 否决
           +-> ADR status 设为 Rejected（不合并）
           +-> 记录否决理由
           +-> 作者可以提出申诉（升级至研发总监）
```

### 4.3 不可删除原则 (Immutable Records)

**ADR 是组织记忆的一部分，一旦创建就不可物理删除。** 理由如下：

1. **错误的决策也是有价值的记录**：知道"我们试过方案X但失败了"比不知道更有价值；未来有人提议方案X时，可以快速定位历史记录。

2. **状态变更替代删除**：
   - 不再有效的 ADR -> 标记为 `Deprecated`（过时，无替代方案）
   - 被新决策替代 -> 标记为 `Superseded`（过时，有替代方案）
   - 新的 ADR 的 `superseded_by` 字段自动链接

3. **物理保护机制**：
   - Git pre-commit hook 禁止删除 `docs/adrs/` 目录下已合并到 main 的文件
   - SQLite 中 ADR 记录标记为 `is_protected = 1`
   - 如果确有特殊需要删除（如误创建），需要研发总监书面批准

### 4.4 定期复审

ADR 的默认复审周期为 **90天**（可在 Frontmatter 中通过 `review_cadence_days` 自定义）。

**复审时需要回答的三个核心问题**：

1. 当初做出这个决策时的约束条件是否发生了变化？
   - 是否有新的模型/技术出现？
   - 硬件资源是否发生了变化？
   - 业务需求是否发生了变化？

2. 决策执行后的实际效果如何？
   - 是否达到了预期的正面后果？
   - 是否出现了预期的负面后果？
   - 是否出现了意料之外的后果？

3. 是否需要修订或替代当前决策？
   - 约束条件变化不大且效果符合预期 -> 保持 Accepted，顺延 `next_review_date`
   - 约束条件有变但当前方案仍最优 -> 更新 Context 部分，保持 Accepted
   - 需要重新决策 -> 创建新 ADR，当前 ADR 标记为 Superseded

**复审结果记录**（以 Git commit 形式）：

```
docs(adr): review ADR-001 -- decision still valid

Reviewed constraints and outcomes:
- Latency requirement unchanged (P99 <= 200ms)
- BERT model accuracy in production: 82% (above 80% threshold)
- No new model meets cost/latency requirements better
- Next review: 2026-10-20
```

### 4.5 索引与检索

所有 ADR 通过 Skill-H 仪表盘提供以下检索能力：

- **按决策大类浏览**：ModelSelection / ArchitectureDesign / DataStrategy ...
- **按团队浏览**：ai-search / ai-recommend / saas-core ...
- **按状态过滤**：Accepted / Deprecated / Superseded
- **按时间线浏览**：按 decision_date 排序，看到决策的时间演进
- **全文搜索**：搜索 ADR 正文中的关键词
- **关联图谱**：查看 ADR 与 EXP、CA、PRD 的关联关系网络

命令行快速查询：

```bash
wc adr list --status Accepted       # 查看所有生效的 ADR
wc adr list --team ai-search        # 查看特定团队的 ADR
wc adr list --category ModelSelection  # 查看特定决策类别
wc adr list --due-soon              # 查看即将到期需要复审的 ADR
```

---

## 8. CLI 工具集成 (phodal/adr)

WisdomCore 推荐使用 [phodal/adr](https://github.com/phodal/adr) 作为 ADR 管理的 CLI 工具，它是 npryce/adr-tools 的 JS 重写版本，原生支持中文，与 WisdomCore 的 ADR 工作流完美适配。

### 8.1 安装与初始化

```bash
# 全局安装
npm install -g adr

# 在 WisdomCore 根目录初始化（指定 ADR 目录）
adr init 02_AI_Exploration/decisions
```

### 8.2 常用命令

| 命令 | 用途 | 对应 Skill-A 操作 |
|------|------|------------------|
| `adr new "选择向量搜索引擎"` | 创建新 ADR（自动编号） | "记录一个架构决策" |
| `adr list` | 列出所有 ADR | "列出所有架构决策" |
| `adr generate toc` | 生成目录页 | 自动更新 ADR 索引 |
| `adr export csv` | 导出 CSV | 度量采集数据源 |
| `adr logs` | 查看决策时间线 | "ADR 时间线" |

### 8.3 与 WisdomCore Frontmatter 的集成

phodal/adr 生成的 ADR 文件使用标准 Markdown 格式。需要在创建后手动补充 WisdomCore 扩展的 Frontmatter 字段（`related_experiments`, `team`, `product_line`）。建议流程：

1. `adr new "{title}"` — 生成基础 ADR 文件
2. 手动或通过 Skill-A 补充 WisdomCore Frontmatter
3. `git add` + Pre-commit Hook 自动校验 + SQLite 同步

> **自动化增强**：可编写一个 `adr-post-create.sh` 钩子，在 `adr new` 执行后自动注入 WisdomCore Frontmatter 模板。
