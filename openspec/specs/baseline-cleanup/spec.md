# baseline-cleanup Specification

## Purpose
定义 HydroCore 清理后的基线边界，确保默认源码、初始化数据、文档和协作配置不再暴露旧项目业务语义，并为后续水处理二开提供可验证、可扩展的通用底座。

## Requirements
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
