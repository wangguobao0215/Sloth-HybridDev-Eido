# Sloth-HybridDev-Eido 用户手册

> 版本：1.1.1 | 适用于 QoderWork 桌面版
>
> 本手册为 **深构 (WisdomCore/HybridDev)** 混合研发管理技能的完整使用指南，帮助用户快速上手并充分发挥技能能力。

---

## 一、技能定位与适用场景

深构（WisdomCore）是面向 SaaS + AI 混合研发团队（5-20 人）的管理中枢。它解决的核心矛盾是：SaaS 产品开发属于需求驱动的线性过程（确定性），AI 能力开发属于假设驱动的循环过程（概率性），两种范式并行运作时产生语言鸿沟、进度不同步、责任模糊和知识流失四大冲突。

适用场景包括：管理混合 SaaS+AI 研发团队、创建 SaaS 与 AI 工作流之间的协作协议、记录 AI 实验、生成健康度仪表盘或文档治理报告、生产 GTM 资产（作战卡、发版说明等）。

**能力边界**：深构管理的是知识文档和协作承诺，不管理具体开发任务；存储方式为本地 Markdown + SQLite，不依赖云端服务；输出是建议和报告供人工决策，不自动执行业务操作。

---

## 二、安装与初始化

### 2.1 加载技能

在 QoderWork 中加载本技能即可使用，无需额外安装。技能加载后，深构自动进入待命状态。

### 2.2 初始化知识库

首次使用时，发送以下指令初始化 WisdomCore 知识库：

```
帮我初始化 WisdomCore 知识库
```

初始化会创建以下目录结构和 SQLite 数据库：

```
WisdomCore/
├── docs/           # 文档存储目录
│   ├── prd/        # 产品需求文档 (PRD)
│   ├── ca/         # 协作协议 (CA)
│   ├── exp/        # 实验报告 (EXP)
│   ├── adr/        # 架构决策记录 (ADR)
│   └── gtm/        # GTM 资产
├── db/             # SQLite 数据库
└── reports/        # 生成的报告
```

### 2.3 推荐工具链

深构集成了 7 个开源工具（均可选安装，不影响核心功能）：

| 工具 | 用途 | 安装命令 |
|------|------|----------|
| sqlite-utils | SQLite 查询加速层 | `pip install sqlite-utils` |
| Datasette | SQLite 可视化浏览 | `pip install datasette` |
| remark-lint | Markdown 规范检查 | `npm install remark-lint` |
| phodal/adr | ADR 管理工具 | `npm install -g adr` |
| git-cliff | CHANGELOG 自动生成 | `cargo install git-cliff` |
| Mermaid CLI | 图表渲染 | `npm install -g @mermaid-js/mermaid-cli` |
| markdownlint | Markdown 格式校验 | `npm install -g markdownlint-cli` |

---

## 三、核心架构

深构采用"中枢神经系统"架构，四个子 Skill 形成有方向性的数据管线：

```
感觉神经（输入）        脊髓（数据通路）        大脑（分析）        运动神经（输出）
┌─────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Skill-S 协议 │────>│              │────>│ Skill-H 视图 │────>│ Skill-G GTM  │
│ Skill-A 实验 │────>│  父 Skill    │────>│              │────>│              │
│              │     │  双写通路    │     │              │     │              │
└─────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
```

**双写机制**：所有文档同时写入 Markdown 文件和 SQLite 数据库。Markdown 文件是真实数据源（Source of Truth），SQLite 仅作查询加速层。两者冲突时以 Markdown 为准，可随时通过 `rebuild-index` 全量重建索引。

详细架构图、SQLite Schema（6 表）和目录结构见 [architecture.md](architecture.md)。

---

## 四、子 Skill 使用指南

### 4.1 Skill-S：协作协议生成器

**目标用户**：SaaS 产品经理、Tech Lead

**用途**：引导创建 SaaS 与 AI 之间的标准化协作协议（Collaboration Agreement），确保每份协议包含验收标准、降级策略和双 Owner 绑定。

**常用指令**：

| 场景 | 示例指令 |
|------|----------|
| 创建新协议 | "创建一份智能搜索的协作协议" |
| 创建含降级策略的协议 | "创建协议，AI 能力是意图识别，降级方案用规则引擎兜底" |
| 查看协议状态 | "查看 CA-2026-001 的当前状态" |
| 推进协议状态 | "把 CA-2026-001 提交评审" |
| 违约检测 | "检查哪些协议存在违约" |

**创建协议流程**：

1. 用自然语言描述需求，深构引导你补全结构化字段
2. 系统自动生成 YAML Frontmatter（含验收标准、SLA、降级策略）
3. 绑定双 Owner（SaaS 侧 + AI 侧各一人）
4. 自动检查依赖链关系
5. 协议进入 Draft 状态，等待评审

**降级策略模板库**（6 套标准模板）：

| 模板 | 适用场景 |
|------|----------|
| 静默降级 | AI 失败时无感切换到规则逻辑 |
| 功能关闭 | 直接关闭依赖 AI 的功能模块 |
| 缓存回退 | 使用上次成功的 AI 结果 |
| 规则替代 | 预置的确定性规则引擎接管 |
| 人工兜底 | 转人工处理 |
| 分流降级 | 部分流量走 AI，部分走规则 |

详见 [skill-s-agreement-forge.md](skill-s-agreement-forge.md)。

### 4.2 Skill-A：实验记录员

**目标用户**：AI 产品经理、算法工程师

**用途**：模板化 AI 实验全生命周期管理，包括实验报告创建、归档、多轮对比和 Bad Case 追踪。

**常用指令**：

| 场景 | 示例指令 |
|------|----------|
| 记录新实验 | "记录一个 A/B 测试实验，对比 GPT-4 和微调模型的意图识别准确率" |
| 归档实验 | "归档 EXP-2026-003" |
| 登记 Bad Case | "注册一个 Bad Case：用户说'我要退货'被误分类为咨询" |
| 对比实验 | "对比 EXP-2026-001 到 003 的结果" |
| 触发 ADR | "这个实验结论涉及架构决策，帮我创建 ADR" |

**支持的实验方法论**（5 种）：A/B Test、Benchmark、Prototype、Survey、Pilot。

**Bad Case 分类**（5 类）：Accuracy（准确性）、Latency（延迟）、Safety（安全性）、UX（用户体验）、Integration（集成问题）。

详见 [skill-a-experiment-log.md](skill-a-experiment-log.md)。

### 4.3 Skill-H：混合视图生成器

**目标用户**：研发总监、Tech Lead

**用途**：通过 SQLite 聚合查询和全量文档扫描，计算 6 大核心指标，生成健康度报告、识别僵尸文档和断裂依赖链。

**注意**：建议在积累 4-6 周文档数据后再启用此 Skill，以获得有意义的分析结果。

**常用指令**：

| 场景 | 示例指令 |
|------|----------|
| 查看全景仪表盘 | "给我看混合视图仪表盘" |
| 生成周报 | "生成本周健康度周报" |
| 检测僵尸文档 | "哪些文档已经腐烂了？" |
| 依赖链校验 | "检查依赖链完整性" |
| 查看趋势 | "展示过去一个月的文档鲜度趋势" |

**6 大核心指标**：

| 指标 | 缩写 | 健康阈值 | 含义 |
|------|------|----------|------|
| 协作协议覆盖率 | ACR | ≥90% | 已建协议数 / 应建协议数 |
| 协议合规率 | ACMR | ≥85% | 合规协议数 / 活跃协议数 |
| 文档鲜度 | DF | ≥70% | 新鲜+临近文档占比 |
| 需求-实验关联率 | RELR | ≥80% | 已关联实验 / 总实验数 |
| GTM 产出周期 | GTT | ≤5天 | 从协议 Active 到首份 GTM 资产 |
| 僵尸文档数 | ZDC | ≤3篇 | 腐烂等级文档总数 |

详见 [skill-h-hybrid-view.md](skill-h-hybrid-view.md)。

### 4.4 Skill-G：GTM 知识架构师

**目标用户**：市场人员、销售人员

**用途**：从 PRD + CA + EXP 文档中提取市场可用信息，生成四类 GTM 资产，内置脱敏翻译规则防止过度承诺。

**常用指令**：

| 场景 | 示例指令 |
|------|----------|
| 生成作战卡 | "为智能搜索功能生成销售作战卡" |
| 生成发版说明 | "生成 v2.3 的发版说明" |
| Feature Brief | "写一份意图识别能力的 Feature Brief" |
| 客户 FAQ | "生成智能搜索的客户 FAQ" |
| 脱敏翻译 | "把这份实验报告翻译成市场语言" |

**四类 GTM 资产**：

| 资产类型 | 内容要素 |
|----------|----------|
| Feature Brief | 电梯演讲、目标用户画像、差异化优势 |
| Battle Card | 我方优劣势、竞品对比、应对话术、历史赢单率 |
| Release Notes | 亮点/新功能/改进/修复/Breaking Changes |
| Customer FAQ | 常见问题与标准答案 |

**脱敏翻译规则**：技术指标自动转换为用户语言（如 "P95 latency 200ms" → "响应速度快于同类产品"），自动加入限定词防止过度承诺。AI PM 必须确认技术准确性后方可发布。

详见 [skill-g-gtm-architect.md](skill-g-gtm-architect.md)。

---

## 五、管理流程

### 5.1 协作协议状态机

协作协议共有 7 种状态和 13 条合法转换路径：

```
Draft → UnderReview → Active → InViolation → Renegotiating
                        ↓                        ↓
                   Superseded              （回到 Active）
                   Archived
```

**关键规则**：

- Draft 不可跳过评审直达 Active
- 没有降级策略的协议不允许进入 UnderReview
- 终态（Superseded/Archived）不可逆
- 4 级违约严重度：Warning → Minor → Major → Critical，系统自动检测

**常用指令**：

| 操作 | 示例 |
|------|------|
| 查看状态 | "CA-2026-001 现在什么状态？" |
| 推进状态 | "把 CA-2026-001 推进到 Active" |
| 触发违约 | "标记 CA-2026-002 违约，原因是 SLA 超标" |
| 重新协商 | "发起 CA-2026-002 的重新协商" |
| 归档 | "归档 CA-2026-001" |

详见 [agreement-state-machine.md](agreement-state-machine.md)。

### 5.2 文档治理

每份文档有一个鲜度评分，计算公式为：

```
鲜度 = (当前日期 - updated_at) / review_cadence_days × 100%
```

鲜度四级机制：

| 等级 | 鲜度值 | 含义 |
|------|--------|------|
| 新鲜 | <80% | 文档处于正常状态 |
| 临近 | 80-99% | 即将到期，需安排更新 |
| 过期 | 100-199% | 已过审阅周期，需立即处理 |
| 腐烂 | ≥200% | 严重滞后，可能已失效 |

**Owner 制度**：每份文档必须有明确 Owner（协作协议为 SaaS + AI 双 Owner）。无主文档超 5 天由研发总监强制指派。连续 30 天腐烂状态的文档自动创建归档提案。

详见 [document-governance.md](document-governance.md)。

### 5.3 变更管理

变更分为四类：接口变更、标准变更、策略变更、配置变更。每次变更自动进行依赖链追踪（有向图），生成影响分析报告，列出所有受影响的文档和协议。

紧急变更可走绿色通道，但事后必须补齐审批记录。

详见 [change-management.md](change-management.md)。

### 5.4 ADR（架构决策记录）

当实验结论涉及架构决策时，Skill-A 会自动建议创建 ADR。ADR 遵循不可变原则：一旦记录，原文不可修改，只能通过新的 ADR 标记旧决策为 Superseded。

ADR 模板包含：上下文、候选方案、决策结果、验证计划。

详见 [adr-framework.md](adr-framework.md)。

---

## 六、协作仪式

深构定义了 4 级协作仪式体系，建议团队按节奏执行：

| 级别 | 仪式 | 频率 | 主要内容 |
|------|------|------|----------|
| L1 | 每日站会 | 每天 | 检查协议状态变更、新增违约 |
| L2 | 文档巡检 | 每周 | 文档鲜度扫描、僵尸文档处理 |
| L3 | Sprint Review | 双周 | 指标快照对比、实验进展回顾 |
| L4 | 架构回顾 | 季度 | ADR 回顾、指标趋势分析、公约修订 |

详见 [metrics-ceremonies-risk-impl.md](metrics-ceremonies-risk-impl.md)。

---

## 七、意图路由速查

不确定该说什么？以下是按场景整理的关键词速查表：

| 你想做什么 | 说这些关键词 | 路由到 |
|-----------|-------------|--------|
| 创建协作协议 | 创建协议、新建协作、SaaS对接AI、降级策略 | Skill-S |
| 记录 AI 实验 | 记录实验、归档实验、Bad Case、实验报告 | Skill-A |
| 查看全局状态 | 全景视图、健康度、仪表盘、周报 | Skill-H |
| 生成市场资料 | 销售卡、发版说明、GTM、作战卡 | Skill-G |
| 管理协议状态 | 协议状态、违约、重新协商 | 状态机 |
| 检查文档健康 | 文档鲜度、腐烂检测、僵尸文档 | 文档治理 |
| 分析变更影响 | 变更影响、依赖链、影响分析 | 变更管理 |
| 记录架构决策 | 决策记录、ADR、架构决策 | ADR 框架 |
| 查看团队指标 | 度量、指标、覆盖率、合规率 | 度量体系 |
| 配置权限 | 权限、角色、谁能看、CODEOWNERS | 安全权限 |
| 查看风险 | 风险、缓解、风险矩阵 | 风险矩阵 |
| 规划实施 | 实施、上线计划、落地路径 | 实施路径 |
| 安装工具 | 工具链、安装、sqlite-utils | 工具链 |

---

## 八、最佳实践

### 8.1 启动阶段（第 1-2 周）

1. 初始化知识库
2. 梳理团队现有的 SaaS-AI 协作点，为每个协作点创建协作协议
3. 将进行中的 AI 实验补录到系统中
4. 安装推荐工具链中你需要的部分

### 8.2 运行阶段（第 3-4 周）

1. 养成协议先行的习惯：先创建协议再开始开发
2. 实验结束后及时归档并关联到协议
3. 启用每周文档巡检（L2 仪式）
4. 开始使用 Skill-H 生成第一份周报

### 8.3 优化阶段（第 5-8 周）

1. 关注 6 大核心指标的趋势变化
2. 使用 Skill-G 为已稳定的 AI 能力生成 GTM 资产
3. 启用 Sprint Review 仪式（L3），回顾指标并调整策略
4. 处理积压的僵尸文档和断裂依赖链

### 8.4 成熟阶段（第 8 周+）

1. 所有 6 大指标进入健康区间
2. 协作仪式已成为团队习惯
3. 协议和实验数据足够支持季度架构回顾（L4 仪式）
4. 可考虑扩展到多团队协同场景

---

## 九、常见问题

**Q: 深构和 Jira/飞书什么关系？**
A: 深构不替代 Jira 管任务、不替代飞书做沟通。它在二者之上建立结构化的知识层与承诺层。Jira 管"做什么"，飞书管"怎么沟通"，深构管"为什么这样做"和"承诺了什么"。

**Q: SQLite 数据库坏了怎么办？**
A: Markdown 文件是真实数据源，SQLite 只是查询加速层。执行 `rebuild-index` 可从 Markdown 全量重建 SQLite 数据库，不会丢失任何数据。

**Q: 可以不用 SQLite 吗？**
A: 可以。没有 SQLite 时，深构会直接扫描 Markdown 文件进行查询，只是速度会慢一些。SQLite 是可选的性能优化层。

**Q: 协作协议和普通需求文档有什么区别？**
A: 协作协议是 SaaS 侧和 AI 侧之间的"握手承诺"，包含验收标准、SLA、降级策略和双 Owner。普通需求文档只描述功能，协作协议还约定了"做不到怎么办"。

**Q: 团队只有 3 个人，适合用吗？**
A: v1.1 版本针对 5-20 人团队设计，但 3 人团队如果同时有 SaaS 和 AI 两条线，也可以使用简化版流程。建议先从 Skill-S（协作协议）和 Skill-A（实验记录）开始，按需扩展。

**Q: 如何和其他 Sloth-Eido 技能配合？**
A: Skill-G 生成的 Battle Card 和 Feature Brief 可直接导入 Sloth-Sales-Eido 的客户管理流程；售前顾问可通过 Sloth-PSC-Eido 从 GTM 资产中获取技术底气和量化数据。

---

## 十、参考文档索引

| 文档 | 内容 |
|------|------|
| [architecture.md](architecture.md) | 系统架构、双写机制、SQLite Schema、目录结构 |
| [parent-skill.md](parent-skill.md) | 父 Skill 规范、Frontmatter Schema、模板 |
| [agreement-state-machine.md](agreement-state-machine.md) | 7 态 13 转换、违约检测、重新协商流程 |
| [document-governance.md](document-governance.md) | Owner 制度、鲜度机制、腐烂检测 |
| [change-management.md](change-management.md) | 变更分类、依赖链追踪、影响分析 |
| [adr-framework.md](adr-framework.md) | ADR 模板、治理规则、不可变原则 |
| [skill-s-agreement-forge.md](skill-s-agreement-forge.md) | Skill-S 详细规范 |
| [skill-a-experiment-log.md](skill-a-experiment-log.md) | Skill-A 详细规范 |
| [skill-h-hybrid-view.md](skill-h-hybrid-view.md) | Skill-H 详细规范 |
| [skill-g-gtm-architect.md](skill-g-gtm-architect.md) | Skill-G 详细规范 |
| [metrics-ceremonies-risk-impl.md](metrics-ceremonies-risk-impl.md) | 指标、仪式、风险、实施路径 |
