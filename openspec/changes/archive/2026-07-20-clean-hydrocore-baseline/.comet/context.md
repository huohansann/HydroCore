# Comet Design Handoff

- Change: clean-hydrocore-baseline
- Phase: design
- Mode: compact
- Context hash: 27e94686cc571a644aeeb9e794119b7e628ca2d124dcba99d540897135ae6b47

Generated-by: comet-handoff.sh

OpenSpec remains the canonical capability spec. This handoff is a deterministic, source-traceable context pack, not an agent-authored summary.

## openspec/changes/clean-hydrocore-baseline/proposal.md

- Source: openspec/changes/clean-hydrocore-baseline/proposal.md
- Lines: 1-32
- SHA256: 50dc8d6fe9113645b08138a9459f835ea0e0de2058dc1567a019c2d507b72bd7

```md
## Why

HydroCore 当前已经清理成较瘦的前后端壳，但代码、数据库脚本、环境模板和文档里仍残留窑炉、液位、压力、预测等旧项目语义。若直接作为水处理系统二开基础，这些残留会误导后续建模、配置、菜单和数据初始化，并增加交接和部署风险。

本次变更用于把仓库整理成一个可维护、可验证、可继续扩展的水处理二开基线，而不是直接实现具体水处理业务模块。

## What Changes

- 清理后端数据库初始化脚本中的旧业务表、旧菜单/配置种子数据和弱示例账号风险，保留系统运行所需的通用基础结构。
- 梳理后端系统配置枚举、配置编码和资源模板，移除或改名仍指向旧窑炉、液位、压力、预测语义的默认项。
- 清理前端环境模板、主题变量、通用图表组件和系统配置枚举中的旧业务命名，使默认界面和二开入口保持中性。
- 整理根目录、前端、后端的运行与二开文档，说明如何启动、验证、配置环境变量、接入真实水处理模块。
- 移除无用的本地 Agent/IDE/技能配置残留，保留对 Comet/OpenSpec 工作流有用的全局调用方式说明。
- 不新增水处理工艺、监控大屏、控制算法、数字孪生、业务数据库模型或新的对外 API。

## Capabilities

### New Capabilities

- `baseline-cleanup`: 定义 HydroCore 作为水处理二开基线时必须满足的代码语义、配置模板、初始化数据、文档和验证要求。

### Modified Capabilities

- 无。当前 `openspec/specs/` 尚无既有能力规格，本次以新增基线能力规格承载要求。

## Impact

- 影响后端 `hydrocore-be` 的数据库 schema/seed 脚本、系统配置枚举、配置编码常量、资源模板和后端文档。
- 影响前端 `hydrocore-fe` 的环境模板、主题变量、通用图表组件、系统配置枚举、图标/组件残留和前端文档。
- 影响根目录文档、OpenSpec/Comet 变更文档和仓库级忽略/说明文件。
- 不改变现有认证、权限、通用系统管理 API 的外部契约；如必须调整示例种子数据，应在文档中说明迁移与初始化影响。
- 验证范围包括后端编译与测试、前端构建，以及关键文档和模板的人工一致性检查。

```

## openspec/changes/clean-hydrocore-baseline/design.md

- Source: openspec/changes/clean-hydrocore-baseline/design.md
- Lines: 1-77
- SHA256: 5b331770642935e42ea1f22a0ebac03d35552852d302aa2214082cbe6fe5e128

```md
## Context

HydroCore 当前包含根仓库以及 `hydrocore-be`、`hydrocore-fe` 两个前后端项目目录。前期验证显示后端 Maven 编译、后端测试和前端构建均可通过，但仓库仍保留旧项目语义：数据库初始化脚本含窑炉/液位/压力/预测表和种子数据，后端系统配置仍暴露 `CONTROL`、`FORECAST` 等旧分类，前端主题变量和通用图表组件仍带有 kiln、forecast、GAS 等命名。

本次设计目标是建立“二开基线清理”的实施边界：只清理通用基线中会误导后续水处理二开的旧语义、弱示例数据和不清晰文档，不在本阶段引入真实水处理工艺模型。

## Goals / Non-Goals

**Goals:**

- 让后端数据库初始化脚本、配置枚举、配置编码和 Nacos/env 模板不再默认暴露旧窑炉、液位、压力、预测业务语义。
- 让前端环境模板、主题变量、通用图表组件和系统配置枚举保持中性命名，可作为水处理系统二开起点。
- 保留认证、权限、菜单、组织、用户、系统配置等通用底座能力，并保持后端测试与前端构建通过。
- 用根目录、前端、后端文档说明启动、验证、环境配置、种子数据策略和后续水处理模块接入边界。
- 明确本仓库使用全局 Comet/OpenSpec/skills 的方式，避免恢复无用的仓库级 Agent/IDE 配置。

**Non-Goals:**

- 不实现水处理工艺流程、设备台账、监测大屏、控制算法、告警策略、报表模型或数字孪生。
- 不新增业务数据库 schema、外部 API、前端业务页面或复杂权限模型。
- 不对嵌套前后端项目做大规模框架升级、依赖升级或 UI 重设计。
- 不清理构建产物中的历史字符串，除非它们来自源码或模板并影响后续构建输出。

## Decisions

### 1. 以“源码与初始化入口”为清理优先级

优先修改 `src/main/resources/db`、`src/main/resources/nacos`、`bootstrap.yml`、前端 `src/`、`.env*` 模板和文档。`target/`、`dist/` 等构建产物不作为人工编辑对象，通过重新构建自然刷新。

备选方案是全仓库字符串级替换，但这会误改生成产物、锁文件和第三方输出，风险高且不利于审查。本次采用入口优先策略，便于把行为变化限制在可维护源文件内。

### 2. 旧业务 schema 采用删除或中性化，不迁移为水处理 schema

对明显属于旧项目的表、菜单、配置和测试样例，若通用底座不依赖则删除；若组件或接口仍需要通用示例，则改为中性命名和中性样例。不会把 `kiln_info` 等表直接改名为水处理表，因为这会伪造尚未设计的领域模型。

备选方案是预置一套水处理领域 schema，但当前需求是二开基础，不是业务建模。提前建模会把未知工艺假设固化到基线里。

### 3. 保持公共契约稳定

认证、权限、菜单、组织、用户、系统配置等通用 API 的路径、请求格式和核心权限语义保持稳定。必要的种子数据调整应只影响默认初始化内容，并在文档中说明。

备选方案是同时重命名 API 和模块包名，但这会扩大二开前的迁移成本，并可能破坏已通过的基础验证。

### 4. 前端命名清理以通用组件语义为准

主题变量从 `--kiln-*` 等旧业务名迁移为 `--app-*` 或组件语义名；图表组件中的 forecast/GAS 命名按实际功能改为 series/chart/data 等中性术语。组件行为保持不变，避免把清理任务变成图表重构。

备选方案是删除相关图表组件，但它们可作为通用可视化底座，直接删除会降低二开可用性。

### 5. 文档作为交付边界的一部分

根目录文档说明整体结构和 Comet/OpenSpec 使用；后端文档说明配置、数据库初始化、Nacos 发布和验证命令；前端文档说明环境变量、构建和扩展入口。文档必须同步记录哪些旧业务语义已清理、哪些真实水处理能力仍需后续 change 设计。

备选方案是只改代码不写文档，但基线项目的主要价值在于可交接和可二开，文档缺失会让清理结果难以复用。

## Risks / Trade-offs

- 旧业务残留可能被通用代码间接引用 -> 通过 `rg` 检查关键词，并运行后端测试和前端构建验证。
- 删除 seed 数据可能影响本地演示登录或菜单展示 -> 保留最小管理员/演示账号策略，替换弱密码或标注文档化初始化步骤。
- 配置枚举改名可能影响前端/后端映射 -> 同步修改两端枚举和使用点，避免只改一端。
- Nacos/bootstrap 默认值过度收紧可能降低本地启动便利性 -> 使用模板和文档区分本地默认、生产占位符和敏感凭据。
- 清理范围可能滑向真实水处理业务开发 -> tasks 中明确不新增领域表、页面和算法，把业务设计留给后续 change。

## Migration Plan

1. 先做关键词与依赖盘点，确认旧业务语义出现位置和是否被运行路径引用。
2. 清理后端数据库、配置、模板和文档，运行后端编译与测试。
3. 清理前端环境、主题、图表、枚举和文档，运行前端构建。
4. 更新根目录说明和验证记录，确认 Comet/OpenSpec artifact 与仓库状态一致。

回滚策略：若清理导致基础验证失败，优先按失败模块回退该模块的具体清理项，而不是恢复全部旧业务数据；无法快速修复时记录阻塞并停留在 build 阶段。

## Open Questions

- 默认 seed 是否保留演示账号，还是只保留初始化管理员占位流程，需要在实施时结合现有认证测试决定。
- 旧预测/控制相关代码是否全部未使用，需在 build 阶段通过引用搜索和编译结果确认后再删除。
- 是否需要新增 `.env.example` 或只调整现有 `.env.development`、`.env.production`，取决于前端当前启动脚本和团队部署习惯。

```

## openspec/changes/clean-hydrocore-baseline/tasks.md

- Source: openspec/changes/clean-hydrocore-baseline/tasks.md
- Lines: 1-39
- SHA256: 802b5414abdd38fed6472881d55a02ec394ff7cb0a553069ac707d5ee7529a97

```md
## 1. 基线盘点

- [ ] 1.1 对源码、模板、SQL 和文档执行旧业务关键词盘点，覆盖 `kiln`、`forecast`、`level`、`pressure`、`GAS`、`temperament`、`predict`、`窑炉`、`预测`、`液位`、`压力`。
- [ ] 1.2 将每个命中项分类为源码行为、初始化数据、环境模板、历史文档、生成产物、第三方图标元数据或可删除残留。
- [ ] 1.3 定义允许保留的历史文档例外，并在最终验证记录中列出所有仍有意保留的旧业务术语。

## 2. 后端清理

- [ ] 2.1 清理 `hydrocore-be/src/main/resources/db/schema/hydrocore_schema.sql`，确保默认 schema 和种子数据不再安装旧窑炉、液位、压力、预测或弱示例凭据相关内容。
- [ ] 2.2 复查 `hydrocore-be/src/main/resources/db/migration/`，仅保留用于旧安装迁移到干净基线的脚本，并把历史迁移注释标记清楚。
- [ ] 2.3 替换或移除 `SysConfigModuleEnum`、`SysConfigCodeConstants`、相关组装器和后端系统配置 API 文档中的旧系统配置模块与编码。
- [ ] 2.4 清理后端 Nacos 与 `bootstrap.yml` 模板，使默认配置名称保持 HydroCore 中性语义，且旧预测或旧控制测试数据不作为基线能力启用。
- [ ] 2.5 在不破坏保留的通用数据 API 前提下，删除或中性化后端预测/控制测试 JSON、服务注释和示例说明。
- [ ] 2.6 运行后端编译和测试，并修复后端清理造成的任何失败。

## 3. 前端清理

- [ ] 3.1 将 `--kiln-*` 主题变量及对应 Tailwind 引用替换为 `--app-*` 或组件语义名称。
- [ ] 3.2 清理前端系统配置枚举、标签页、表单、标签和常量，使默认配置界面不再暴露旧预测或压力语义。
- [ ] 3.3 对保留为通用图表工具的组件和工具函数进行中性命名，处理 `singleForecast`、`multiForecast`、`GAS` 等旧业务词。
- [ ] 3.4 复查 iconfont 和组件残留，删除未使用的旧预测/控制图标，或记录通用图标元数据被保留的原因。
- [ ] 3.5 用安全示例和模板替换前端硬编码环境地址，并补充本地代理配置说明。
- [ ] 3.6 运行前端构建，并修复前端清理造成的任何失败。

## 4. 文档与仓库策略

- [ ] 4.1 更新根目录 README 和 `docs/`，说明 HydroCore 是水处理系统二开基线，并明确真实业务模块属于非目标。
- [ ] 4.2 更新后端文档，覆盖数据库初始化、Nacos/bootstrap 配置、验证命令和种子凭据策略。
- [ ] 4.3 更新前端文档，覆盖 env 文件、代理设置、构建验证和二开扩展入口。
- [ ] 4.4 删除或忽略根目录、前端和后端中无用途的 `.claude`、`.cursor`、`.agents`、`.codex` 与重复 skills 配置；如保留，必须写明用途。
- [ ] 4.5 文档化 Comet/OpenSpec/skills 由全局安装调用，本仓库只保存 OpenSpec 产物。

## 5. 最终验证

- [ ] 5.1 重新执行旧业务关键词盘点，确认仅保留允许的历史文档或第三方图标元数据例外。
- [ ] 5.2 在 `hydrocore-be` 中运行 `mvn -q -DskipTests compile`。
- [ ] 5.3 在 `hydrocore-be` 中运行 `mvn -q test`。
- [ ] 5.4 在 `hydrocore-fe` 中运行 `pnpm run build`。
- [ ] 5.5 更新 Comet/OpenSpec 验证记录，写明命令、结果、剩余例外和真实水处理模块的后续建议。

```

## openspec/changes/clean-hydrocore-baseline/specs/baseline-cleanup/spec.md

- Source: openspec/changes/clean-hydrocore-baseline/specs/baseline-cleanup/spec.md
- Lines: 1-75
- SHA256: 2456e1bfc6bb2c3e038a5d7278f23209f73fecf60b9c25811a1e72867f9008b3

```md
## ADDED Requirements

### Requirement: Clean baseline business semantics
HydroCore 作为水处理二开基线时 SHALL 不在默认源码、初始化脚本、环境模板和用户可见文档中暴露旧窑炉、液位、压力、预测等项目语义，除非文档明确标记为历史迁移说明。

#### Scenario: Old business keywords are removed from baseline entry points
- **WHEN** 开发者检查数据库初始化脚本、Nacos/env 模板、前端默认界面配置和根/前后端文档
- **THEN** 这些基线入口不再以默认能力形式提供 kiln、level、pressure、forecast、窑炉、液位、压力、预测等旧业务语义

#### Scenario: Historical context remains isolated
- **WHEN** 文档必须解释旧项目清理背景
- **THEN** 文档 MUST 将其标记为历史说明或迁移说明，且不得把旧业务项描述为当前基线能力

### Requirement: Safe initialization data
系统初始化数据 SHALL 只包含运行通用底座所需的最小菜单、权限、用户、角色和配置数据，并 MUST 避免弱示例密码、旧业务菜单和旧业务配置作为默认种子。

#### Scenario: Database seed supports generic startup
- **WHEN** 使用仓库提供的初始化脚本准备本地开发数据库
- **THEN** 系统 MUST 保留登录、权限、菜单、组织、用户和系统配置等通用底座运行所需数据

#### Scenario: Legacy seed data is not installed
- **WHEN** 初始化脚本执行完成
- **THEN** 默认数据 MUST 不创建旧窑炉、液位控制、压力预测、温度预测或同类旧项目业务表和菜单

#### Scenario: Credentials are documented safely
- **WHEN** 初始化数据包含本地演示账号或管理员账号
- **THEN** 文档 MUST 说明用途、修改方式和安全注意事项，且默认值不得作为生产凭据使用

### Requirement: Neutral backend configuration
后端系统配置 SHALL 使用通用或水处理中性的模块、编码和模板命名，并 MUST 保持既有通用系统管理能力可编译、可测试。

#### Scenario: Backend config enums are neutral
- **WHEN** 开发者查看系统配置模块枚举和配置编码常量
- **THEN** 默认模块与编码 MUST 不再以旧预测、旧控制、旧窑炉、旧液位或旧压力语义命名

#### Scenario: Backend verification remains green
- **WHEN** 完成后端清理后运行后端编译和测试
- **THEN** Maven 编译与测试 MUST 通过，或失败原因必须被记录为需要修复的 build 阻塞

### Requirement: Neutral frontend baseline
前端基线 SHALL 使用通用主题变量、图表参数、系统配置枚举和环境模板命名，并 MUST 保持构建通过。

#### Scenario: Theme and chart names are generic
- **WHEN** 开发者查看前端主题样式、通用图表组件和公共工具函数
- **THEN** 变量、props、常量和默认文案 MUST 使用 app、chart、series、data 等中性语义，而不是 kiln、forecast、GAS 等旧业务语义

#### Scenario: Frontend environment is template-safe
- **WHEN** 开发者查看前端环境配置
- **THEN** 默认模板 MUST 避免硬编码团队内网地址作为唯一可用配置，并说明本地代理和生产地址如何设置

#### Scenario: Frontend build remains green
- **WHEN** 完成前端清理后运行前端构建
- **THEN** TypeScript 检查与 Vite 构建 MUST 通过，或失败原因必须被记录为需要修复的 build 阻塞

### Requirement: Development handoff documentation
仓库 SHALL 提供足够的根目录、后端和前端文档，使二开人员能够启动、验证、配置和扩展 HydroCore 基线。

#### Scenario: New developer can find startup path
- **WHEN** 新开发者阅读根目录 README 和前后端文档
- **THEN** 文档 MUST 指向后端启动、前端启动、数据库初始化、Nacos/env 配置和验证命令

#### Scenario: Water-treatment extension boundary is explicit
- **WHEN** 二开人员准备新增真实水处理能力
- **THEN** 文档 MUST 说明本次基线不包含工艺模型、监测页面、控制算法和业务 schema，并建议通过后续 OpenSpec/Comet change 设计这些能力

### Requirement: Repository agent configuration policy
仓库 SHALL 保持轻量的协作配置策略：默认使用全局 Comet/OpenSpec/skills 能力，仓库内不保留无明确用途的本地 `.claude`、`.cursor`、`.agents`、`.codex` 或重复 skills 配置。

#### Scenario: Local agent files are justified or absent
- **WHEN** 开发者检查根目录、前端目录和后端目录的 agent/IDE 配置
- **THEN** 无用途或重复的本地配置 MUST 被删除或忽略；保留项 MUST 在文档中说明用途

#### Scenario: Comet remains callable globally
- **WHEN** 开发者在仓库中调用 Comet workflow
- **THEN** Comet/OpenSpec MUST 通过全局安装和仓库 `openspec/` artifact 工作，而不依赖重复的仓库本地 skills 副本

```
