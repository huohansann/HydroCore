# project-baseline-audit Specification

## Purpose
TBD - created by archiving change audit-project-baseline. Update Purpose after archive.
## Requirements
### Requirement: Repository baseline audit coverage
HydroCore SHALL 提供仓库基线审计，覆盖后端、前端、文档、OpenSpec/Comet 产物、数据库脚本、构建配置、生成物和本地协作配置。

#### Scenario: Audit covers primary repository entry points
- **WHEN** 执行项目基线审计
- **THEN** 审计 MUST 检查根目录文档与仓库策略、`hydrocore-be`、`hydrocore-fe`、`docs`、`openspec/specs`、active OpenSpec changes，以及相关 archived change 上下文

#### Scenario: Audit distinguishes source from generated artifacts
- **WHEN** 审计遇到 `target`、`dist`、`node_modules`、构建缓存或生成类型文件等目录
- **THEN** 审计 MUST 将其与源文件分开分类，并且 MUST NOT 要求在生成物内部手工做源码式编辑

### Requirement: Optimization and deletion classification
审计 SHALL 将每个发现分类为必须修复、建议优化、可删除候选、保留例外或后续 change 候选。

#### Scenario: Deletion candidate includes evidence
- **WHEN** 审计将文件、配置、依赖、路由、脚本或文档段落标记为可删除候选
- **THEN** 该发现 MUST 包含证据，例如无引用、属于生成物、违反策略、用途重复或存在误导性基线语义

#### Scenario: Retained exception includes rationale
- **WHEN** 审计保留看似过时或与 legacy 相关的内容
- **THEN** 该发现 MUST 记录保留原因，例如迁移支持、外部 API 兼容、生命周期历史，或缺少安全删除证据

#### Scenario: Follow-up change boundary is explicit
- **WHEN** 某个发现需要新业务能力、public API 变更、数据库 schema 变更或架构重设计
- **THEN** 审计 MUST 将其标记为后续 OpenSpec/Comet change，而不是在本基线审计 change 内实现

### Requirement: Baseline audit report
HydroCore SHALL 产出基线审计报告，作为未来开发判断优化、删除、保留或延后处理事项的决策记录。

#### Scenario: Report is actionable
- **WHEN** 基线审计报告完成
- **THEN** 报告 MUST 为每个非平凡发现记录分类、影响路径、证据、建议动作、风险等级和验证方法

#### Scenario: Report protects future extension work
- **WHEN** 后续开发者阅读基线审计报告
- **THEN** 他们 MUST 能识别当前基线边界、已知清理风险、保留例外，以及应拆分为未来 OpenSpec/Comet workflow 的变更

### Requirement: Frontend and backend source remediation
基线审计 SHALL 覆盖前后端源码层面的优化和整改，但 MUST 限制为低风险、可验证、不会改变既有 public API、数据库 schema 或业务能力边界的质量改进。

#### Scenario: Backend remediation remains within baseline scope
- **WHEN** 审计发现后端源码中的重复实现、无引用代码、误导性命名、过时占位逻辑或未说明的兼容代码
- **THEN** 实施 MAY 在有证据支撑时进行低风险整改，并且 MUST 保持现有 API 契约、数据库 schema 和运行时验证通过

#### Scenario: Frontend remediation remains within baseline scope
- **WHEN** 审计发现前端源码中的无引用页面或组件入口、误导性占位实现、重复工具代码、过时环境入口或影响扩展的轻量结构问题
- **THEN** 实施 MAY 在有证据支撑时进行低风险整改，并且 MUST 保持现有路由契约、构建配置和生产构建通过

#### Scenario: High-risk source changes are deferred
- **WHEN** 前后端源码问题需要重构公共模块边界、调整 API、改变业务数据流、迁移框架或引入新能力
- **THEN** 审计 MUST 将其记录为后续 OpenSpec/Comet change，而不是在本 change 内直接实现

### Requirement: Baseline audit verification
审计实施 SHALL 验证：任何已批准的清理或文档更新完成后，仓库质量检查仍然通过。

#### Scenario: Backend verification remains green
- **WHEN** 审计清理或文档更新完成
- **THEN** 后端验证 MUST 包含 `mvn.cmd -q test` 或已记录的等价命令；任何失败 MUST 阻塞完成，直到修复或明确记录为已接受阻塞项

#### Scenario: Frontend verification remains green
- **WHEN** 审计清理或文档更新完成
- **THEN** 前端验证 MUST 包含 `pnpm.cmd run build` 或已记录的等价命令；任何失败 MUST 阻塞完成，直到修复或明确记录为已接受阻塞项

#### Scenario: OpenSpec artifacts remain complete
- **WHEN** open 阶段准备退出
- **THEN** `openspec status --change audit-project-baseline --json` MUST 显示所有必需 apply artifacts 已完成，并且 `comet guard audit-project-baseline open --apply` MUST 在 design 开始前通过

