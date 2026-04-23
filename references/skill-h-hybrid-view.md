# Skill-H: 混合视图生成器 (WisdomCore-HybridView)

> 本文档为 Skill-H 的完整参考手册，涵盖全景仪表盘六面板设计、依赖图、
> 健康度评分模型、僵尸文档检测、智能报告生成及异常检测。

---

## 1. Skill 概况

| 属性 | 内容 |
|------|------|
| 名称 | **WisdomCore-HybridView** |
| Skill ID | `wisdomcore-h` |
| 目标用户 | 研发总监、产品总监、技术负责人、项目经理 |
| 核心价值 | 上帝视角看清混合研发全局状态——哪些功能卡在 AI 上、哪些协议快到期、哪些文档已成僵尸 |
| 输入 | SQLite 索引数据 + Markdown 文档内容 |
| 输出 | 结构化 Markdown 报告、Mermaid 依赖图、健康度评分、异常预警列表 |
| 依赖 | `wisdomcore.db`（SQLite 数据库）、所有文档目录 |
| 部署阶段 | Phase 2（Skill-S 和 Skill-A 上线 2-3 周后部署，作为数据消费者） |

**设计哲学**：管理者最怕两件事——"不知道全貌"和"出了问题才知道"。Skill-H 不生产
数据，只消费 Skill-S 和 Skill-A 产生的数据，将其转化为管理者能快速理解和决策的
可视化报告。所有输出均为 Markdown 格式（含 Mermaid 图表），可在任何编辑器或
Git 平台中渲染查看。

---

## 2. 全景仪表盘（Panoramic Dashboard）— 六大面板

用户可通过自然语言指令呼出（如"给我看看全局状态"、"系统健康度怎么样"）。

### 面板 1: 协作协议状态总览

以表格形式呈现各状态的协议数量和占比：

| 状态 | 数量 | 占比 | 趋势(vs 上周) |
|------|------|------|---------------|
| Draft (草稿) | N | X% | +/-N |
| Active (生效) | N | X% | 持平/变化 |
| InViolation (违约) | N | X% | 重点关注 |
| UnderReview (评审中) | N | X% | -- |
| Renegotiating (重新协商) | N | X% | -- |
| Superseded (已取代) | N | X% | -- |
| Archived (归档) | N | X% | -- |

自动预警：当 InViolation 数量增加时标注需关注的具体协议 ID。

### 面板 2: 文档鲜度热力图

按文档类型和更新时间分组，用标识表示鲜度：
- 新鲜 (<7 天) / 正常 (7-30 天) / 偏旧 (30-60 天) / 过期 (>60 天)
- 覆盖类型：PRD、CA、EXP、ADR、GTM
- 自动列出过期文档的 ID 及过期天数

### 面板 3: 依赖关系图

使用 Mermaid 语法生成可视化依赖网络（详见第 3 节）。

### 面板 4: 阻塞点列表

自动检测并列出当前系统中的阻塞点：

| 优先级 | 阻塞描述 | 阻塞原因 | 影响范围 | 建议行动 | 责任人 |
|-------|---------|---------|---------|---------|--------|
| P0 | 功能被阻塞 | 协议违约/实验未达标 | 影响发版 | 加速实验或调整目标 | Owner |
| P1 | PRD 无对应协议 | 团队尚未发起 | 影响规划 | 使用 Skill-S 创建 | Owner |

### 面板 5: 里程碑追踪

以文本进度条展示各 AI 能力的计划 vs 实际进度：

```
语义搜索能力 (CA-2026-001)
计划: ████████████████████░░░░ 80% (目标: 2026-05-15)
实际: ██████████████████████░░ 90% (预计: 2026-05-10) 超前

智能推荐能力 (CA-2026-003)
计划: ████████████░░░░░░░░░░░░ 50% (目标: 2026-06-01)
实际: ████████░░░░░░░░░░░░░░░░ 35% (预计: 2026-06-20) 滞后 19 天
```

### 面板 6: 核心度量指标卡片

六大核心指标的实时状态：当前值、上周值、趋势方向、健康度标识。

---

## 3. 全景依赖图（Panoramic Dependency Graph）

这是 Skill-H 最复杂也最有价值的功能，提供整个混合研发体系的拓扑结构可视化。

### 3.1 节点类型定义

| 节点 | 符号 | 数据源 | 状态映射 |
|------|------|--------|---------|
| PRD (SaaS 需求) | 蓝色 | `01_SaaS_Requirements/` | Draft / Active / Done |
| EXP (AI 实验) | 绿色 | `02_AI_Exploration/experiments/` | Running / Completed / Failed / Superseded |
| CA (协作协议) | 黄色 | `03_Collaboration_Agreements/` | Draft / Active / InViolation / Archived |
| ADR (决策记录) | 紫色 | `02_AI_Exploration/decisions/` | Proposed / Accepted / Deprecated |
| GTM (上市资产) | 棕色 | `04_GTM_Knowledge/` | Draft / UnderReview / Approved / Published |

### 3.2 边类型定义

| 边类型 | 含义 | Mermaid 线型 | 典型用例 |
|-------|------|-------------|---------|
| `requires` | A 需要 B 提供的能力 | 实线箭头 `-->` | PRD requires CA |
| `validates` | A 验证了 B 的可行性 | 虚线箭头 `-.->` | EXP validates CA |
| `derives` | A 是从 B 衍生/产出的 | 点线箭头 `-..->` | ADR derives EXP |
| `generates` | A 产生了 B | 粗线箭头 `==>` | CA generates GTM |

### 3.3 节点颜色编码规则

```python
def get_node_color(doc):
    """根据文档状态和鲜度决定节点颜色"""
    if doc.status in ('Archived', 'Superseded', 'Deprecated'):
        return 'gray'        # 灰色：已归档
    if doc.status == 'InViolation':
        return 'red'         # 红色：违约/阻塞
    if doc.status == 'Failed':
        return 'red'         # 红色：实验失败

    days_since_update = (today - doc.updated_at).days
    if days_since_update > 60:
        return 'red'         # 红色：严重过期
    if days_since_update > 30:
        return 'yellow'      # 黄色：即将过期

    if doc.type == 'agreement':
        review_gap = (doc.next_review_date - today).days
        if review_gap <= 3:
            return 'yellow'  # 黄色：评审临近

    return 'green'           # 绿色：一切正常
```

### 3.4 图生成查询

```sql
-- 获取所有需要展示的节点
SELECT id, title, type, status, updated_at
FROM documents
WHERE status NOT IN ('Archived', 'Superseded')
  OR updated_at > date('now', '-90 days');

-- 获取所有边关系
SELECT source_id, target_id, relation_type
FROM dependencies
WHERE source_id IN (上述节点列表) OR target_id IN (上述节点列表);
```

输出为 Mermaid `graph` 代码块，可在支持 Mermaid 的编辑器或 Git 平台中渲染。

---

## 4. 健康度评分模型（Health Score Model）

### 4.1 评分公式

```
Total Health Score = Σ(metric_ratio × weight) × 100, where metric_ratio ∈ [0, 1]
```

> **说明**：每个指标的 `metric_ratio` 取值范围为 0 到 1（0 表示最差，1 表示最佳），
> 所有指标的 `weight` 之和等于 1.0，最终结果乘以 100 转换为百分制分数。

### 4.2 六大指标及权重

| 指标 | 权重 | 计算公式 |
|------|------|---------|
| 协作协议覆盖率 | 25% | 有协议的 AI 功能数 / 总 AI 功能数 |
| 协议合规率 | 25% | Active 协议数 / (Active + InViolation) |
| 文档鲜度 | 20% | 30 天内更新的非归档文档数 / 总非归档文档数 |
| 需求-实验关联率 | 15% | 有关联实验的 AI 需求数 / 总 AI 需求数 |
| 决策记录覆盖率 | 10% | 有 ADR 的重要决策数 / 总重要决策数 |
| GTM 就绪率 | 5% | 有 GTM 资料的已发布功能数 / 总已发布功能数 |

#### 僵尸文档数 (ZDC) 归一化规则

僵尸文档数 (Zombie Document Count) 作为健康度的补充修正因子，归一化公式如下：

```
ZDC_ratio = max(0, 1 - ZDC_count / 8)
```

其中 8 为危险阈值。0 个僵尸文档时 ZDC_ratio = 1.0（完美），8 个及以上僵尸文档时 ZDC_ratio = 0.0。
该比率可作为健康度评分的惩罚因子或独立预警指标使用。

### 4.3 健康等级映射

| 等级 | 分数范围 | 含义 | 建议行动 |
|------|---------|------|---------|
| A | >= 90 | 优秀——体系运转良好 | 保持节奏，持续优化 |
| B | 75 ~ 89 | 良好——有小问题需关注 | 处理预警项，定期检查 |
| C | 60 ~ 74 | 一般——存在明显短板 | 重点补齐薄弱环节 |
| D | 40 ~ 59 | 较差——多个环节失控 | 召集专项会议，制定改进计划 |
| F | < 40 | 危险——体系几近失效 | 紧急干预，回归基本流程 |

### 4.4 评分输出示例

```markdown
## 系统健康度评分报告

**总分：76.5 / 100 (等级 B)**

| 指标 | 得分 | 权重 | 加权得分 | 状态 |
|------|------|------|---------|------|
| 协作协议覆盖率 | 85.7% | 25% | 21.4 | 健康 |
| 协议合规率 | 80.0% | 25% | 20.0 | 预警 |
| 文档鲜度 | 72.5% | 20% | 14.5 | 预警 |
| 需求-实验关联率 | 90.0% | 15% | 13.5 | 健康 |
| 决策记录覆盖率 | 66.7% | 10% | 6.7 | 预警 |
| GTM 就绪率 | 50.0% | 5% | 2.5 | 偏低 |

**主要风险**：
1. GTM 就绪率偏低（50%），已发布功能缺少市场资料
2. 2 个协议处于违约状态，拉低合规率
3. 3 份文档超过 60 天未更新

**改进建议**：
1. 【紧急】处理违约协议
2. 【本周】使用 Skill-G 为已发布功能生成 GTM 资料
3. 【持续】建立文档定期更新机制
```

---

## 5. 僵尸文档检测（Zombie Document Detection）

### 5.1 定义

状态不在 `{Archived, Superseded, Deprecated}` 中，且 `updated_at` 距今超过
60 天的文档。这些文档既没有被正式归档，也长期没人维护，是管理的灰色地带。

### 5.2 检测查询

```sql
SELECT id, title, type, status, owner, updated_at,
       julianday('now') - julianday(updated_at) AS days_stale
FROM documents
WHERE status NOT IN ('Archived', 'Superseded', 'Deprecated')
  AND julianday('now') - julianday(updated_at) > 60
ORDER BY days_stale DESC;
```

### 5.3 输出格式

| # | 文档 ID | 标题 | 类型 | 状态 | 责任人 | 已过期天数 | 建议操作 |
|---|---------|------|------|------|--------|-----------|---------|
| 1 | CA-2026-002 | 智能客服问答能力 | CA | Active | 王五 | 75 天 | 刷新或归档 |
| 2 | GTM-Feature-推荐 | 推荐功能介绍 | GTM | Draft | 赵六 | 90 天 | 归档（功能已变更） |
| 3 | EXP-2026-002 | 初版推荐实验 | EXP | Completed | 李华 | 82 天 | 归档（已被取代） |

### 5.4 建议操作分类

- **刷新 (Update)** — 文档内容仍有效，需要更新到最新状态
- **归档 (Archive)** — 文档已过时或被替代，应标记为 Archived/Superseded
- **升级 (Escalate)** — 无法判断文档状态，需联系责任人确认

---

## 6. 智能报告生成（Smart Report Generation）

### 6.1 周报模板

```markdown
# WisdomCore 周报 (日期范围)

## 本周概要
- 新增文档：N 份（分类明细）
- 更新文档：N 份
- 关闭/归档：N 份
- 健康度评分：XX.X (等级) <- 上周 XX.X，变化 +/-

## 协作协议动态
- 新建协议、违约协议、即将到期评审

## 实验进展
- 新启动实验、已完成实验、达标情况
- Bad Case 累计统计：Open 数 / 本周新增 / 本周 Resolved

## 阻塞点
- P0/P1 级阻塞及原因

## 下周关注
- 关键事件预告
```

### 6.2 月报模板

在周报基础上增加：
- **健康度趋势**：各周评分与等级变化表
- **本月成就**：关键里程碑完成情况
- **待改进项**：长期偏低的指标分析
- **下月建议**：优先改进事项

### 6.3 专题报告

针对特定产品线或团队的深度分析，按用户指定维度生成。

---

## 7. 异常检测与预警（Anomaly Detection & Alert）

Skill-H 持续监测以下 8 种异常模式，发现后立即在报告中高亮提示。

### 7.1 异常模式清单

| # | 异常类型 | 检测条件 | 严重程度 | 建议行动 |
|---|---------|---------|---------|---------|
| 1 | 协议即将过期 | `next_review_date` 距今 <= 3 天 | Medium | 准备评审材料 |
| 2 | 实验超期运行 | `status='Running'` 且 `created_at` 距今 > 30 天 | Medium | 确认实验是否仍在进行 |
| 3 | 孤立 PRD | PRD 无关联 CA | Info | 如含 AI 需求请创建协议 |
| 4 | 孤立 CA | CA 无关联 EXP | Medium | AI 团队可能尚未开始探索 |
| 5 | 单人过载 | 某人 own 的活跃文档数 > 10 | Medium | 建议分担工作负载 |
| 6 | 循环依赖 | `dependencies` 图中存在环 | High | 检查文档关联是否正确 |
| 7 | 协议长期 Draft | `status='Draft'` 且 `created_at` 距今 > 14 天 | Info | 是否需要推进到 Active |
| 8 | 连续失败实验 | 同一 CA 下连续 3 个 EXP 为 Failed/Reject | High | 评估方向可行性 |

### 7.2 循环依赖检测算法

```python
def detect_cycles():
    graph = build_adjacency_list_from_dependencies()
    visited = set()
    rec_stack = set()
    path = []
    cycles = []
    def dfs(node):
        visited.add(node)
        rec_stack.add(node)
        path.append(node)
        for neighbor in graph.get(node, []):
            if neighbor not in visited:
                dfs(neighbor)
            elif neighbor in rec_stack:
                idx = path.index(neighbor)
                cycles.append(path[idx:] + [neighbor])
        path.pop()
        rec_stack.remove(node)
    for node in graph:
        if node not in visited:
            dfs(node)
    return cycles
```

---

## 8. 技术实现要点

| 要点 | 实现方式 | 原因 |
|------|---------|------|
| 数据查询 | 全部通过 SQLite 查询 | 性能保障，5-20 人团队规模足够 |
| 图表可视化 | Mermaid 语法 | 无需额外依赖，GitHub/GitLab/VS Code 原生支持 |
| 进度条 | 文本字符绘制 | 纯文本兼容性最佳 |
| 报告格式 | 标准 Markdown | 可纳入 Git 版本管理 |
| 报告存储 | `05_Reports/` 目录 | 历史报告可追溯 |
| 无 UI 框架 | 不使用 Streamlit/Gradio | Skill-H 是 QoderWork Skill，输出为文本/Markdown |

---

## 9. 与其他 Skill 的交互

| 交互方向 | 交互内容 | 数据源 |
|---------|---------|--------|
| H <- 父 Skill | 读取 SQLite 索引获取全局数据 | `wisdomcore.db` |
| H <- Skill-S | 消费协作协议数据（状态、评审日期等） | `documents` 表 WHERE `type='agreement'` |
| H <- Skill-A | 消费实验数据（状态、关键指标、Bad Case 统计） | `documents` 表 + `bad_cases` 表 |
| H -> 用户 | 输出各类报告和分析结果 | Markdown 文本输出 |
| H -> Skill-G | 提供健康度数据和文档状态摘要 | 健康度评分 API |
| H -> 父 Skill | 报告写入 `05_Reports/` 目录 | Markdown 文件 |
