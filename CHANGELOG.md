# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

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
