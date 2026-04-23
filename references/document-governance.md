# 文档治理机制 (Document Governance)

> 文档治理是 WisdomCore 体系的长效保障机制。再好的流程设计，
> 如果文档得不到持续维护，最终都会沦为"写完即死"的纸上谈兵。
> 本文建立从所有权、鲜度监控到腐烂升级的完整治理体系。

---

## 1. 文档所有权制度

### 1.1 所有权规则

WisdomCore 中每份文档都必须有明确的 Owner，写在 Frontmatter 中：

| 文档类型 | Owner 字段 | 规则 |
|---------|-----------|------|
| PRD（产品需求文档） | `owner: "ZhangSan"` | 单一 Owner，通常是产品经理 |
| EXP（实验报告） | `owner: "LiSi"` | 单一 Owner，通常是 AI 工程师 |
| CA（协作协议） | `saas_owner: "ZhangSan"` + `ai_owner: "LiSi"` | 双 Owner，分别代表 SaaS 和 AI 团队 |
| ADR（架构决策记录） | `author: "LiSi"` | 单一 Author（决策提出者），但 Tech Lead 有审批权 |
| GTM（上市策略文档） | `owner: "WangWu"` | 单一 Owner，通常是市场/GTM 负责人 |

### 1.2 Owner 的义务

每位文档 Owner 需要承担以下义务：

1. **定期复审**：在 `next_review_date` 之前完成文档内容的复审与更新
2. **响应变更通知**：当关联文档发生变更时，在3个工作日内评估影响并做出响应
3. **参加评审会议**：出席与所管文档相关的 Sprint Review 环节和专题评审会
4. **保持 Frontmatter 准确**：确保 `updated_at`、`status`、`tags` 等元数据与实际内容一致
5. **交接义务**：离职或转岗时，必须完成文档所有权交接（见 1.3）

### 1.3 Owner 变更流程

当文档 Owner 需要变更时（离职、转岗、职责调整），执行以下流程：

```
步骤1: 旧 Owner 提名新 Owner
  |-- wc doc transfer CA-2026-001 --new-owner "ZhaoLiu"

步骤2: 新 Owner 确认接手
  |-- 系统发送确认请求，新 Owner 需在5个工作日内确认
  |-- 确认时新 Owner 需声明已阅读并理解文档内容

步骤3: 更新 Frontmatter
  |-- 系统自动更新 owner 字段
  |-- 提交 Git commit，commit message 注明所有权变更

步骤4: 通知相关方
  |-- 通知关联文档的 Owner，告知联系人变更
  |-- 在 Skill-H 仪表盘更新 Owner 映射
```

如果新 Owner 在5个工作日内未确认，系统自动通知研发总监介入协调。

### 1.4 无主文档处理

当出现以下情况时，文档进入"无主"状态：
- Owner 离职且未完成交接
- Owner 拒绝继续维护（需提供书面理由）

无主文档的处理规则：
1. 系统自动标记为 `owner: "UNOWNED"` 并通知研发总监
2. 研发总监需在10个工作日内指定新 Owner
3. 超过10个工作日仍无主的文档，自动列入下一次 Sprint Review 的讨论议程
4. 超过30天仍无主的文档，自动生成归档提案

---

## 2. 文档鲜度机制

### 2.1 鲜度计算公式

文档鲜度（Freshness）用一个百分比数值衡量文档的"老化程度"：

```
鲜度 = (当前日期 - updated_at) / review_cadence_days x 100%
```

其中：
- `updated_at`：文档最后一次实质性更新的日期（Frontmatter 字段）
- `review_cadence_days`：该文档类型对应的复审周期（天数）

### 2.2 鲜度等级定义（4 级）

| 等级 | 标识 | 鲜度值 | 含义 | 系统行为 |
|------|------|--------|------|---------|
| 新鲜 | Fresh (绿色) | < 80% | 文档在复审周期内，内容可信 | 无特殊处理，仪表盘显示绿色 |
| 临近 | Due Soon (黄色) | 80% ~ 99% | 即将到达复审日期，提醒 Owner 准备更新 | 发送提醒通知给 Owner；仪表盘标黄 |
| 过期 | Overdue (红色) | 100% ~ 199% | 已超过复审周期，内容可能过时 | 通知 Owner（每3天重复通知）；仪表盘标红 |
| 腐烂 | Decayed (骷髅) | >= 200% | 严重过期，内容高度不可信 | 通知 Owner + 研发总监；仪表盘标灰+骷髅图标 |

### 2.3 各文档类型的复审周期

| 文档类型 | 复审周期 (`review_cadence_days`) | 说明 |
|---------|-------------------------------|------|
| PRD | 14 天 | 产品需求变化较快，需要频繁确认 |
| EXP（进行中） | 7 天 | 实验进行期间需要频繁更新进展 |
| EXP（已完成） | 30 天 | 已完成实验的复审主要确认结论是否仍有效 |
| CA | 按 `review_cadence_days` 字段（默认14天） | 协作协议可以自定义复审周期 |
| ADR | 90 天 | 架构决策相对稳定，季度复审即可 |
| GTM | 30 天 | 上市策略需要月度复审以跟踪市场变化 |

### 2.4 next_review_date 自动计算规则

`next_review_date` 的自动计算逻辑：

```python
def calculate_next_review_date(doc):
    base_date = doc.updated_at  # 以最后更新日期为基准
    cadence = doc.review_cadence_days

    # 特殊规则：实验报告根据状态决定周期
    if doc.type == "EXP":
        if doc.status in ["Running", "Planned"]:
            cadence = 7
        else:  # Completed, Failed, Cancelled
            cadence = 30

    next_review = base_date + timedelta(days=cadence)

    # 如果计算出的日期是周末，顺延到下周一
    while next_review.weekday() >= 5:  # 5=Saturday, 6=Sunday
        next_review += timedelta(days=1)

    return next_review
```

---

## 3. 腐烂检测与升级

### 3.1 每日自动扫描

系统每日凌晨 02:00 执行文档鲜度扫描，使用以下 SQL 查询：

```sql
-- 查询所有过期文档 (鲜度 >= 100%)
SELECT
    d.id,
    d.type,
    d.title,
    COALESCE(d.owner, d.saas_owner),
    d.updated_at,
    d.review_cadence_days,
    d.next_review_date,
    ROUND(
        JULIANDAY('now') - JULIANDAY(d.updated_at)
    ) / d.review_cadence_days * 100 AS freshness_pct
FROM documents d
WHERE d.status NOT IN ('Archived', 'Superseded', 'Deprecated')
  AND JULIANDAY('now') > JULIANDAY(d.next_review_date)
ORDER BY freshness_pct DESC;
```

### 3.2 通知升级规则

| 鲜度等级 | 通知对象 | 通知频率 | 通知方式 |
|---------|---------|---------|---------|
| Due Soon (黄色) | Owner | 到期前3天发送一次 | IM 消息 |
| Overdue (红色) | Owner | 每3天重复通知 | IM 消息 + 邮件 |
| Decayed (骷髅) | Owner + 研发总监 | 每3天重复通知 | IM 消息 + 邮件 + 仪表盘全局横幅 |

### 3.3 长期腐烂文档的自动归档提案

当文档连续处于 Decayed 状态超过30天时：

1. 系统自动创建归档提案（PR）
2. 归档提案包含：
   - 腐烂文档清单
   - 每份文档的最后更新日期和当前鲜度
   - 关联的其他文档（受影响范围）
   - 建议操作：归档 / 转交 / 紧急更新
3. 归档提案发送给 Owner 和研发总监
4. **不自动执行归档**，必须经过人工确认
5. 如果 Owner 确认归档，则执行归档流程（更新状态为 Archived）
6. 如果 Owner 承诺更新，则给予7天宽限期

### 3.4 Sprint Review 中的文档鲜度议程

每次 Sprint Review 必须包含一个固定议程环节（约5-10分钟）：

**文档健康度仪表盘展示**（由 Skill-H 驱动）：

```
+----------------------------------------------------------+
|              文档健康度 Sprint Review Dashboard            |
+----------------------------------------------------------+
|  总文档数: 42    新鲜: 35 (83%)   临近: 4 (10%)           |
|                  过期: 2 (5%)     腐烂: 1 (2%)            |
+----------------------------------------------------------+
|  需要关注的文档:                                           |
|  [红] CA-2026-003 (推荐系统协议)  Owner: WangWu  过期12天  |
|  [红] EXP-005 (LLM摘要实验)      Owner: ZhaoLiu 过期5天   |
|  [灰] PRD-2025-008 (旧搜索需求)  Owner: UNOWNED 腐烂45天  |
+----------------------------------------------------------+
|  Owner 健康度排名:                                         |
|  ZhangSan: 95%  LiSi: 88%  WangWu: 62%  ZhaoLiu: 71%    |
+----------------------------------------------------------+
```

---

## 4. 个人文档健康度评分

### 4.1 评分计算方式

每位 Owner 的文档健康度评分计算方式：

```
个人文档健康度 = Sum(每份文档的鲜度得分) / 负责文档总数

单份文档的鲜度得分:
  - 新鲜 (Fresh):     100分
  - 临近 (Due Soon):   80分
  - 过期 (Overdue):    40分
  - 腐烂 (Decayed):     0分
```

### 4.2 使用原则

文档健康度评分的使用遵循以下原则：

1. **公开透明**：在 Sprint Review 中展示，所有团队成员可见
2. **非直接考核**：不作为 KPI 直接考核指标，避免为了"刷分"而进行无意义更新
3. **管理参考**：作为研发总监的管理参考工具，用于识别潜在的流程风险
4. **趋势监控**：关注趋势而非绝对值——持续下降的健康度比偶尔的低分更值得关注
5. **正向激励**：可设立"文档先锋"月度表彰（非物质奖励），鼓励持续维护

### 4.3 研发总监的一键查看

研发总监可以通过 Skill-H 一键查看每位 Owner 的文档健康度：

```bash
wc dashboard owner-health
```

输出示例：

```
Owner 文档健康度报告 (2026-04-23)
===================================
 Owner      文档数  新鲜  临近  过期  腐烂  健康度
-----------------------------------
 ZhangSan     6     5     1     0     0    97%
 LiSi         8     6     1     1     0    85%
 WangWu       5     2     1     1     1    56%  !!
 ZhaoLiu      4     3     0     1     0    85%
===================================
 团队平均                                   81%
```
