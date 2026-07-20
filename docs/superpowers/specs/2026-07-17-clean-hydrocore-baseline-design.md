---
comet_change: clean-hydrocore-baseline
role: technical-design
canonical_spec: openspec
archived-with: 2026-07-20-clean-hydrocore-baseline
status: final
---

# Clean HydroCore Baseline 技术设计

## 背景

HydroCore 已从旧项目中抽离出前后端基础壳，但仍有旧窑炉、液位、压力、预测、控制等语义残留在源码、初始化 SQL、Nacos/env 模板、前端主题变量、系统配置枚举和文档中。当前目标不是开发水处理业务，而是把仓库整理成可二开的干净基线：后续团队可以在此基础上重新设计水处理工艺、设备、监测、算法和报表，而不会被旧业务模型误导。

本设计遵循 OpenSpec change `clean-hydrocore-baseline`，新能力规格为 `baseline-cleanup`。实现阶段必须保持通用系统底座可运行，包括认证、权限、菜单、组织、用户、系统配置、通用数据查询和前端基础构建。

## 采用方案

采用“源入口优先清理”方案。

只人工清理可维护源文件和交付入口，包括：

- 后端 `hydrocore-be/src/main/resources/db`、`nacos`、`bootstrap.yml`、系统配置枚举/常量、相关服务注释和后端文档。
- 前端 `hydrocore-fe/src`、环境模板、主题样式、系统配置模型、通用图表工具和前端文档。
- 根目录 README、`docs/`、OpenSpec/Comet artifact 和本地 agent/IDE 配置策略。

不人工编辑 `target/`、`dist/`、锁文件和第三方生成内容。它们应通过重新构建自然刷新或作为例外记录。

## 实施边界

### 后端

后端清理聚焦四类入口。

1. 数据库初始化

`hydrocore_schema.sql` 应只保留通用底座运行所需的表和种子数据。旧窑炉表、液位控制表、预测结果表、压力/温度预测相关默认数据应删除。若某些迁移脚本用于从旧安装清理菜单或角色，可保留在 `db/migration`，但注释必须明确它是历史迁移脚本，不是当前基线能力。

2. 系统配置

`SysConfigModuleEnum`、`SysConfigCodeConstants` 和后端系统配置 API 文档需要同步中性化。默认模块不应再暴露 `FORECAST`、旧 `CONTROL` 或旧预测点位菜单。若仍需要系统配置示例，应使用通用配置或水处理中性的占位名称，不提前建模真实工艺配置。

3. Nacos 与启动配置

`bootstrap.yml` 和 `src/main/resources/nacos` 中可以保留本地开发便利默认值，但生产敏感值必须使用占位或文档说明。旧 forecast/control 测试开关不应作为默认基线能力启用。历史迁移说明可以存在，但必须与当前配置模板分开表达。

4. 通用数据 API

若旧 `forecast` 命名只存在于仍被通用图表数据接口调用的兼容方法中，实施时先做引用确认。能改成中性命名则改；若改名会扩大 API 契约，应保留行为并在验证例外中说明，避免为了字符串清理破坏通用数据查询。

### 初始化账号策略

用户已确认采用保留最小管理员/本地演示 seed 账号的策略。

要求：

- 只保留通用系统启动所需的最小账号、角色和权限。
- 替换弱示例密码；如密码以 hash 形式存储，应确认 hash 对应的明文仅用于本地开发，并在文档中要求首次修改。
- 文档必须明确 seed 账号不得用于生产，生产部署应改密或重新初始化账号。
- 不保留旧业务角色、旧菜单绑定或旧项目演示账号。

### 前端

前端清理聚焦默认用户可见面和可复用组件。

1. 主题变量

`--kiln-*` 应迁移为 `--app-*` 或更具体的组件语义变量，`tailwind.css` 同步引用新变量名。颜色值本身不需要大改，除非构建或样式引用要求。

2. 系统配置页面

前端 `SysConfigModuleEnum`、配置页 tabs、表单 options 和常量应与后端模块保持一致。旧预测配置、压力配置等不应出现在默认配置界面。

3. 通用图表组件

保留可复用图表能力，但将 `singleForecast`、`multiForecast`、`GAS` 等旧业务命名迁移到中性 series/data 命名。组件行为不做重构，避免把基线清理扩大为可视化系统重写。

4. 环境模板

前端环境配置不应只指向团队内网地址。建议提供 `.env.example` 或把现有 `.env.development` 调整为 localhost/占位说明，并在 README 中说明本地代理与生产地址配置。

5. 图标和组件残留

`iconfont` 元数据可能包含大量第三方或历史图标名称。实施时先确认是否被引用；未使用旧预测/控制图标可删除，保留的通用图标元数据作为关键词例外记录。

### 文档与仓库策略

根目录、前端、后端文档需要共同回答四个问题：

- 如何启动后端、前端和依赖服务。
- 如何初始化数据库、配置 Nacos/env。
- 如何运行验证命令。
- 后续水处理业务应通过哪些扩展入口进入，哪些能力本次基线明确不包含。

本仓库默认使用全局 Comet/OpenSpec/skills。无用途或重复的 `.claude`、`.cursor`、`.agents`、`.codex`、skills 配置应删除或忽略；若保留某个 IDE 配置，必须在文档中说明用途。

## 依赖与数据流

清理顺序应保持从风险低到风险高：

1. 关键词盘点，建立命中分类和例外清单。
2. 后端 SQL 与配置清理，保证种子数据和配置模块不再发布旧业务默认项。
3. 后端编译和测试，捕获被旧配置间接依赖的代码。
4. 前端枚举、主题、图表、环境模板清理，保证 UI 与后端配置模块一致。
5. 前端构建验证。
6. 文档更新和最终关键词复查。

配置数据流应保持简单：后端系统配置模块定义为事实来源，前端配置枚举只镜像可展示模块。若后端删除某模块，前端必须同步删除对应 tab 和表单 option。

## 风险与缓解

- 风险：删除旧 seed 后无法登录。缓解：保留最小管理员/本地演示账号，替换弱密码并文档化首次改密。
- 风险：系统配置枚举前后端不一致。缓解：同一任务内修改后端 enum/constants、前端 enum/tabs/options 和 API 文档。
- 风险：旧 forecast 命名被通用数据 API 兼容路径依赖。缓解：先引用搜索，再决定中性化或作为兼容例外记录。
- 风险：关键词强清理误改 `target/`、`dist/`、iconfont 或历史设计文档。缓解：只编辑源入口，生成物不手工处理，历史文档例外必须标注。
- 风险：清理工作滑向真实水处理业务建模。缓解：禁止新增业务 schema、业务页面、控制算法和领域 API；这些通过后续 Comet change 设计。

## 验证策略

实施完成后至少运行：

```powershell
cd hydrocore-be
mvn -q -DskipTests compile
mvn -q test
```

```powershell
cd hydrocore-fe
pnpm run build
```

同时执行关键词复查：

```powershell
rg -n -i "kiln|forecast|level|pressure|GAS|temperament|predict|窑炉|预测|液位|压力" hydrocore-be\src hydrocore-fe\src hydrocore-be\docs hydrocore-fe\docs README.md docs -u
```

允许例外只能是：

- 明确标注为历史迁移或历史设计说明的文档。
- 第三方或 iconfont 元数据中未被默认界面引用的名称。
- 因公共 API 兼容必须保留且已在验证记录中说明的中性包装或兼容方法。

## 回滚策略

若验证失败，按模块回退具体清理项：

- 后端失败优先回看 SQL、配置枚举、Nacos/bootstrap 和通用数据 API 兼容点。
- 前端失败优先回看配置枚举、主题变量引用和图表工具命名。
- 若无法快速定位，暂停 build 阶段并记录阻塞，不恢复全部旧业务模型。

## 交付结果

本 change 完成后，HydroCore 应表现为一个干净的水处理二开底座：默认配置、初始化数据、前端界面和文档不再把旧窑炉/预测/液位/压力业务作为当前能力；通用系统底座仍能编译、测试和构建；后续真实水处理模块可以通过新的 OpenSpec/Comet change 设计和实施。
