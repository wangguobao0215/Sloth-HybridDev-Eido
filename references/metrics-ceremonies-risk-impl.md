# 度量、仪式、安全、风险与实施路径 — 运营参考手册

> 本文档整合第 13-17 章内容，涵盖度量体系、协作仪式、安全权限、风险矩阵
> 及 8 周 4 阶段实施路径。作为 Sloth-HybridDev-Eido 的运营参考手册使用。

---

## 第一部分：度量体系

### 1.1 度量哲学

三条核心信念：

1. **度量是为了改进，不是为了惩罚** — 聚焦"流程哪里需要优化"，而非"谁做得不好"
2. **每个指标必须有明确的行动触发点** — 三个阈值（健康/预警/危险）+ 对应行动方案
3. **指标数量控制在 6 个以内** — 新增指标必须同时替换一个现有指标

设计原则：`少量指标 x 明确阈值 x 自动采集 x 行动驱动 = 有效度量`

### 1.2 六大核心指标

#### 指标 1：协作协议覆盖率 (ACR)

- **定义**：有效协作协议覆盖的 AI 功能占全部含 AI 依赖功能的比例
- **公式**：`ACR = COUNT(DISTINCT ca.related_prd WHERE ca.status='Active') / COUNT(prd WHERE ai_dependency=true) x 100%`
- **阈值**：健康 >= 90% | 预警 70%-89% | 危险 < 70%
- **危险触发动作**：
  1. Skill-H 自动列出无协议的 AI 需求清单
  2. 研发总监在周巡检中逐项审查
  3. 确认需要的需求强制在下一 Sprint 创建协议
  4. 连续两个 Sprint 仍为危险则升级至 CTO

```sql
-- ACR: 有效协作协议覆盖的 AI 功能占比
SELECT ROUND(
  CAST(covered.cnt AS REAL) / NULLIF(total_ai.cnt, 0) * 100, 1
) AS agreement_coverage_rate_pct
FROM
  (SELECT COUNT(DISTINCT d.id) AS cnt FROM documents d
   JOIN dependencies dep ON dep.source_id = d.id AND dep.relation_type = 'governed_by'
   JOIN documents ca ON ca.id = dep.target_id
   WHERE d.type = 'PRD'
     AND json_extract(d.frontmatter_json, '$.ai_dependency') = 'true'
     AND ca.type = 'CollaborationAgreement'
     AND ca.status = 'Active') AS covered,
  (SELECT COUNT(*) AS cnt FROM documents d
   WHERE d.type = 'PRD'
     AND json_extract(d.frontmatter_json, '$.ai_dependency') = 'true'
     AND d.status NOT IN ('Archived', 'Superseded')) AS total_ai;
```

#### 指标 2：协议合规率 (ACMR)

- **定义**：Active 协议占所有应当活跃协议 (Active + InViolation) 的比例
- **公式**：`ACMR = COUNT(ca WHERE status='Active') / COUNT(ca WHERE status IN ('Active','InViolation')) x 100%`
- **阈值**：健康 >= 85% | 预警 70%-84% | 危险 < 70%
- **危险触发动作**：
  1. InViolation 协议 10 个工作日内必须进入 Renegotiating
  2. Owner 提交根因分析（约束不合理 vs 执行不到位）
  3. 连续 3 周危险则暂停新功能开发，专项治理

```sql
-- ACMR: Active 协议占 Active + InViolation 协议的比例
SELECT ROUND(
  CAST(active.cnt AS REAL) / NULLIF(active.cnt + violated.cnt, 0) * 100, 1
) AS agreement_compliance_rate_pct
FROM
  (SELECT COUNT(*) AS cnt FROM documents
   WHERE type = 'CollaborationAgreement' AND status = 'Active') AS active,
  (SELECT COUNT(*) AS cnt FROM documents
   WHERE type = 'CollaborationAgreement' AND status = 'InViolation') AS violated;
```

#### 指标 3：文档鲜度 (DF)

- **定义**：在评审周期内的文档占全部活跃文档的比例
- **各类评审周期**：PRD 14 天 / EXP 进行中 7 天・已完成 30 天 / CA 14 天（默认） / ADR 90 天 / GTM 30 天
- **阈值**：健康 >= 70% | 预警 50%-69% | 危险 < 50%
- **危险触发动作**：Sprint Review 安排 20 分钟专项清理

```sql
-- DF: 在评审周期内的文档占全部活跃文档的比例
SELECT ROUND(
  CAST(SUM(CASE WHEN julianday('now') - julianday(d.updated_at) <
    CASE d.type
      WHEN 'PRD' THEN 14
      WHEN 'Experiment' THEN
        CASE WHEN d.status IN ('Completed') THEN 30 ELSE 7 END
      WHEN 'CollaborationAgreement' THEN 14
      WHEN 'ADR' THEN 90
      WHEN 'GTM' THEN 30
    END
    THEN 1 ELSE 0 END) AS REAL) / NULLIF(COUNT(*), 0) * 100, 1
) AS document_freshness_pct
FROM documents d
WHERE d.status NOT IN ('Archived', 'Superseded');
```

#### 指标 4：需求-实验关联率 (RELR)

- **定义**：有关联实验记录的 AI 需求占全部 AI 需求的比例
- **阈值**：健康 >= 80% | 预警 60%-79% | 危险 < 60%
- **危险触发动作**：AI 团队优先启动缺失实验的探索；将"实验关联"纳入 DoR 检查项

```sql
-- RELR: 有关联实验的AI需求占比
SELECT ROUND(
  CAST(COUNT(DISTINCT CASE
    WHEN EXISTS (
      SELECT 1 FROM dependencies dep
      WHERE dep.source_id = d.id
        AND dep.relation_type IN ('implements','governed_by')
    ) THEN d.id END) AS REAL)
  / NULLIF(COUNT(DISTINCT d.id), 0) * 100, 1
) AS relr
FROM documents d
WHERE d.type = 'PRD'
  AND json_extract(d.frontmatter_json, '$.ai_dependency') = 'true'
  AND d.status NOT IN ('Archived');
```

#### 指标 5：GTM 产出周期 (GTT)

- **定义**：功能发布到 GTM 资料就绪的平均天数（统计最近 90 天）
- **阈值**：健康 <= 5 天 | 预警 5-10 天 | 危险 > 10 天
- **危险触发动作**：GTM 资料纳入发布流程前置条件；Sprint Planning 预留 GTM 工时

```sql
-- GTT: 功能发布到GTM就绪的平均天数
SELECT ROUND(AVG(
    julianday(g.created_at) - julianday(sh.changed_at)
), 1) AS avg_gtt_days
FROM documents g
JOIN dependencies dep ON dep.source_id = g.id
JOIN documents d ON d.id = dep.target_id
JOIN status_history sh ON sh.document_id = d.id
  AND sh.to_status = 'Released'
WHERE g.type = 'GTM'
  AND d.type = 'PRD'
  AND d.status = 'Released';
```

#### 指标 6：僵尸文档数 (ZDC)

- **定义**：超过 60 天未更新且非归档状态的文档数量（绝对值）
- **阈值**：健康 <= 3 篇 | 预警 4-8 篇 | 危险 > 8 篇
- **危险触发动作**：Owner 必须 3 个工作日内给出处置意见（更新/归档/移交）

```sql
-- ZDC: 超过60天未更新且非归档的文档数
SELECT COUNT(*) AS zombie_count
FROM documents
WHERE status NOT IN ('Archived', 'Superseded')
  AND julianday('now') - julianday(updated_at) > 60;
```

### 1.3 度量快照与趋势

**自动快照**：每周日 22:00 由 `scripts/metrics_snapshot.py` 运行，写入 SQLite：

```sql
-- 行式存储，与 architecture.md 规范 Schema 一致
CREATE TABLE IF NOT EXISTS metrics_snapshots (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    snapshot_date TEXT NOT NULL,
    metric_name TEXT NOT NULL,
    metric_value REAL NOT NULL,
    dimension TEXT,
    metadata_json TEXT,
    created_at TEXT DEFAULT (datetime('now'))
);
```

**趋势展示**：Skill-H 读取最近 8 周数据输出趋势表：

```
Week    ACR     ACMR    DF      RELR    GTT     ZDC
W01     72%     80%     55%     65%     8.2d    6
...
W08     91%     88%     73%     82%     4.1d    2
```

### 1.4 PDCA 持续改进循环

- **Plan**：月度报告后，研发总监与产品总监共同分析，聚焦 1-2 个最需改进的指标
- **Do**：制定并执行具体改进动作
- **Check**：2-4 周后验证改进效果（对比指标趋势）
- **Act**：有效动作固化为标准流程；无效动作及时放弃

每个 PDCA 循环不超过 4 周，快速试错、快速迭代。

### 1.5 反度量陷阱

- **陷阱一**：把度量当 KPI 考核 -> 导致"优化指标"而非"优化流程"
- **陷阱二**：追求所有指标 100% -> 意味着过度管理
- **陷阱三**：指标膨胀 -> 严格遵守"6 个以内"铁律

**总结**：度量是方向盘，不是鞭子。

---

## 第二部分：协作仪式与日历

### 2.1 仪式设计原则

仪式解决三个核心问题：确保工具被使用、确保数据被更新、确保问题被发现。
核心原则："固定时间做固定的事"。

### 2.2 四级仪式体系

```
周级   -> 战术执行：协议状态、阻塞点、即时行动
双周级 -> 战术回顾：Sprint 成果、指标趋势、计划调整
月级   -> 战略协调：GTM 对齐、跨部门协作、资源调配
季度级 -> 战略回顾：架构演进、流程优化、方向调整
```

#### 仪式 1：周级 — 协议状态巡检（15 分钟站会）

| 项目 | 说明 |
|------|------|
| 频率 | 每周一上午 10:00 |
| 时长 | 15 分钟（严格计时） |
| 参与者 | SaaS PM 代表 + AI PM 代表 + 研发总监 |
| 工具 | Skill-H 周度巡检报告（提前 30 分钟自动发送） |

**标准议程**：
1. [3 分钟] 协议变更速览 — 新增/关闭/状态变更
2. [5 分钟] 违约协议处理进展 — 根因 + Renegotiating 计划
3. [3 分钟] 文档鲜度预警 — 即将过期文档确认
4. [2 分钟] 阻塞点识别 — 跨团队依赖/资源不足
5. [2 分钟] 待办确认 — Action Items 责任人和截止日期

#### 仪式 2：双周级 — 混合 Sprint Review（60 分钟）

| 项目 | 说明 |
|------|------|
| 频率 | 每两周周五下午 14:00 |
| 时长 | 60 分钟 |
| 参与者 | 全体 SaaS + AI 产品研发人员 |
| 工具 | Skill-H 全景仪表盘 + 实际系统 Demo |

**标准议程**：
1. [15 分钟] SaaS 功能 Demo + AI 集成验证
2. [15 分钟] AI 实验进展汇报 + Bad Case 消解情况
3. [10 分钟] 协作协议健康度报告
4. [10 分钟] 度量指标回顾 + PDCA 效果验证
5. [10 分钟] 下一 Sprint 计划预览

#### 仪式 3：月级 — GTM 知识审查（30 分钟）

| 项目 | 说明 |
|------|------|
| 频率 | 每月第一个周三 14:00 |
| 时长 | 30 分钟 |
| 参与者 | 产品总监 + 市场部代表 + 销售代表 |

**标准议程**：
1. [10 分钟] 上月新增 GTM 资产展示 + 使用反馈
2. [10 分钟] 市场与销售反馈（准确性/表述/缺失资料）
3. [10 分钟] 下月 GTM 计划（优先级排序 + 责任分配）

#### 仪式 4：季度级 — 架构回顾（120 分钟）

| 项目 | 说明 |
|------|------|
| 频率 | 每季度最后一周 |
| 时长 | 120 分钟 |
| 参与者 | 研发总监 + Tech Lead + AI 负责人 + 产品总监 |

**标准议程**：
1. [30 分钟] 度量趋势分析 — 本季 vs 上季
2. [30 分钟] ADR 回顾 — 决策复盘与修正
3. [20 分钟] 标签体系审查 — 清理/新增/一致性
4. [20 分钟] 僵尸文档批量清理
5. [20 分钟] 流程改进提案

### 2.3 升级路径（协作分歧处理）

| 级别 | 处理人 | 时限 | 适用场景 |
|------|--------|------|---------|
| L1 | Owner 自行协商 | 48 小时 | 细节调整（指标微调、降级优化） |
| L2 | 各自上级介入 | 96 小时 | 产品方向分歧（功能范围、资源分配） |
| L3 | 研发总监裁决 | 48 小时 | 跨团队资源冲突、技术路线分歧 |
| L4 | CTO 裁决 | 极端情况 | 影响公司战略 / 研发总监为分歧方 |

### 2.4 仪式评效

每季度评估：产出有效性、时间效率、参与度、改进速率。
调整机制：连续 2 个月无有效产出 -> 降低频率或合并；持续超时 -> 缩减议程。

---

## 第三部分：安全与权限分层

### 3.1 角色权限矩阵

| 角色 | `01_SaaS/` | `02_AI/` | `03_CA/` | `04_GTM/` | `.meta/` |
|------|-----------|---------|---------|----------|---------|
| SaaS PM | 读写 | 只读 | 读写（SaaS 侧字段） | 只读 | 无权限 |
| AI PM | 只读 | 读写 | 读写（AI 侧字段） | 无权限 | 无权限 |
| 研发总监 | 读写 | 读写 | 读写（全部） | 读写 | 读写 |
| Tech Lead | 读写 | 读写 | 只读 | 无权限 | 只读 |
| 市场/销售 | 无权限 | 无权限 | 只读（脱敏版） | 读写 | 无权限 |
| 外部顾问 | 只读 | 只读（脱敏版） | 只读（脱敏版） | 只读 | 无权限 |

### 3.2 实现阶段

**小团队阶段（5-20 人）**：
- 基于信任的软约束 + Git branch protection + CODEOWNERS 文件
- Pre-commit hook 校验（非法修改 Warning，不阻断）

```
# CODEOWNERS
/01_SaaS/    @saas-pm-team
/02_AI/      @ai-pm-team
/03_CA/      @saas-pm-team @ai-pm-team
/04_GTM/     @product-director @marketing-team
/.meta/      @tech-lead
```

> **markdownlint 风格一致性**：所有文档在 pre-commit 时经过 markdownlint-cli 检查，确保标题层级、列表缩进、空行规范的一致性。配置文件 `.meta/.markdownlint.yaml`：
>
> ```yaml
> MD013: false          # 不限制行长度（中文文档天然较长）
> MD033: false          # 允许 HTML 内联（Mermaid 等）
> MD041: false          # 不要求第一行是 H1（Frontmatter 优先）
> MD024:
>   siblings_only: true # 允许不同层级下的同名标题
> ```

**成长阶段（20-50 人）**：Git submodules + CI/CD 强制权限检查 + 审计日志增强

**企业阶段（50+ 人）**：Enterprise RBAC + SSO 集成 + 审批流引擎

### 3.3 敏感数据处理

- **AI 实验数据脱敏**：PII 移除、用户行为仅记录统计信息、Bad Case 使用合成数据
- **商业数据保护**：ARR/MRR 用相对增长率、客户名称匿名化、内部/外部版本分离
- **凭证管理**：`.env`/`credentials.json`/`*.pem` 等绝对禁止入库；Pre-commit hook 扫描凭证泄露模式

### 3.4 审计日志

> **注**：下表对应 architecture.md 中的 `change_log` 表。此处展示的列与基础
> Schema 一致；如需扩展（如增加 `change_source`、`ip_address` 等审计增强字段），
> 应在 `.meta/schema.sql` 中追加列并同步更新 architecture.md。

```sql
CREATE TABLE IF NOT EXISTS change_log (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  document_id TEXT    NOT NULL,
  field_name  TEXT    NOT NULL,
  old_value   TEXT,
  new_value   TEXT,
  changed_by  TEXT    NOT NULL,
  changed_at  TEXT    NOT NULL DEFAULT (datetime('now')),
  commit_sha  TEXT,
  FOREIGN KEY (document_id) REFERENCES documents(id) ON DELETE CASCADE
);
```

---

## 第四部分：风险矩阵与缓解

### 4.1 完整风险矩阵 (R1-R12)

| ID | 风险描述 | 可能性 | 影响 | 等级 | 核心缓解策略 | 监控指标 |
|----|---------|--------|------|------|-------------|---------|
| R1 | 文档腐烂：团队停止更新文档 | 高 | 致命 | 红 | 鲜度自动检测 + Sprint Review 固定审查 + 过期文档不算有效 | DF |
| R2 | 协议流于形式：创建但不遵守 | 中 | 高 | 红 | 状态机管理 + InViolation 自动检测 + 升级机制 | ACMR |
| R3 | Git 门槛过高：非技术人员放弃 | 中 | 高 | 黄 | Skill 封装引导式工作流 + 简化命令 + 专人辅导 | 非技术用户创建频率 |
| R4 | 脚本无人维护：开发者离职 | 中 | 中 | 黄 | Tech Lead 兜底 + 脚本极简(<200 行) + 充分注释 | 脚本错误率 |
| R5 | 性能瓶颈：文档增长后查询变慢 | 低 | 中 | 绿 | SQLite 索引优化 + 定期归档 + >500 篇评估迁移 | 查询 P95 |
| R6 | 权限混乱：规模扩大后边界不清 | 低 | 高 | 黄 | CODEOWNERS + 入职培训覆盖权限 | 未授权修改次数 |
| R7 | 数据不一致：与 Jira/飞书冲突 | 中 | 中 | 黄 | 明确 Source of Truth 边界 + 每月对齐检查 | 冲突次数 |
| R8 | AI 团队抵触：认为文档是负担 | 高 | 中 | 黄 | "不记录不排期"制度 + Skill-A 自动填充 + 价值展示 | 实验报告提交率 |
| R9 | GTM 过度承诺：描述超出实际 | 低 | 高 | 黄 | Skill-G 脱敏翻译 + 审签流程 + 事实检查 | 客户投诉数 |
| R10 | 知识断层：关键人员离职 | 中 | 高 | 黄 | Owner 制度 + ADR 记录上下文 + Backup Owner | 单点 Owner 文档数 |
| R11 | SQLite 索引漂移/损坏 | 中 | 高 | 黄 | weekly rebuild dry-run comparison | 重建前后 diff 行数 |
| R12 | AI 模型漂移 | 高 | 高 | 红 | CA monitoring cadence + InViolation tracking | ACMR + InViolation 持续天数 |

### 4.2 R1 文档腐烂 — 多层防线

1. **自动检测**：`metrics_snapshot.py` 每周计算鲜度，低于阈值自动通知
2. **仪式驱动**：周巡检 + Sprint Review 过期文档清理
3. **制度约束**：过期文档不计入有效统计、不能被引用
4. **文化建设**：让团队理解更新文档是为了自己不踩坑

### 4.3 R8 AI 团队抵触 — 降本增值

- **成本降低**：Skill-A 自动从代码/Notebook 提取信息填充模板
- **价值提升**：Bad Case Registry 帮助定位问题、实验报告帮助新人了解历史
- **制度保障**："不记录不排期"确保记录与工作的对称性

### 4.4 风险监控日历

| 频率 | 覆盖风险 | 检查方式 |
|------|---------|---------|
| 每周 | R1, R2, R8, R11, R12 | Skill-H 周报自动包含 |
| 每月 | R3, R4, R7 | 统计非技术用户活跃度、脚本错误、数据一致性抽查 |
| 每季 | R5, R6, R9, R10 | 查询性能测试、Git 分析、GTM 反馈、单点 Owner 统计 |

### 4.5 风险升级机制

- **绿 -> 黄**：Sprint Review 新增议程讨论 + 增加监控频率
- **黄 -> 红**：48 小时内召开专项会议 + 缓解计划含行动/责任人/截止/验收
- **红色持续 2 周无改善**：研发总监向 CTO 提交升级报告
- **等级下降必须有数据支撑**：度量指标实际变化，不能仅凭主观判断

---

## 第五部分：实施路径

### 5.1 总览：4 阶段 8 周

```
阶段一 (W1-2)      阶段二 (W3-4)      阶段三 (W5-6)      阶段四 (W7-8)
基础搭建            生产者上线          消费者上线          度量与优化
目录/SQLite/Git     Skill-S + Skill-A   Skill-H + Skill-G   度量/仪式/PDCA
示范文档            种子用户+试点       全员培训+仪表盘     基准建立+运营
```

设计原则：先基础后应用 -> 先生产者后消费者 -> 同步开发分批上线。

### 5.2 阶段一：基础搭建（W1-2）

**第 1 周交付物**：
- 完整目录结构 `01_SaaS/` `02_AI/` `03_CA/` `04_GTM/` `.meta/`
- 团队名册 `.meta/team_roster.yml`
- 标签分类体系 `.meta/tag_taxonomy.yml`
- Frontmatter JSON Schema（prd/exp/ca/adr/gtm 五套）
- SQLite Schema（documents/dependencies/status_history/change_log/bad_cases/metrics_snapshots）
- Git 仓库初始化（.gitignore / CODEOWNERS / README.md）
- 工具链安装验证：sqlite-utils + remark-lint + Datasette + git-cliff 安装成功并通过冒烟测试

**第 2 周交付物**：
- Pre-commit hook: Frontmatter 校验 + SQLite 同步
- Git branch protection（1 人 Review + Pre-commit 通过）
- 5 份示范文档（2 PRD + 1 EXP + 1 CA + 1 ADR）
- 文档创建指南（面向非技术人员）

**验收标准**：
- 合规文档通过校验，不合规文档报错
- 文档 commit 后 SQLite 自动更新
- 至少 2 名团队成员能独立完成创建-提交流程

### 5.3 阶段二：生产者上线（W3-4）— Skill-S + Skill-A

**第 3 周交付物**：Skill-S/Skill-A 开发完成、降级策略模板库（6 套）、Bad Case Registry Schema、选定试点项目

**第 4 周交付物**：种子用户培训（2 小时工作坊）、至少 3 份真实协作协议、至少 2 份真实实验报告、反馈收集与迭代修复

**验收标准**：
- SaaS PM 能通过 Skill-S 在 30 分钟内创建完整协作协议
- AI PM 能通过 Skill-A 归档实验报告并自动建立链接
- 种子用户满意度 >= 6/10

### 5.4 阶段三：消费者上线（W5-6）— Skill-H + Skill-G

**前提条件**：SQLite 至少 8 份文档（含阶段一示范文档；3+ PRD, 2+ EXP, 2+ CA, 1+ ADR）+ 至少 2 周真实数据

**第 5 周交付物**：Skill-H/Skill-G 开发完成、首版全景仪表盘、销售作战卡模板、发版说明模板、6 大度量指标验证

**第 6 周交付物**：全员培训（3 小时工作坊）、研发总监使用 Skill-H、市场部使用 Skill-G 生成首份销售卡、度量快照首次运行
- Datasette Web UI 上线，团队成员可自助浏览文档索引和度量指标
- Mermaid CLI 导出依赖图 SVG 至 05_Reports/

**验收标准**：
- 研发总监能看到全景依赖图和健康度评分
- 市场部能一键生成销售作战卡 Draft
- 6 大度量指标全部可正确计算
- 全员培训参与率 >= 90%

### 5.5 阶段四：度量与优化（W7-8）

**第 7 周交付物**：首次执行全部三级仪式（周巡检/Sprint Review/GTM 审查）、度量周报首次生成、风险矩阵首次评估

**第 8 周交付物**：第二次巡检（对比上周改善）、度量月报首次生成、首个 PDCA 改进循环、全体满意度调查、阶段总结报告

**验收标准**：
- 4 级仪式均执行至少 1 次
- 6 大指标有基准值（至少 2 周数据）
- 至少发现并修复 1 个流程问题
- 全体成员满意度 >= 7/10
- 团队已习惯自主使用深构系统

### 5.6 成功标准（8 周后评估）

| 维度 | 成功标准 | 评估方式 |
|------|---------|---------|
| **采纳** | >= 80% AI 需求有对应协作协议 | ACR 指标 |
| **效率** | 协作协议创建时间 < 30 分钟 | 实际操作计时抽样 |
| **质量** | 协议合规率 >= 85% | ACMR 指标 |
| **文化** | "先写协议再写代码"成为共识 | 匿名问卷认同度 >= 7/10 |
| **满意度** | 全体成员满意度 >= 7/10 | 匿名问卷 10 分制 |

**评估时间点**：第 8 周最后一个工作日，由研发总监执行。

成功标准是"底线"而非"天花板"。8 周后如果所有维度均达标，说明系统已初步落地；
真正的价值需要在 3-6 个月的持续运营中充分体现。深构不是一次性项目，
而是持续演进的管理系统。

### 5.7 规模化路径

| 维度 | 触发条件 | 当前方案 | 升级方案 |
|------|---------|---------|---------|
| 团队规模 | > 30 人 | Git 单仓库 | Git submodules / monorepo 工具 |
| 文档数量 | > 500 篇 | SQLite | DuckDB 或 PostgreSQL |
| 地理分布 | 多地办公 | 本地 + Git remote | 云同步 + 异步协作强化 |
| 合规要求 | 企业级审计 | 信任 + CODEOWNERS | RBAC + 审批流 + SSO |

**升级原则**：数据格式不变（始终 Markdown + YAML Frontmatter）、渐进式升级、
有明确前提条件和决策标准、升级决策本身也是 ADR。

### 5.8 git-cliff 自动 CHANGELOG 生成

WisdomCore 推荐使用 [git-cliff](https://github.com/orhun/git-cliff) 从 Git commit 历史自动生成 CHANGELOG.md，减少手动维护风险（对应风险 R3）。

**配置文件 `.meta/cliff.toml`：**

```toml
[changelog]
header = "# Changelog\n\nAll notable changes to WisdomCore will be documented in this file.\n"
body = """
{% for group, commits in commits | group_by(attribute="group") %}
### {{ group | upper_first }}
{% for commit in commits %}
- {{ commit.message | upper_first }} ({{ commit.id | truncate(length=7, end="") }})
{% endfor %}
{% endfor %}
"""
trim = true

[git]
conventional_commits = true
commit_parsers = [
    { message = "^feat", group = "Added" },
    { message = "^update", group = "Changed" },
    { message = "^fix", group = "Fixed" },
    { message = "^status", group = "Status Changes" },
    { message = "^archive", group = "Archived" },
    { message = "^meta", group = "Meta" },
    { message = "^refactor", group = "Refactored" },
]
```

**发版流程（增强版）：**

```bash
# 1. 更新 SKILL.md version 字段
# 2. 自动生成 CHANGELOG
git cliff --config .meta/cliff.toml -o CHANGELOG.md

# 3. 提交 + 打 tag
git add -A && git commit -m "meta(release): v1.1.0"
git tag v1.1.0 && git push origin main --tags
```
