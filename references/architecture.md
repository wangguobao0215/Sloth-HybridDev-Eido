# 技术架构参考手册

> 本文档为 WisdomCore 技术架构的操作级参考，提炼自设计方案第三章。
> AI Agent 在执行初始化、校验、查询等操作时应参照本文档。

---

## 1. Markdown + SQLite 双写架构

### 1.1 设计哲学

智核（WisdomCore）采用**人机双通道**架构——Markdown 面向人类，SQLite 面向机器。

| 维度 | Markdown（人类通道） | SQLite（机器通道） |
|------|----------------------|---------------------|
| 编辑体验 | 任意编辑器、离线可用、Git diff 友好 | 不适合直接编辑 |
| 版本控制 | Git 原生支持、PR Review 天然适配 | 二进制文件，Git diff 无意义 |
| 结构化查询 | 极差，需全文解析 | 极强，SQL 任意组合 |
| 聚合统计 | 不可能 | `GROUP BY`、`COUNT`、窗口函数 |
| AI Agent 消费 | 需要解析 Frontmatter | 直接 SQL 查询，毫秒级响应 |
| 跨文档关联 | 手动维护链接 | `JOIN` + 外键约束 |
| 离线可用性 | 完全离线 | 完全离线（SQLite 是嵌入式数据库） |

### 1.2 核心原则

**Markdown 永远是 Single Source of Truth。SQLite 是派生索引，可随时从 Markdown 全量重建。**

- 任何写操作必须先写 Markdown，再同步 SQLite
- SQLite 损坏或丢失时，执行 `wisdomcore rebuild-index` 即可完全恢复
- 冲突解决策略：Markdown 内容优先，SQLite 被覆盖

### 1.3 同步机制

**路径一：Pre-commit Hook（推荐，保证一致性）**
在 `git commit` 时自动触发 Frontmatter 校验 + SQLite 同步。每次提交时 SQLite 与 Markdown 保持一致。

**路径二：On-save Watcher（可选，提升开发体验）**
文件保存时实时同步 SQLite，用于本地开发时的即时查询。不保证与 Git 状态一致，仅作为辅助。

### 1.4 架构流程图

```
用户/AI Agent → 编辑 Markdown(Frontmatter) → git add → Stage Area
                        |                                  |
                on-save watcher(可选)               git commit
                        |                                  |
                        v                                  v
                  实时索引更新(非权威)             Pre-commit Hook 触发
                        |                          1. JSON Schema 校验
                        |                          2. 解析 Frontmatter
                        |                          3. Upsert SQLite
                        |                                  |
                        v                                  v
                    .wisdomcore.db (SQLite)
                    ├── documents       ├── dependencies
                    ├── status_history  ├── metrics_snapshots
                    ├── change_log      └── bad_cases
                              |
                    +---------+---------+
                    v                   v
              AI Agent SQL 查询    Dashboard 可视化报表
```

### 1.5 全量重建脚本

当 SQLite 数据库损坏或需要从零重建时执行：

```bash
#!/usr/bin/env bash
# wisdomcore-rebuild-index.sh
# 从 Markdown 文件全量重建 SQLite 索引

set -euo pipefail

WISDOMCORE_ROOT="${WISDOMCORE_ROOT:-.}"
DB_PATH="${WISDOMCORE_ROOT}/.wisdomcore.db"

echo "=== WisdomCore 索引全量重建 ==="
echo "根目录: ${WISDOMCORE_ROOT}"

# 1. 备份现有数据库（如存在）
if [ -f "${DB_PATH}" ]; then
    BACKUP_PATH="${DB_PATH}.bak.$(date +%Y%m%d%H%M%S)"
    cp "${DB_PATH}" "${BACKUP_PATH}"
    echo "已备份旧数据库至: ${BACKUP_PATH}"
    rm "${DB_PATH}"
fi

# 2. 创建新数据库并初始化 Schema
echo "正在初始化数据库 Schema..."
sqlite3 "${DB_PATH}" < "${WISDOMCORE_ROOT}/.meta/schema.sql"

# 3. 遍历所有 Markdown 文件，解析 Frontmatter 并写入
echo "正在扫描 Markdown 文件..."
TOTAL=0; SUCCESS=0; FAILED=0

find "${WISDOMCORE_ROOT}" -name "*.md" \
    -not -path "*/.git/*" \
    -not -path "*/.meta/*" \
    -not -path "*/node_modules/*" \
    -print0 | while IFS= read -r -d '' md_file; do

    TOTAL=$((TOTAL + 1))
    if python3 "${WISDOMCORE_ROOT}/.meta/scripts/sync_frontmatter.py" \
        --db "${DB_PATH}" \
        --file "${md_file}" \
        --root "${WISDOMCORE_ROOT}" 2>/dev/null; then
        SUCCESS=$((SUCCESS + 1))
    else
        FAILED=$((FAILED + 1))
        echo "  [WARN] 解析失败: ${md_file}"
    fi
done

echo ""
echo "=== 重建完成 ==="
echo "总文件数: ${TOTAL} | 成功: ${SUCCESS} | 失败: ${FAILED}"
echo "数据库路径: ${DB_PATH}"
```

> **sqlite-utils 重写版（推荐）：** 上述脚本的核心同步逻辑可通过 sqlite-utils 大幅简化：
>
> ```bash
> # 从 Frontmatter JSON 批量导入
> python3 -c "
> import sqlite_utils, yaml, pathlib, json
> db = sqlite_utils.Database('.wisdomcore.db')
> for md in pathlib.Path('.').rglob('*.md'):
>     text = md.read_text()
>     if text.startswith('---'):
>         fm = yaml.safe_load(text.split('---')[1])
>         if fm and 'id' in fm:
>             fm['file_path'] = str(md)
>             fm['frontmatter_json'] = json.dumps(fm, ensure_ascii=False)
>             db['documents'].upsert(fm, pk='id')
> "
> ```

---

## 2. 推荐工具链 (Recommended Toolchain)

WisdomCore 不重复造轮子。以下开源工具经过评估，与 Markdown + SQLite 双写架构天然契合，作为官方推荐工具链。

### 2.1 核心层（P0 — 架构基石）

| 工具 | 版本要求 | 用途 | 安装 |
|------|---------|------|------|
| **sqlite-utils** | ≥ 3.35 | SQLite 读写的 Python CLI + 库。替代自研的 `sync_frontmatter.py` 核心逻辑，提供 `insert`, `upsert`, `query` 等原子操作 | `pip install sqlite-utils` |
| **remark-lint-frontmatter-schema** | ≥ 3.0 | Markdown Frontmatter 按 JSON Schema 校验。替代自研 `validate_frontmatter.py`，支持行号定位错误 | `npm install remark-cli remark-lint-frontmatter-schema` |

### 2.2 增强层（P1 — 显著提升体验）

| 工具 | 版本要求 | 用途 | 安装 |
|------|---------|------|------|
| **Datasette** | ≥ 0.65 | 为 `.wisdomcore.db` 提供只读 Web UI + JSON API。Skill-H 仪表盘的自助服务模式 | `pip install datasette` |
| **phodal/adr** | ≥ 3.0 | ADR CLI 工具（中文友好）。支持 `adr new`, `adr list`, `adr generate toc`, CSV 导出 | `npm install -g adr` |
| **git-cliff** | ≥ 2.0 | 从 Git commit 历史自动生成 Keep a Changelog 格式的 CHANGELOG.md | `cargo install git-cliff` 或 `brew install git-cliff` |

### 2.3 可视化层（P2 — 丰富输出）

| 工具 | 版本要求 | 用途 | 安装 |
|------|---------|------|------|
| **Mermaid CLI (mmdc)** | ≥ 11.0 | 将 `.mermaid` 依赖图转为 SVG/PNG，嵌入报告和文档 | `npm install -g @mermaid-js/mermaid-cli` |
| **markdownlint-cli** | ≥ 0.41 | Markdown 风格检查（标题层级、列表缩进、空行规范），集成 pre-commit | `npm install -g markdownlint-cli` |

### 2.4 观望层（v2.0 候选）

| 工具 | 说明 |
|------|------|
| **Trackio** (HuggingFace) | 轻量级 ML 实验追踪。当团队需要管理模型训练实验时引入，作为 Skill-A 的扩展层 |
| **DVC** (Data Version Control) | Git 原生的大文件版本控制。当 Skill-A 需追踪模型权重/数据集时引入 |
| **memweave / sqlite-memory** | Agent 记忆层。参考其 Markdown 日记 + SQLite 索引架构，为 WisdomCore 增加 `.meta/memory/` 交互日志目录 |

### 2.5 工具链集成架构

```
用户/AI Agent
    |
    v
编辑 Markdown (Frontmatter + 正文)
    |
    v
git add → git commit
    |
    +--→ [Pre-commit Hook Pipeline]
    |       |
    |       ├── markdownlint-cli     → Markdown 风格检查
    |       ├── remark-lint          → Frontmatter JSON Schema 校验
    |       ├── sqlite-utils upsert  → Frontmatter → SQLite 同步
    |       └── ADR 删除拦截          → 禁止删除已合并 ADR
    |
    +--→ [Post-commit / Tag Hook]
    |       └── git-cliff            → 自动生成 CHANGELOG.md
    |
    v
.wisdomcore.db (SQLite)
    |
    ├── sqlite-utils query   → CLI 查询（脚本 / Agent）
    ├── Datasette serve      → Web UI 自助仪表盘 (http://localhost:8001)
    └── mmdc                 → 依赖图 SVG/PNG 导出至 05_Reports/
```

### 2.6 快速安装

```bash
# 核心层（必装）
pip install sqlite-utils
npm install -g remark-cli remark-lint-frontmatter-schema

# 增强层（推荐）
pip install datasette
npm install -g adr
brew install git-cliff   # macOS; 其他平台: cargo install git-cliff

# 可视化层（可选）
npm install -g @mermaid-js/mermaid-cli markdownlint-cli
```

---

## 3. 目录结构

```
WisdomCore-Root/
├── .meta/                                    # 系统元数据与配置（不含业务文档）
│   ├── config.yaml                           # 全局配置：团队名、产品线、默认审批人等
│   ├── team-roster.yaml                      # 团队白名单：成员列表、角色、权限
│   ├── tag-taxonomy.yaml                     # 标签体系定义：二级分类 + 治理规则
│   ├── schema.sql                            # SQLite 完整建表语句
│   ├── schema/                               # Frontmatter JSON Schema 校验文件
│   │   ├── prd.schema.json
│   │   ├── experiment.schema.json
│   │   ├── agreement.schema.json             # 最复杂
│   │   ├── adr.schema.json
│   │   └── gtm.schema.json
│   ├── scripts/                              # 自动化脚本
│   │   ├── sync_frontmatter.py               # Frontmatter -> SQLite 同步
│   │   ├── validate_frontmatter.py           # Frontmatter JSON Schema 校验
│   │   ├── rebuild_index.sh                  # 全量重建 SQLite 索引
│   │   ├── generate_dashboard.py             # 从 SQLite 生成仪表盘数据
│   │   ├── metrics_snapshot.py             # 每周自动采集 6 大核心指标
│   │   └── serve_dashboard.sh              # 启动 Datasette Web UI: datasette .wisdomcore.db
│   ├── hooks/                                # Git Hook 脚本
│   │   ├── pre-commit                        # Pre-commit: 校验 + 同步
│   │   └── commit-msg                        # Commit message 格式校验
│   └── templates/                            # 文档模板（新建文档时复制）
│       ├── prd-template.md
│       ├── experiment-template.md
│       ├── agreement-template.md
│       ├── adr-template.md
│       ├── release-note-template.md
│       ├── battle-card-template.md
│       └── feature-brief-template.md
│   └── memory/                            # Agent 交互日志（memweave 架构启发）
│       └── {YYYY-MM-DD}-session.md        # 每日会话摘要
│
├── 01_SaaS_Requirements/                     # SaaS需求池
│   ├── {YYYY}-Q{N}/                          # 按季度组织
│   │   └── PRD-{slug}.md
│   └── _backlog/                             # 未排期需求
│
├── 02_AI_Exploration/                        # AI探索池
│   ├── experiments/                          # 实验报告 (EXP-YYYY-NNN-{slug}.md)
│   └── decisions/                            # 架构决策记录 (ADR-NNN-{slug}.md)
│
├── 03_Collaboration_Agreements/              # 协作协议库 (CA-YYYY-NNN-{slug}.md)
│
├── 04_GTM_Knowledge/                         # GTM知识资产
│   ├── release-notes/                        # RN-{version}.md
│   ├── battle-cards/                         # BC-{competitor-slug}.md
│   └── feature-briefs/                       # FB-{feature-slug}.md
│
├── 05_Reports/                               # 报告区（度量快照、健康度报告等）
│
├── 06_Archive/                               # 归档区（已关闭/废弃的文档）
│   ├── agreements/
│   ├── experiments/
│   └── requirements/
│
├── .wisdomcore.db                            # SQLite 索引数据库（Git ignored）
├── .gitignore                                # 忽略 .wisdomcore.db、*.bak 等
└── .git/
```

**目录设计原则：**

1. **数字前缀排序**：`01_` ~ `06_` 确保文件管理器中自然排序
2. **按季度/编号组织**：需求按季度，实验/协议按编号，GTM 按子类型
3. **归档分离**：已关闭文档移入 `06_Archive/`，保持活跃区清洁
4. **元数据隔离**：`.meta/` 存放所有系统级配置，不与业务文档混合
5. **SQLite 在根目录**：`.wisdomcore.db` 放根目录方便脚本定位，加入 `.gitignore`

---

## 4. SQLite Schema

```sql
-- ============================================================================
-- WisdomCore SQLite Schema v1.0.0
-- Markdown Frontmatter 为权威数据源，SQLite 为派生索引
-- ============================================================================

PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;

-- 1. 文档主表 (documents)
CREATE TABLE documents (
    id              TEXT PRIMARY KEY,           -- CA-2026-001, PRD-2026-001, EXP-2026-001 等
    type            TEXT NOT NULL,              -- 文档类型
    file_path       TEXT NOT NULL UNIQUE,       -- 相对于 WisdomCore-Root 的路径
    title           TEXT NOT NULL,
    status          TEXT NOT NULL,              -- 各类型状态机不同
    priority        TEXT,                       -- P0-P3，仅 PRD 使用
    saas_owner      TEXT,
    ai_owner        TEXT,
    owner           TEXT,                       -- 通用 Owner（PRD/EXP/GTM/ADR 使用）
    subtype         TEXT,                       -- GTM 子类型: ReleaseNote | BattleCard | FeatureBrief
    team            TEXT DEFAULT 'default',
    product_line    TEXT DEFAULT 'default',
    risk_level      TEXT,                       -- High | Medium | Low
    tags            TEXT DEFAULT '[]',          -- JSON 数组
    created_at      TEXT NOT NULL,              -- ISO-8601
    updated_at      TEXT NOT NULL,              -- ISO-8601
    next_review_date TEXT,
    target_release  TEXT,                       -- PRD/GTM 使用
    version         INTEGER DEFAULT 1,
    frontmatter_json TEXT NOT NULL,             -- 完整 Frontmatter JSON 序列化
    content_hash    TEXT,                       -- 正文 SHA-256，变更检测
    CONSTRAINT valid_type CHECK (
        type IN ('PRD', 'Experiment', 'CollaborationAgreement', 'ADR', 'GTM')
    ),
    CONSTRAINT valid_priority CHECK (
        priority IS NULL OR priority IN ('P0', 'P1', 'P2', 'P3')
    ),
    CONSTRAINT valid_risk CHECK (
        risk_level IS NULL OR risk_level IN ('High', 'Medium', 'Low')
    )
);

-- 2. 依赖关系表 (dependencies)
CREATE TABLE dependencies (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    source_id       TEXT NOT NULL,
    target_id       TEXT NOT NULL,
    relation_type   TEXT NOT NULL,
    description     TEXT,
    created_at      TEXT NOT NULL DEFAULT (datetime('now')),
    FOREIGN KEY (source_id) REFERENCES documents(id) ON DELETE CASCADE,
    FOREIGN KEY (target_id) REFERENCES documents(id) ON DELETE CASCADE,
    CONSTRAINT valid_relation CHECK (
        relation_type IN (
            'depends_on',    -- A 依赖 B（B 是 A 的前置条件）
            'implements',    -- A 实现了 B（实验实现了需求）
            'governed_by',   -- A 受 B 约束（需求受协议约束）
            'supersedes',    -- A 替代了 B（新协议替代旧协议）
            'references',    -- A 引用了 B（一般性引用）
            'blocks',        -- A 阻塞 B
            'informs',       -- A 为 B 提供信息（ADR 通知 PRD）
            'requires',      -- A 要求 B（强依赖）
            'validates',     -- A 验证 B（测试/实验验证需求）
            'derives'        -- A 派生自 B（从 B 衍生）
        )
    ),
    CONSTRAINT no_self_ref CHECK (source_id != target_id),
    UNIQUE(source_id, target_id, relation_type)
);

-- 3. 状态变更历史表 (status_history)
CREATE TABLE status_history (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    document_id     TEXT NOT NULL,
    from_status     TEXT,                       -- 首次创建时为 NULL
    to_status       TEXT NOT NULL,
    changed_by      TEXT NOT NULL,
    change_reason   TEXT,
    changed_at      TEXT NOT NULL DEFAULT (datetime('now')),
    commit_sha      TEXT,
    document_type   TEXT,                       -- for fast filtering
    violation_severity TEXT,                    -- for violation tracking
    metadata_json   TEXT,                       -- for PR links etc.
    FOREIGN KEY (document_id) REFERENCES documents(id) ON DELETE CASCADE
);

-- 4. 度量快照表 (metrics_snapshots)
CREATE TABLE metrics_snapshots (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    snapshot_date   TEXT NOT NULL,              -- YYYY-MM-DD
    metric_name     TEXT NOT NULL,
    metric_value    REAL NOT NULL,
    dimension       TEXT DEFAULT 'global',     -- global | team:{name} | product_line:{name}
    metadata_json   TEXT,
    created_at      TEXT NOT NULL DEFAULT (datetime('now')),
    UNIQUE(snapshot_date, metric_name, dimension)
);
-- 预定义指标: ACR(协议合规率), ACMR(协议违约率), DF(文档新鲜度),
-- RELR(发布就绪率), GTT(GTM 上线时间), ZDC(零漂移覆盖率)

-- 5. 变更日志表 (change_log)
CREATE TABLE change_log (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    document_id     TEXT NOT NULL,
    field_name      TEXT NOT NULL,
    old_value       TEXT,
    new_value       TEXT,
    changed_by      TEXT NOT NULL,
    changed_at      TEXT NOT NULL DEFAULT (datetime('now')),
    commit_sha      TEXT,
    FOREIGN KEY (document_id) REFERENCES documents(id) ON DELETE CASCADE
);

-- 6. Bad Case 注册表 (bad_cases)
CREATE TABLE bad_cases (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    case_id         TEXT NOT NULL UNIQUE,       -- BC-YYYY-NNN
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

### 4.1 索引设计

```sql
-- 文档主表索引
CREATE INDEX idx_documents_type          ON documents(type);
CREATE INDEX idx_documents_status        ON documents(status);
CREATE INDEX idx_documents_team          ON documents(team);
CREATE INDEX idx_documents_product_line  ON documents(product_line);
CREATE INDEX idx_documents_priority      ON documents(priority);
CREATE INDEX idx_documents_risk_level    ON documents(risk_level);
CREATE INDEX idx_documents_updated_at    ON documents(updated_at);
CREATE INDEX idx_documents_next_review   ON documents(next_review_date);
CREATE INDEX idx_documents_saas_owner    ON documents(saas_owner);
CREATE INDEX idx_documents_ai_owner      ON documents(ai_owner);
CREATE INDEX idx_documents_type_status   ON documents(type, status);        -- 最常用
CREATE INDEX idx_documents_team_pl       ON documents(team, product_line);  -- 多团队
CREATE INDEX idx_documents_type_team     ON documents(type, team);

-- 依赖关系表索引
CREATE INDEX idx_deps_source             ON dependencies(source_id);
CREATE INDEX idx_deps_target             ON dependencies(target_id);
CREATE INDEX idx_deps_relation           ON dependencies(relation_type);

-- 状态历史表索引
CREATE INDEX idx_status_hist_doc         ON status_history(document_id);
CREATE INDEX idx_status_hist_time        ON status_history(changed_at);

-- 变更日志索引
CREATE INDEX idx_changelog_doc           ON change_log(document_id);
CREATE INDEX idx_changelog_time          ON change_log(changed_at);
CREATE INDEX idx_changelog_field         ON change_log(document_id, field_name);

-- 度量快照索引
CREATE INDEX idx_metrics_date            ON metrics_snapshots(snapshot_date);
CREATE INDEX idx_metrics_name            ON metrics_snapshots(metric_name);
CREATE INDEX idx_metrics_dimension       ON metrics_snapshots(dimension);

-- Bad Case 索引
CREATE INDEX idx_badcases_status         ON bad_cases(status);
CREATE INDEX idx_badcases_severity       ON bad_cases(severity);
CREATE INDEX idx_badcases_category       ON bad_cases(category);
CREATE INDEX idx_badcases_team           ON bad_cases(team);
```

### 4.2 常用查询视图

```sql
-- 活跃协议视图
CREATE VIEW v_active_agreements AS
SELECT id, title, status, saas_owner, ai_owner, team, product_line,
       risk_level, next_review_date, updated_at
FROM documents
WHERE type = 'CollaborationAgreement'
  AND status IN ('Active', 'InViolation', 'Renegotiating');

-- 逾期评审文档视图
CREATE VIEW v_overdue_reviews AS
SELECT id, type, title, status, saas_owner, ai_owner, next_review_date,
       julianday('now') - julianday(next_review_date) AS overdue_days
FROM documents
WHERE next_review_date IS NOT NULL
  AND next_review_date < date('now')
  AND status NOT IN ('Archived', 'Superseded', 'Deprecated', 'InViolation', 'Renegotiating');

-- 团队文档统计视图
CREATE VIEW v_team_summary AS
SELECT team,
       COUNT(*) AS total_docs,
       SUM(CASE WHEN type = 'PRD' THEN 1 ELSE 0 END) AS prd_count,
       SUM(CASE WHEN type = 'Experiment' THEN 1 ELSE 0 END) AS exp_count,
       SUM(CASE WHEN type = 'CollaborationAgreement' THEN 1 ELSE 0 END) AS ca_count,
       SUM(CASE WHEN type = 'ADR' THEN 1 ELSE 0 END) AS adr_count,
       SUM(CASE WHEN type = 'GTM' THEN 1 ELSE 0 END) AS gtm_count,
       SUM(CASE WHEN risk_level = 'High' THEN 1 ELSE 0 END) AS high_risk_count
FROM documents
GROUP BY team;

-- 文档关联图视图
CREATE VIEW v_dependency_graph AS
SELECT d.source_id, d.target_id, d.relation_type,
       s.title AS source_title, s.type AS source_type,
       t.title AS target_title, t.type AS target_type
FROM dependencies d
JOIN documents s ON d.source_id = s.id
JOIN documents t ON d.target_id = t.id;
```

---

## 5. Frontmatter JSON Schema (CollaborationAgreement)

协作协议是系统中最复杂的文档类型，其 Schema 作为完整示例。其他类型结构更简单，可参照此规范推导。

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://wisdomcore.local/schema/agreement.schema.json",
  "title": "WisdomCore CollaborationAgreement Frontmatter",
  "type": "object",
  "required": [
    "id", "type", "title", "status", "saas_owner", "ai_owner",
    "team", "product_line", "risk_level", "acceptance_criteria",
    "fallback_strategy", "review_cadence_days", "tags",
    "created_at", "updated_at"
  ],
  "properties": {
    "id":                { "type": "string", "pattern": "^CA-\\d{4}-\\d{3}$" },
    "type":              { "const": "CollaborationAgreement" },
    "title":             { "type": "string", "minLength": 5, "maxLength": 120 },
    "status":            { "enum": ["Draft","UnderReview","Active","InViolation","Renegotiating","Superseded","Archived"] },
    "saas_owner":        { "type": "string", "minLength": 1 },
    "ai_owner":          { "type": "string", "minLength": 1 },
    "team":              { "type": "string", "pattern": "^[a-z][a-z0-9-]{1,30}$" },
    "product_line":      { "type": "string", "pattern": "^[a-z][a-z0-9-]{1,30}$" },
    "risk_level":        { "enum": ["High", "Medium", "Low"] },
    "acceptance_criteria": {
      "type": "array", "minItems": 1,
      "items": {
        "type": "object",
        "required": ["metric", "operator", "threshold"],
        "properties": {
          "metric":    { "type": "string" },
          "operator":  { "enum": [">=", "<=", ">", "<", "==", "!="] },
          "threshold": { "type": ["number", "string"] },
          "unit":      { "type": "string" },
          "measurement_method": { "type": "string" }
        }
      }
    },
    "fallback_strategy":    { "type": "string", "minLength": 10 },
    "review_cadence_days":    { "type": "integer", "minimum": 7, "maximum": 180 },
    "related_requirements": { "type": "array", "items": { "pattern": "^PRD-\\d{4}-\\d{3}$" }, "default": [] },
    "related_experiments":  { "type": "array", "items": { "pattern": "^EXP-\\d{4}-\\d{3}$" }, "default": [] },
    "supersedes":           { "type": "string", "pattern": "^CA-\\d{4}-\\d{3}$" },
    "version":              { "type": "integer", "minimum": 1, "default": 1 },
    "next_review_date":     { "type": "string", "format": "date" },
    "tags":                 { "type": "array", "minItems": 1, "items": { "pattern": "^[A-Za-z][A-Za-z0-9_-]{1,30}$" } },
    "created_at":           { "type": "string", "format": "date" },
    "updated_at":           { "type": "string", "format": "date" },
    "approvers":            { "type": "array", "items": { "type": "string" } },
    "scope":                { "type": "string", "minLength": 1 },
    "applicable_services":  { "type": "array", "items": { "type": "string" } },
    "fallback_trigger":     { "type": "string" },
    "fallback_contact":     { "type": "string" },
    "review_checklist":     { "type": "array", "items": { "type": "string" } },
    "violation_history":    { "type": "array", "items": { "type": "object" } },
    "superseded_by":        { "type": "string", "pattern": "^CA-\\d{4}-\\d{3}$" }
  },
  "if":   { "properties": { "risk_level": { "const": "High" } } },
  "then": { "properties": { "review_cadence_days": { "maximum": 30 } } }
}
```

**条件约束**: 当 `risk_level` 为 `High` 时，`review_cadence_days` 不得超过 30 天。

---

## 6. Pre-commit Hook 脚本

> **v1.1 增强版 Pre-commit Hook** 集成了 markdownlint-cli（风格检查）和 remark-lint-frontmatter-schema（Schema 校验），替代自研的 `validate_frontmatter.py`。sqlite-utils 替代手工 SQL 写入。原始脚本保留作为 fallback。

```bash
#!/usr/bin/env bash
# .meta/hooks/pre-commit
# 功能：1) 校验 Frontmatter 格式  2) 同步更新 SQLite 索引

set -euo pipefail

WISDOMCORE_ROOT="$(git rev-parse --show-toplevel)"
DB_PATH="${WISDOMCORE_ROOT}/.wisdomcore.db"
SCHEMA_DIR="${WISDOMCORE_ROOT}/.meta/schema"
SCRIPTS_DIR="${WISDOMCORE_ROOT}/.meta/scripts"
ERRORS=0

echo "WisdomCore Pre-commit 校验开始..."

# 检查是否有 ADR 文件被删除（ADR 不可删除，只能归档）
DELETED_ADRS=$(git diff --cached --name-only --diff-filter=D \
    | grep -E '^02_AI_Exploration/decisions/ADR-' \
    || true)

if [ -n "${DELETED_ADRS}" ]; then
    echo "[BLOCK] ADR 文件不允许删除，请使用归档（archive）流程："
    echo "${DELETED_ADRS}"
    exit 1
fi

# 获取本次提交中变更的 Markdown 文件
CHANGED_FILES=$(git diff --cached --name-only --diff-filter=ACM \
    | grep -E '\.md$' \
    | grep -v '^\.meta/' \
    | grep -v '^06_Archive/' \
    || true)

if [ -z "${CHANGED_FILES}" ]; then
    echo "无需校验的 Markdown 文件变更"
    exit 0
fi

echo "检测到 $(echo "${CHANGED_FILES}" | wc -l | tr -d ' ') 个变更文件"

for file in ${CHANGED_FILES}; do
    FULL_PATH="${WISDOMCORE_ROOT}/${file}"
    echo ""
    echo "--- 校验: ${file} ---"

    # Step 1: 根据目录推断 Schema
    SCHEMA_FILE=""
    case "${file}" in
        01_SaaS_Requirements/*)         SCHEMA_FILE="${SCHEMA_DIR}/prd.schema.json" ;;
        02_AI_Exploration/experiments/*) SCHEMA_FILE="${SCHEMA_DIR}/experiment.schema.json" ;;
        02_AI_Exploration/decisions/*)   SCHEMA_FILE="${SCHEMA_DIR}/adr.schema.json" ;;
        03_Collaboration_Agreements/*)   SCHEMA_FILE="${SCHEMA_DIR}/agreement.schema.json" ;;
        04_GTM_Knowledge/*)             SCHEMA_FILE="${SCHEMA_DIR}/gtm.schema.json" ;;
        *)  echo "  [SKIP] 不在受管目录中: ${file}"; continue ;;
    esac

    # Step 2: 校验 Frontmatter
    if ! python3 "${SCRIPTS_DIR}/validate_frontmatter.py" \
        --file "${FULL_PATH}" --schema "${SCHEMA_FILE}"; then
        echo "  [FAIL] Frontmatter 校验失败: ${file}"
        ERRORS=$((ERRORS + 1))
        continue
    fi
    echo "  [PASS] Frontmatter 校验通过"

    # Step 3: 同步 SQLite
    if ! python3 "${SCRIPTS_DIR}/sync_frontmatter.py" \
        --db "${DB_PATH}" --file "${FULL_PATH}" --root "${WISDOMCORE_ROOT}"; then
        echo "  [WARN] SQLite 同步失败（非阻塞）: ${file}"
    else
        echo "  [SYNC] SQLite 索引已更新"
    fi
done

echo ""
if [ ${ERRORS} -gt 0 ]; then
    echo "校验失败: ${ERRORS} 个文件未通过 Frontmatter 校验，请修复后重新提交。"
    exit 1
fi

echo "WisdomCore Pre-commit 校验全部通过"
exit 0
```

### Commit-msg Hook

```bash
#!/usr/bin/env bash
# .meta/hooks/commit-msg

set -euo pipefail

COMMIT_MSG_FILE="$1"
COMMIT_MSG=$(head -1 "${COMMIT_MSG_FILE}")

PATTERN="^(feat|update|status|archive|fix|meta|refactor)\((prd|exp|ca|adr|gtm|schema|hook)\): .{5,100}$"

if ! echo "${COMMIT_MSG}" | grep -qE "${PATTERN}"; then
    echo ""
    echo "Commit message 格式错误"
    echo "期望格式: {type}({scope}): {description}"
    echo "  type:  feat | update | status | archive | fix | meta | refactor"
    echo "  scope: prd | exp | ca | adr | gtm | schema | hook"
    echo "示例: feat(ca): 新增AI代码审查SLA协议 CA-2026-004"
    echo "当前消息: ${COMMIT_MSG}"
    exit 1
fi

echo "Commit message 格式校验通过"
exit 0
```

---

## 7. Git 工作流

### 7.1 分支策略

```
main                    <-- 已审批通过的正式文档（受保护分支）
├── draft/*             <-- 草稿/WIP 文档编辑
│   ├── draft/ca-2026-004-new-sla
│   └── draft/prd-multi-tenant
├── review/*            <-- 提交评审中的文档
│   └── review/exp-004-result
└── hotfix/*            <-- 紧急修订（如协议违约修复）
│   └── hotfix/ca-2026-001-threshold-update
└── renegotiate/*       <-- 协议重新谈判（Agreement Renegotiation）
    └── renegotiate/ca-2026-002-sla-revision
```

**分支规则：**
- `main` 分支受保护，所有变更必须通过 Pull Request 合并
- `draft/*` 分支无限制，用于日常编辑
- `review/*` 分支用于正式评审流程
- `hotfix/*` 分支用于紧急修订，可加速审批
- `renegotiate/*` 分支用于协议重新谈判，需 SaaS Owner + AI Owner 参与

### 7.2 PR 审批规则

| 文档类型 | 最少审批人数 | 必须包含的审批角色 | 合并策略 |
|---------|-------------|------------------|---------|
| PRD (P0/P1) | 2 | Product Owner + Tech Lead | Squash Merge |
| PRD (P2/P3) | 1 | Product Owner 或 Tech Lead | Squash Merge |
| Experiment | 1 | AI Tech Lead | Squash Merge |
| CA (High) | 3 | SaaS Owner + AI Owner + Engineering Manager | Merge Commit |
| CA (Medium/Low) | 2 | SaaS Owner + AI Owner | Merge Commit |
| ADR | 2 | 提案者之外的 2 位 Senior Engineer | Squash Merge |
| GTM - Release Note | 1 | Product Owner | Squash Merge |
| GTM - Battle Card | 1 | Product Marketing 或 Sales Engineering | Squash Merge |
| GTM - Feature Brief | 1 | Product Owner | Squash Merge |

### 7.3 Commit Message 规范

采用 Conventional Commits 格式，扩展文档类型 scope：

```
{type}({scope}): {description}

type:   feat | update | status | archive | fix | meta | refactor
scope:  prd | exp | ca | adr | gtm | schema | hook

示例:
  feat(ca): 新增AI代码审查SLA协议 CA-2026-004
  status(ca): CA-2026-001 状态变更 Draft -> Active
  update(prd): 更新用户中心重构需求的技术方案
  archive(exp): 归档已完成的LLM代码审查实验 EXP-2026-001
  meta(schema): 更新协作协议Schema，新增降级策略必填校验
```

---

## 8. 外部工具数据流 (Source of Truth 边界)

### 8.1 信息域权威划分

| 信息域 | Source of Truth | WisdomCore 角色 | Jira/飞书 角色 |
|-------|----------------|-----------------|---------------|
| 协作协议（SLA/接口约定） | **WisdomCore** | 权威存储、版本管理 | 不存储 |
| AI实验报告 | **WisdomCore** | 权威存储、知识积累 | 任务卡片引用链接 |
| 架构决策记录 (ADR) | **WisdomCore** | 权威存储、决策追溯 | 不存储 |
| GTM资产 | **WisdomCore** | 权威存储、资产沉淀 | 分发渠道 |
| SaaS需求 (PRD) | **双主** | 结构化Frontmatter+正文 | Sprint Board + 任务拆解 |
| 每日沟通/站会 | 飞书 | 不存储 | **权威渠道** |
| Sprint 看板 | Jira | 不存储 | **权威存储** |
| Bug Tracking | Jira | Bad Case 引用 | **权威存储** |
| 代码评审 | GitHub/GitLab | ADR 引用 | 不存储 |

### 8.2 集成原则

1. **WisdomCore 不替代任何工具**，它补充现有工具缺失的"跨职能协议层"
2. **单向引用优先**：WisdomCore 文档可引用 Jira Issue ID，但 Jira 只需放一个链接指回 WisdomCore
3. **PRD 双维护策略**：需求正文在 WisdomCore，任务拆解在 Jira；通过 `jira_epic_key` 字段关联
4. **GTM 分发机制**：GTM 资产在 WisdomCore 编写和版本管理，发布时通过脚本推送至飞书/Slack 等渠道
5. **异步最终一致**：不追求实时双向同步，接受最终一致性

---

## 9. 多团队预留设计

### 9.1 团队与产品线字段

所有文档 Frontmatter 均包含 `team` 和 `product_line` 字段。单团队阶段使用默认值 `default`，扩展时无需修改 Schema。

**config.yaml 团队配置示例：**

```yaml
# .meta/config.yaml
wisdomcore_version: "1.0.0"

teams:
  default:        { display_name: "默认团队" }
  user-platform:  { display_name: "用户平台团队", tech_lead: "ZhangSan", product_owner: "LiSi" }
  ai-platform:    { display_name: "AI平台团队", tech_lead: "WangWu", product_owner: "ZhaoLiu" }
  growth:         { display_name: "增长团队", tech_lead: "SunQi", product_owner: "ZhouBa" }

product_lines:
  default:        { display_name: "默认产品线" }
  main-app:       { display_name: "主应用" }
  ai-copilot:     { display_name: "AI Copilot" }
  data-platform:  { display_name: "数据平台" }

defaults:
  team: "default"
  product_line: "default"

naming:
  team_code_pattern: "^[a-z][a-z0-9-]{1,30}$"
  product_line_code_pattern: "^[a-z][a-z0-9-]{1,30}$"
```

### 9.2 多团队 SQL 查询示例

```sql
-- 按团队查看活跃协议
SELECT id, title, status, risk_level, next_review_date
FROM documents
WHERE type = 'CollaborationAgreement'
  AND team = 'user-platform'
  AND status IN ('Active', 'InViolation');

-- 按产品线统计各类文档数量
SELECT product_line, type, COUNT(*) AS doc_count,
       SUM(CASE WHEN risk_level = 'High' THEN 1 ELSE 0 END) AS high_risk
FROM documents
WHERE product_line = 'main-app'
GROUP BY product_line, type;

-- 跨团队协议健康度总览
SELECT team, COUNT(*) AS total_agreements,
       SUM(CASE WHEN status = 'Active' THEN 1 ELSE 0 END) AS active,
       SUM(CASE WHEN status = 'InViolation' THEN 1 ELSE 0 END) AS violated,
       ROUND(SUM(CASE WHEN status = 'InViolation' THEN 1.0 ELSE 0 END)
           / COUNT(*) * 100, 1) AS violation_rate_pct
FROM documents
WHERE type = 'CollaborationAgreement'
  AND status NOT IN ('Archived', 'Superseded')
GROUP BY team
ORDER BY violation_rate_pct DESC;
```

### 9.3 未来扩展路径

- **Phase 1（当前）**：单团队，`team` / `product_line` = `default`
- **Phase 2（2-3个团队）**：在 config.yaml 注册团队代码，文档标注所属团队
- **Phase 3（跨BU）**：引入团队级 Dashboard 视图，按 `team` 过滤展示
- **Phase 4（企业级）**：支持 RBAC 权限、团队级 Schema 扩展字段

---

## 10. Datasette 自助仪表盘配置

### 10.1 启动命令

```bash
# 基础启动（只读模式）
datasette .wisdomcore.db --metadata metadata.json -p 8001

# 带 SQL 查询界面
datasette .wisdomcore.db --metadata metadata.json -p 8001 --setting sql_time_limit_ms 5000
```

### 10.2 metadata.json 配置

```json
{
  "title": "WisdomCore Dashboard",
  "description": "混合研发管理知识库 — 文档索引、依赖关系、健康度指标",
  "databases": {
    "wisdomcore": {
      "tables": {
        "documents": { "label_column": "title", "sort_desc": "updated_at" },
        "dependencies": { "label_column": "relation_type" },
        "metrics_snapshots": { "sort_desc": "collected_at" }
      },
      "queries": {
        "active-agreements": {
          "title": "活跃协作协议",
          "sql": "SELECT id, title, status, risk_level, saas_owner, ai_owner, next_review_date FROM documents WHERE type='CollaborationAgreement' AND status IN ('Active','InViolation') ORDER BY risk_level DESC"
        },
        "health-metrics-latest": {
          "title": "最新健康度指标",
          "sql": "SELECT metric_name, metric_value, collected_at FROM metrics_snapshots WHERE collected_at = (SELECT MAX(collected_at) FROM metrics_snapshots) ORDER BY metric_name"
        },
        "zombie-documents": {
          "title": "僵尸文档（超过 90 天未更新）",
          "sql": "SELECT id, title, type, status, owner, updated_at, CAST(julianday('now') - julianday(updated_at) AS INTEGER) AS days_stale FROM documents WHERE status NOT IN ('Archived','Superseded') AND julianday('now') - julianday(updated_at) > 90 ORDER BY days_stale DESC"
        },
        "dependency-graph": {
          "title": "依赖关系全景",
          "sql": "SELECT s.id AS source, s.title AS source_title, d.relation_type, t.id AS target, t.title AS target_title FROM dependencies d JOIN documents s ON d.source_id=s.id JOIN documents t ON d.target_id=t.id ORDER BY d.relation_type, s.id"
        }
      }
    }
  }
}
```

### 10.3 Mermaid 依赖图 SVG 导出

```bash
# 从 SQLite 生成 Mermaid 文件，再导出 SVG
python3 -c "
import sqlite_utils
db = sqlite_utils.Database('.wisdomcore.db')
print('graph LR')
for row in db.execute('''
    SELECT s.id, d.relation_type, t.id
    FROM dependencies d
    JOIN documents s ON d.source_id = s.id
    JOIN documents t ON d.target_id = t.id
''').fetchall():
    print(f'    {row[0]} -->|{row[1]}| {row[2]}')
" > 05_Reports/dependency-graph.mmd

mmdc -i 05_Reports/dependency-graph.mmd -o 05_Reports/dependency-graph.svg -t dark
```
