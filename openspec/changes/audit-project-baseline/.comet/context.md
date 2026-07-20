# Comet Design Handoff

- Change: audit-project-baseline
- Phase: design
- Mode: compact
- Context hash: 50a8de31775e8135ce55a4eeb8ca189bcc4fbbb286c15f41aad448e09cdcc618

Generated-by: comet-handoff.sh

OpenSpec remains the canonical capability spec. This handoff is a deterministic, source-traceable context pack, not an agent-authored summary.

## openspec/changes/audit-project-baseline/proposal.md

- Source: openspec/changes/audit-project-baseline/proposal.md
- Lines: 1-27
- SHA256: 5c141a5b0d2e7520e33a78553aebe1ffb9de791fcd6ab998598dfd90e1232aac

```md
## Why

HydroCore 已完成一轮基础清理，但未来几个月的开发都会基于当前仓库继续扩展；如果此时仍残留结构债、无用资产、重复配置或验证缺口，后续功能会不断放大这些问题。

本变更建立一次可复核的项目基线审计，明确哪些内容可以优化、哪些内容需要删除、哪些内容必须暂时保留，并把结论落实为后续开发可依赖的清单和验证标准。

## What Changes

- 新增覆盖后端、前端、数据库脚本、文档、OpenSpec/Comet 产物、构建配置和本地协作配置的基线审计流程。
- 输出按风险分类的审计结论：必须修复、建议优化、可删除、需保留、需要后续独立 change 处理。
- 对明显无用、重复、误导未来开发或违反既有基线策略的文件/配置提出删除或调整任务。
- 明确不在本 change 中直接引入新的业务能力、公共 API、数据库业务 schema 或大规模架构重写。
- 将验证标准固定为后端测试、前端构建、静态扫描和审计报告一致性检查。

## Capabilities

### New Capabilities
- `project-baseline-audit`: 覆盖当前仓库基础质量审计、优化/删除候选识别、保留例外记录和未来扩展风险控制。

### Modified Capabilities
- 无。`runtime-contracts` 和 `baseline-cleanup` 只作为审计输入与验证基线，本 change 不修改其已有需求语义。

## Impact

- 影响范围包括 `hydrocore-be/`、`hydrocore-fe/`、`docs/`、`openspec/`、根目录 README、构建脚本、数据库初始化/迁移脚本、环境模板和本地协作配置。
- 可能产生删除无用文件、整理文档、补充验证脚本或记录后续独立变更的实现任务。
- 不改变现有前后端运行时 API 契约，不新增生产业务表，不引入新的前端页面或后端业务模块。

```

## openspec/changes/audit-project-baseline/design.md

- Source: openspec/changes/audit-project-baseline/design.md
- Lines: 1-68
- SHA256: 6416f2f47c973997146d17d93093cbae0192311f24b43735041aac575ddf481f

```md
## Context

HydroCore 当前是未来水处理系统二次开发的基础仓库，包含 Spring Boot 后端、Vue 3 前端、仓库级文档、OpenSpec/Comet 生命周期产物和本地协作配置策略。

近期已完成运行时契约统一和一次基础清理，但仓库仍需要一次面向未来扩展的系统化审计：不是只扫旧业务关键字，而是判断哪些内容会影响后续几个月持续开发，包括无用生成物、本地 IDE/agent 配置、私有依赖、文档编码可读性、验证入口、前后端模块边界和可删除候选。

## Goals / Non-Goals

**Goals:**

- 建立覆盖后端、前端、文档、OpenSpec/Comet、数据库脚本和构建配置的审计方法。
- 将发现结果分成必须处理、建议优化、可删除、需保留例外、后续独立 change 五类。
- 优先识别会阻碍未来开发的基础问题：误导性文件、重复配置、生成物污染、不可复现依赖、未说明的历史兼容代码、验证缺口。
- 给出每项删除或优化的判断依据和验证方式，避免为了“看起来干净”误删仍被运行时依赖的内容。

**Non-Goals:**

- 不新增真实水处理业务模块、业务表、页面、控制算法或预测算法。
- 不在 open/design 阶段直接改应用代码。
- 不把大型架构重写纳入本 change；如果审计发现需要架构调整，应拆为后续独立 change。
- 不手工编辑 `target/`、`dist/`、`node_modules/` 等生成物；只记录其版本控制/忽略策略。

## Decisions

### Decision 1: 采用“审计报告优先，代码修改其次”的流程

先形成可复核审计清单，再决定 build 阶段执行哪些删除或优化。这样可以把“可删”与“需保留例外”分开，降低误删风险。

替代方案是直接边扫边删。该方案速度更快，但容易把历史迁移脚本、外部兼容方法或当前没有测试覆盖的配置误判为无用。

### Decision 2: 审计范围按仓库入口分层

审计按以下入口分层执行：

- 根目录：README、`.gitignore`、OpenSpec/Comet、本地协作配置策略。
- 后端：`pom.xml`、`src/`、`resources/`、数据库初始化/迁移脚本、后端文档和测试。
- 前端：`package.json`、环境模板、`src/`、`public/`、构建脚本、前端文档和生成物策略。
- 历史产物：`docs/superpowers/` 和 `openspec/changes/archive/` 只作为历史上下文，除非其内容破坏当前流程或误导新开发。

替代方案是只按文件类型扫描，但它无法准确区分源入口、生成物和历史文档。

### Decision 3: 删除候选必须有引用证据

每个删除候选至少满足一种证据：

- 无运行时引用，且不是 OpenSpec/Comet 生命周期必需文件。
- 与当前基线策略冲突，例如无说明的本地 `.claude` / `.cursor` / IDE 配置副本。
- 属于构建产物或依赖目录，且应由构建工具生成。
- 文档或配置内容会误导未来开发者认为旧业务能力仍是当前基线。

保留例外也必须记录原因，例如历史迁移脚本、外部兼容 API、构建工具缓存或当前验证必需文件。

### Decision 4: 验证以“可继续开发”为标准

完成后必须证明仓库仍可作为开发基线：

- 后端测试通过。
- 前端生产构建通过。
- OpenSpec change artifacts 完整。
- 审计报告中的每个结论都有分类、依据和后续动作。
- 生成物、本地配置和历史文档不混入不相关提交。

## Risks / Trade-offs

- 风险：误删当前没有显式引用但部署或外部集成仍依赖的文件。缓解：删除前做引用扫描，并把外部依赖不明确项归入“后续独立 change”或“需保留例外”。
- 风险：把历史 Comet/OpenSpec 文档中的旧业务描述当作当前问题。缓解：历史目录只在会误导当前入口或破坏流程时处理，其余记录为历史例外。
- 风险：审计范围过大导致一次 change 无法收敛。缓解：本 change 只处理基础基线质量问题，发现架构调整、新业务建模或公共 API 变化时拆出后续 change。
- 风险：文档编码异常影响可读性但修复可能触碰大量历史文件。缓解：优先修复当前入口和新增报告，历史产物只记录风险或后续任务。

```

## openspec/changes/audit-project-baseline/tasks.md

- Source: openspec/changes/audit-project-baseline/tasks.md
- Lines: 1-32
- SHA256: 305a0bda37262ba38dc1a7997428d73ade05e1ea599ea27ded3e102f6f1879ce

```md
## 1. 审计盘点

- [ ] 1.1 记录仓库入口和当前 Git 状态，覆盖根目录文档、`hydrocore-be`、`hydrocore-fe`、`docs`、`openspec`、生成物目录和本地配置目录。
- [ ] 1.2 盘点后端依赖、构建插件、资源模板、数据库脚本、测试和后端文档，识别优化或删除候选项。
- [ ] 1.3 盘点前端依赖、构建脚本、环境模板、生成文件、路由、复用组件、公共资产和前端文档。
- [ ] 1.4 盘点 OpenSpec/Comet 产物、归档 change、根目录协作策略和本地 IDE/agent 配置目录。

## 2. 发现分类

- [ ] 2.1 将审计发现分类为必须修复、建议优化、可删除候选、保留例外或后续 change 候选。
- [ ] 2.2 为每个可删除候选记录引用扫描证据、生成物属性、策略冲突、用途重复或误导性基线语义。
- [ ] 2.3 为每个保留例外记录保留原因，以及未来开发者需要知道的风险。
- [ ] 2.4 将涉及 public API、数据库 schema、新业务能力或架构级调整的发现拆分为后续 change 候选，不在本 change 内实现。

## 3. 基线清理

- [ ] 3.1 只执行有审计证据和当前 specs 支撑的低风险清理。
- [ ] 3.2 当当前入口文档存在误导、不可读或缺少必要基线边界时，更新对应文档。
- [ ] 3.3 避免手工编辑 `target`、`dist`、`node_modules`、缓存和 lockfile 等生成物，除非包管理器或构建工具要求。

## 4. 审计报告

- [ ] 4.1 创建基线审计报告，记录影响路径、分类、证据、建议动作、风险等级和验证方法。
- [ ] 4.2 报告必须包含优化候选、删除候选、保留例外和后续 OpenSpec/Comet change 的独立章节。
- [ ] 4.3 将报告与 `runtime-contracts`、`baseline-cleanup` 和本 change 的 `project-baseline-audit` 需求交叉核对。

## 5. 验证

- [ ] 5.1 在 `hydrocore-be` 下运行后端验证：`mvn.cmd -q test`。
- [ ] 5.2 在 `hydrocore-fe` 下运行前端验证：`pnpm.cmd run build`。
- [ ] 5.3 对 `audit-project-baseline` 运行 OpenSpec status/validation，并在验证记录中保存结果。
- [ ] 5.4 确认最终 Git diff 只包含当前 change 的审计产物和已批准的清理/报告更新。

```

## openspec/changes/audit-project-baseline/specs/project-baseline-audit/spec.md

- Source: openspec/changes/audit-project-baseline/specs/project-baseline-audit/spec.md
- Lines: 1-53
- SHA256: e1477f7ba97c9ea9dfe26311435947b76aac6bfa2b3a0fafaea525b791449cd3

```md
## ADDED Requirements

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

```
