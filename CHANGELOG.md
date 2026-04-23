# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [1.1.1] - 2026-04-23

### Changed

- **SKILL.md `name` 字段改为全小写**（`sloth-hybriddev-eido`），对齐 QoderWork 官方 Skill 规范（仅限小写字母/数字/连字符）
- **SKILL.md `description` 补充触发条件**（增加 "Use when..." 句式），对齐官方规范要求的 WHAT + WHEN 双要素
- **SKILL.md 正文精简 39%**（371行→225行，23KB→16KB），将与 references/ 重复的大段内容（架构图、状态机转换表、度量指标表、仪式日历等）替换为一句话摘要 + 链接，符合官方 "Concise is Key" 和 Progressive Disclosure 原则

## [1.1.0] - 2026-04-23

### Added

- 推荐工具链体系（architecture.md 新增第 2 章）：
  - **sqlite-utils** (P0) — SQLite 读写 CLI + 库，替代自研同步脚本核心逻辑
  - **remark-lint-frontmatter-schema** (P0) — Frontmatter JSON Schema 校验，行号级错误定位
  - **Datasette** (P1) — SQLite Web UI 自助仪表盘，Skill-H 双通道输出
  - **phodal/adr** (P1) — ADR CLI 工具（中文友好），支持 `adr new/list/export`
  - **git-cliff** (P1) — 自动 CHANGELOG 生成，降低 R3 风险
  - **Mermaid CLI** (P2) — 依赖图 SVG/PNG 导出至 05_Reports/
  - **markdownlint-cli** (P2) — Markdown 风格一致性检查
- Skill-H 新增 Datasette 自助服务模式（双通道输出：AI 对话 + Web UI）
- Skill-A 新增 Agent 交互记忆层（memweave 架构启发，`.meta/memory/`）
- ADR 框架新增 phodal/adr CLI 集成章节
- 度量仪式新增 git-cliff 自动 CHANGELOG 配置（`.meta/cliff.toml`）
- 工具链集成架构图（Pre-commit Pipeline + Post-commit Hook + Datasette Serve）
- 文档治理新增 Datasette 实时监控和 JSON API 消费方式
- v2.0 候选工具观望清单：Trackio、DVC、memweave/sqlite-memory

## [1.0.0] - 2026-04-23

### Added

- 完整"中枢神经系统"架构：父Skill（知识本体库）+ 4个子Skill（S/A/H/G）
- SKILL.md 意图路由器，覆盖全流水线与元操作
- 11 个 references/ 深度操作规格文档：
  - `architecture.md` — Markdown + SQLite 双写架构、目录结构、Schema 设计
  - `parent-skill.md` — 五类文档 Frontmatter 完整规范与正文模板
  - `agreement-state-machine.md` — 协作协议 7 状态 15 转换完整状态机
  - `document-governance.md` — 文档所有权、鲜度机制、腐烂检测
  - `change-management.md` — 变更分类、依赖链追踪、影响分析报告
  - `adr-framework.md` — 决策记录模板与治理流程
  - `skill-s-agreement-forge.md` — 协作协议生成器完整功能规格
  - `skill-a-experiment-log.md` — AI 实验记录员完整功能规格
  - `skill-h-hybrid-view.md` — 混合视图生成器仪表盘与健康度模型
  - `skill-g-gtm-architect.md` — GTM 知识架构师翻译规则与模板
  - `metrics-ceremonies-risk-impl.md` — 度量体系、仪式日历、风险矩阵与实施路径
- 度量体系：6 大核心指标（协议覆盖率、合规率、文档鲜度、关联率、GTM周期、僵尸数）
- 4 级协作仪式日历（周/双周/月/季度）
- 安全与权限分层（6 角色 × 5 目录权限矩阵）
- 风险矩阵（R1-R12，含缓解策略与监控指标）
- 8 周 4 阶段实施路径（同步开发、分批上线）
