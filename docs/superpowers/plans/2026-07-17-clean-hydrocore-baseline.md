---
change: clean-hydrocore-baseline
design-doc: docs/superpowers/specs/2026-07-17-clean-hydrocore-baseline-design.md
base-ref: HEAD
archived-with: 2026-07-20-clean-hydrocore-baseline
---

# Clean HydroCore Baseline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** �?HydroCore 整理为可二开的干净基线，保留通用认证、权限、菜单、组织、用户、系统配置、通用数据查询和前端基础构建能力，移除默认交付入口中的旧窑炉、预测、液位、压力、控制语义�?
**Architecture:** 采用源入口优先清理：先盘点残留关键字，再按后�?SQL/配置、前端可见界�?主题、文档和验证记录收敛。只清理可维护源文件、配置模板和交付文档，不手动编辑 `target/`、`dist/`、锁文件或第三方生成内容。后端系统配置模块定义是事实来源，前端配置枚举和 tabs 只镜像后端可展示模块�?
**Tech Stack:** Spring Boot 2.6.13, Java 8, MyBatis-Plus, Nacos, TDengine, Maven, Vue 3, Vite, TypeScript 5.9, Pinia, Naive UI / Element Plus, pnpm.

**Spec:** `docs/superpowers/specs/2026-07-17-clean-hydrocore-baseline-design.md`

**Base Ref Note:** 创建本计划时执行 `git rev-parse HEAD` 返回退出码 1，仓库状态为 `No commits yet on master`；命�?stdout �?`HEAD`，因�?frontmatter 按当前可记录值写�?`base-ref: HEAD`�?
## Global Constraints

- 只创建或修改�?change 范围内的源入口、配置模板、交付文档和验证记录；不手动编辑 `target/`、`dist/`、锁文件或第三方生成内容�?- 不新增真实水处理工艺 schema、业务页面、控制算法、预测算法或领域 API；后续真实水处理模块必须通过新的 OpenSpec/Comet change 设计�?- 保留最小管理员/本地演示 seed 账号、通用角色、通用权限和系统启动所需菜单；不得保留旧业务角色、旧菜单绑定或旧项目演示账号�?- seed 账号文档必须声明不得用于生产，生产部署必须改密或重新初始化账号�?- 后端默认系统配置不得暴露 `FORECAST`、旧 `CONTROL`、旧预测点位菜单、液位控制、温�?压力预测等旧业务项�?- 前端默认配置界面不得暴露旧预测、压力、液位或控制语义�?- `--kiln-*` 前端主题变量必须迁移�?`--app-*` 或更具体组件语义变量；颜色值可保持不变�?- 旧业务关键字复查命令必须覆盖 `kiln|forecast|level|pressure|GAS|temperament|predict|窑炉|预测|液位|压力`�?- 不回退他人修改；执行任务前�?`git status --short` 识别已有改动，实施时只触碰本任务列出的文件�?
---

## File Map

| Path | Responsibility |
|------|----------------|
| `hydrocore-be/src/main/resources/db/schema/hydrocore_schema.sql` | 当前基线 schema �?seed 数据，只保留通用底座表、菜单、角色、权限、系统配�?|
| `hydrocore-be/src/main/resources/db/migration/hydrocore_menu_migrate.sql` | 历史安装迁移脚本，保留时必须标注为历史迁移而非当前基线能力 |
| `hydrocore-be/src/main/resources/db/migration/iot_collection_tables_alter_rtu.sql` | 历史/通用采集迁移脚本，复查是否含旧业务命�?|
| `hydrocore-be/src/main/java/com/siact/hydrocore/module/system/enums/SysConfigModuleEnum.java` | 后端系统配置模块枚举，作为前后端模块一致性的事实来源 |
| `hydrocore-be/src/main/java/com/siact/hydrocore/module/system/constants/SysConfigCodeConstants.java` | 后端系统配置编码常量，移除旧预测/控制/液位常量 |
| `hydrocore-be/src/main/java/com/siact/hydrocore/module/system/processor/ConfigAssembler.java` | 系统配置组装逻辑，确认无旧模块特殊分�?|
| `hydrocore-be/src/main/java/com/siact/hydrocore/module/system/service/impl/SysConfigServiceImpl.java` | 系统配置服务，确认模块过滤与新枚举一�?|
| `hydrocore-be/docs/api/sys-config-api.md` | 后端系统配置 API 文档，替换旧模块示例 |
| `hydrocore-be/src/main/resources/bootstrap.yml` | Nacos 引导配置，保�?HydroCore data-id 和安全占�?|
| `hydrocore-be/src/main/resources/nacos/hydrocore.yml` | 后端主配置模板，删除�?forecast/control 默认开关和测试 fixtures |
| `hydrocore-be/src/main/resources/nacos/hydrocore-constant.yml` | 后端常量配置模板，清理旧业务 key |
| `hydrocore-be/src/main/resources/nacos/hydrocore-config.properties` | 后端属性模板，清理旧业�?key |
| `hydrocore-be/src/main/resources/nacos/README.md` | Nacos 配置说明，区分历史迁移说明与当前模板 |
| `hydrocore-be/src/main/resources/testJson/forecast/*.json` | 旧预测测�?JSON，删除或迁移为通用图表 fixtures |
| `hydrocore-be/src/main/resources/testJson/monitoring/getModelData.json` | 旧监控测�?JSON，删除或改为中�?fixture |
| `hydrocore-be/src/main/java/com/siact/hydrocore/sec/sevice/DataService.java` | 通用数据 API 兼容接口，识�?forecast 命名是否只能作为外部 API 兼容例外 |
| `hydrocore-be/src/main/java/com/siact/hydrocore/sec/sevice/impl/DataServiceImpl.java` | 通用数据 API 实现，移除旧测试模式默认依赖或记录兼容例�?|
| `hydrocore-be/src/main/java/com/siact/hydrocore/tdengine/service/TaosDataService.java` | TDengine 查询接口，识别预测方法是否为外部兼容例外 |
| `hydrocore-be/src/main/java/com/siact/hydrocore/tdengine/service/TaosDataServiceImpl.java` | TDengine 查询实现，保留兼容行为时必须改中性注�?|
| `hydrocore-be/src/main/java/com/siact/hydrocore/core/common/config/ThreadPoolConfig.java` | 线程池配置，删除或中性化 algorithm prediction 语义 |
| `hydrocore-be/src/main/java/com/siact/hydrocore/common/constant/ConstantBase.java` | 通用常量，清理旧预测/气体/负荷告警语义 |
| `hydrocore-be/src/main/java/com/siact/hydrocore/common/constant/ConstantField.java` | 通用字段常量，清�?gas 专用字段或记录外部兼容例�?|
| `hydrocore-fe/src/models/system/config/config.enums.ts` | 前端系统配置模块枚举，镜像后�?`SysConfigModuleEnum` |
| `hydrocore-fe/src/views/system/config/index.tsx` | 前端系统配置 tabs，不再展示旧控制/预测模块 |
| `hydrocore-fe/src/views/system/config/components/config-form.tsx` | 前端系统配置表单 options，不再可选旧控制/预测模块 |
| `hydrocore-fe/src/styles/themes.css` | 主题 CSS 变量定义，`--kiln-*` 改为 `--app-*` |
| `hydrocore-fe/src/styles/tailwind.css` | Tailwind 主题引用，改�?`--app-*` |
| `hydrocore-fe/src/utils/public.js` | 通用图表/数据工具，`singleForecast`、`multiForecast` 改为中�?series/data 命名 |
| `hydrocore-fe/src/utils/constants.ts` | 前端通用常量，清理预测�?label |
| `hydrocore-fe/src/components/base/charts/options.ts` | 通用图表配置，确认无旧业务命�?|
| `hydrocore-fe/src/components/charts/CommonXaxisChart.vue` | 旧通用图表组件，保留时中性化命名 |
| `hydrocore-fe/src/components/charts/LineChart.vue` | 旧通用图表组件，保留时中性化命名 |
| `hydrocore-fe/src/assets/iconfont/iconfont.json` | 图标元数据，仅在确认未引用旧业务图标后删除；保留通用图标时记录例�?|
| `hydrocore-fe/.env` | 前端默认环境模板，不硬编码团队内网地址 |
| `hydrocore-fe/.env.development` | 前端本地开发环境模板，使用 localhost 或占位说�?|
| `hydrocore-fe/.env.production` | 前端生产环境模板，使用占位变量或部署说明 |
| `README.md` | 根文档，说明干净基线、启动、初始化、验证和非目标范�?|
| `docs/README.md` | �?docs 策略，说明产品级/跨仓库文档边�?|
| `hydrocore-be/README.md` | 后端启动、数据库、Nacos、验证和 seed 账号策略 |
| `hydrocore-be/docs/README.md` | 后端 docs 索引 |
| `hydrocore-be/docs/architecture.md` | 后端架构说明，保留历史删除说明但不作为当前能�?|
| `hydrocore-fe/README.md` | 前端 env、代理、构建、扩展入口说�?|
| `hydrocore-fe/docs/README.md` | 前端 docs 索引 |
| `openspec/changes/clean-hydrocore-baseline/tasks.md` | OpenSpec build 任务勾选位置，任务完成后更�?|

### Task 1: 基线关键字盘点与例外清单

**Files:**
- Modify: `openspec/changes/clean-hydrocore-baseline/tasks.md`
- Read only: all paths listed in File Map

**Interfaces:**
- Consumes: current repository at `base-ref: HEAD`
- Produces: a checked baseline inventory in `tasks.md` and a concrete exception list used by later tasks

- [x] **Step 1: Confirm current state**

Run:

```powershell
Set-Location D:\project\HydroCore
git status --short --branch
git rev-parse HEAD
```

Expected: status shows any existing user changes; `git rev-parse HEAD` currently exits 1 in this repository and prints `HEAD` because there are no commits yet. Do not create commits before the user chooses an execution mode.

- [x] **Step 2: Run source-entry keyword inventory**

Run:

```powershell
Set-Location D:\project\HydroCore
rg -n -i "kiln|forecast|level|pressure|GAS|temperament|predict|窑炉|预测|液位|压力" `
  hydrocore-be\src hydrocore-fe\src hydrocore-be\docs hydrocore-fe\docs README.md docs -u
```

Expected: hits are grouped into backend SQL/config, backend Java/API compatibility, frontend visible UI/theme/env, docs/history, iconfont metadata, and generated/third-party content.

- [x] **Step 3: Run focused generated-content exclusion check**

Run:

```powershell
Set-Location D:\project\HydroCore
rg -n -i "kiln|forecast|level|pressure|GAS|temperament|predict|窑炉|预测|液位|压力" `
  hydrocore-be\target hydrocore-fe\dist hydrocore-fe\pnpm-lock.yaml -u
```

Expected: if these paths exist, hits are recorded as generated/lock-file exceptions and are not edited manually.

- [x] **Step 4: Record Task 1 completion**

Edit `openspec/changes/clean-hydrocore-baseline/tasks.md` by checking:

```markdown
- [x] 1.1 对源代码、模板、SQL 和文档执行旧业务关键字盘点，覆盖 `kiln`、`forecast`、`level`、`pressure`、`GAS`、`temperament`、`predict`、`窑炉`、`预测`、`液位`、`压力`
- [x] 1.2 将每个命中项分类为源代码行为、初始化数据、环境模板、历史文档、生成产物、第三方图标元数据或可删除残�?- [x] 1.3 定义允许保留的历史文档例外，并在最终验证记录中列出所有仍有意保留的旧业务术语
```

- [x] **Step 5: Commit if execution mode allows commits**

Run:

```powershell
git add openspec/changes/clean-hydrocore-baseline/tasks.md
git commit -m "chore: record clean baseline inventory"
```

Expected: one local commit containing only `tasks.md` inventory check marks. Skip this step if the selected execution mode says no commits.

### Task 2: 后端初始化数据与配置模板清理

**Files:**
- Modify: `hydrocore-be/src/main/resources/db/schema/hydrocore_schema.sql`
- Modify: `hydrocore-be/src/main/resources/db/migration/hydrocore_menu_migrate.sql`
- Modify: `hydrocore-be/src/main/resources/db/migration/iot_collection_tables_alter_rtu.sql`
- Modify: `hydrocore-be/src/main/resources/bootstrap.yml`
- Modify: `hydrocore-be/src/main/resources/nacos/hydrocore.yml`
- Modify: `hydrocore-be/src/main/resources/nacos/hydrocore-constant.yml`
- Modify: `hydrocore-be/src/main/resources/nacos/hydrocore-config.properties`
- Modify: `hydrocore-be/src/main/resources/nacos/README.md`
- Delete: `hydrocore-be/src/main/resources/testJson/forecast/forecast.json`
- Delete: `hydrocore-be/src/main/resources/testJson/forecast/queryCommonChartData.json`
- Delete: `hydrocore-be/src/main/resources/testJson/forecast/queryTemperature.json`
- Modify or Delete: `hydrocore-be/src/main/resources/testJson/monitoring/getModelData.json`
- Modify: `openspec/changes/clean-hydrocore-baseline/tasks.md`

**Interfaces:**
- Consumes: Task 1 exception list
- Produces: backend baseline SQL/config that installs only generic system foundation and does not enable old forecast/control fixtures by default

- [x] **Step 1: Search backend SQL/config/test fixtures**

Run:

```powershell
Set-Location D:\project\HydroCore
rg -n -i "kiln|forecast|level|pressure|GAS|temperament|predict|窑炉|预测|液位|压力" `
  hydrocore-be\src\main\resources\db `
  hydrocore-be\src\main\resources\nacos `
  hydrocore-be\src\main\resources\bootstrap.yml `
  hydrocore-be\src\main\resources\testJson -u
```

Expected: all default schema/seed/config hits are either removed, renamed to HydroCore-neutral terms, or explicitly classified as historical migration comments.

- [x] **Step 2: Clean `hydrocore_schema.sql`**

Remove DDL and seed data for these old baseline tables and defaults:

```text
level_algorithm_result
level_predicted_data
temperature_predict
old forecast/control/level/pressure sys_config rows
old forecast/control/level/pressure menu rows
old business role rows and role-menu bindings
old project demo accounts that are not the minimal admin/local demo seed
```

Keep generic foundation objects:

```text
sys_user
sys_role
sys_menu
sys_permission
sys_organization
sys_config
sys_user_role
sys_role_menu
device mapping / realtime tables only if they are generic point-mapping foundation
TDengine / common data query support tables only if they are not old business defaults
```

Expected: a fresh import creates login/system-management baseline only, with no old forecast/control/level/pressure defaults.

- [x] **Step 3: Mark migration scripts as historical**

At the top of `hydrocore-be/src/main/resources/db/migration/hydrocore_menu_migrate.sql`, ensure the header states:

```sql
-- Historical migration only: removes obsolete menus/roles from old installations.
-- This file is not part of the fresh HydroCore baseline capability set.
```

At the top of `hydrocore-be/src/main/resources/db/migration/iot_collection_tables_alter_rtu.sql`, add the same style of historical-or-generic scope comment if the script remains necessary. Delete old business rows from these scripts only when they are not required to migrate old installs safely.

- [x] **Step 4: Clean Nacos/bootstrap defaults**

In `bootstrap.yml`, `nacos/hydrocore.yml`, `nacos/hydrocore-constant.yml`, and `nacos/hydrocore-config.properties`:

```yaml
spring:
  application:
    name: hydrocore
```

Keep HydroCore data IDs:

```text
hydrocore.yml
hydrocore-constant.yml
hydrocore-config.properties
```

Remove or disable old default blocks:

```text
forecast.test-mode.enabled
forecast.level
control test switches
pressure/temperature prediction defaults
internal production host values
```

Expected: local development values are localhost or explicit placeholders; production-sensitive values are placeholders with documentation.

- [x] **Step 5: Remove forecast test JSON default dependency**

Delete the `hydrocore-be/src/main/resources/testJson/forecast/` files. If `DataServiceImpl` still needs a local mock for generic chart queries, create a new neutral fixture in:

```text
hydrocore-be/src/main/resources/testJson/common/queryCommonChartData.json
```

The neutral fixture must use generic keys such as:

```json
{
  "dataCode": "sample_data_code",
  "name": "Sample Series",
  "values": []
}
```

Expected: no runtime path references `testJson/forecast/*`.

- [x] **Step 6: Verify backend resource cleanup**

Run:

```powershell
Set-Location D:\project\HydroCore
rg -n -i "forecast|level|pressure|GAS|temperament|predict|窑炉|预测|液位|压力" `
  hydrocore-be\src\main\resources -u
```

Expected: matches are limited to historical migration comments, documented historical Nacos migration notes, or intentionally retained external compatibility notes.

- [x] **Step 7: Record Task 2 completion**

Check these lines in `openspec/changes/clean-hydrocore-baseline/tasks.md`:

```markdown
- [x] 2.1 清理 `hydrocore-be/src/main/resources/db/schema/hydrocore_schema.sql`，确保默�?schema 和种子数据不再安装旧窑炉、液位、压力、预测或弱示例凭据相关内�?- [x] 2.2 复查 `hydrocore-be/src/main/resources/db/migration/`，仅保留用于旧安装迁移到干净基线的脚本，并把历史迁移注释标记清晰
- [x] 2.4 清理后端 Nacos �?`bootstrap.yml` 模板，使默认配置名称保持 HydroCore 中性语义，且旧预测或旧控制测试数据不作为基线能力启�?```

### Task 3: 后端系统配置与通用数据 API 中性化

**Files:**
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/module/system/enums/SysConfigModuleEnum.java`
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/module/system/constants/SysConfigCodeConstants.java`
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/module/system/processor/ConfigAssembler.java`
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/module/system/service/impl/SysConfigServiceImpl.java`
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/sec/sevice/DataService.java`
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/sec/sevice/impl/DataServiceImpl.java`
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/tdengine/service/TaosDataService.java`
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/tdengine/service/TaosDataServiceImpl.java`
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/core/common/config/ThreadPoolConfig.java`
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/common/constant/ConstantBase.java`
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/common/constant/ConstantField.java`
- Modify: `hydrocore-be/docs/api/sys-config-api.md`
- Modify: `openspec/changes/clean-hydrocore-baseline/tasks.md`

**Interfaces:**
- Consumes: Task 2 neutral fixtures and current system configuration API contracts
- Produces: backend code/API docs whose default modules and examples are generic and compile cleanly

- [x] **Step 1: Add focused backend Java/API search**

Run:

```powershell
Set-Location D:\project\HydroCore
rg -n -i "FORECAST|CONTROL|forecast|control|level|pressure|temperament|predict|GAS|预测|液位|压力" `
  hydrocore-be\src\main\java hydrocore-be\docs\api -u
```

Expected: all hits in system config enums/constants/docs are changed; hits in external compatibility methods are documented in the exception list.

- [x] **Step 2: Replace backend config module enum**

Change `SysConfigModuleEnum.java` to:

```java
package com.siact.hydrocore.module.system.enums;

/**
 * System configuration modules exposed by the HydroCore baseline.
 */
public enum SysConfigModuleEnum {
    SYSTEM,
    INTEGRATION
}
```

Expected: no `CONTROL` or `FORECAST` enum values remain in backend defaults.

- [x] **Step 3: Replace backend config code constants**

Change `SysConfigCodeConstants.java` to contain only generic constants used by the system baseline:

```java
package com.siact.hydrocore.module.system.constants;

public class SysConfigCodeConstants {
    public static final String SYSTEM_DISPLAY_NAME = "system_display_name";
    public static final String LOCAL_DEMO_ACCOUNT_NOTICE = "local_demo_account_notice";
    public static final String INTEGRATION_SAMPLE_ENDPOINT = "integration_sample_endpoint";
}
```

Expected: no `temperament_predict_menus`, `control_target_points`, `level_control_datacodes`, or temperature alarm constants remain.

- [x] **Step 4: Update config assembler/service branches**

In `ConfigAssembler.java` and `SysConfigServiceImpl.java`, remove any special-case branches for `FORECAST`, `CONTROL`, prediction menus, or control target points. Keep behavior as:

```java
// allowed modules are defined by SysConfigModuleEnum; unknown modules return empty result or validation error through existing service flow.
```

Expected: `SysConfigModuleEnum.SYSTEM` and `SysConfigModuleEnum.INTEGRATION` are the only module values referenced by system config code.

- [x] **Step 5: Neutralize common data compatibility naming**

In `DataService.java`, `DataServiceImpl.java`, `TaosDataService.java`, and `TaosDataServiceImpl.java`:

1. If a method is only serving a generic chart query, rename internal comments and local variables from forecast/control wording to `series`, `chart`, `timeseries`, or `sample`.
2. If a method signature such as `queryForecastIntervalVal(...)` is required by an external DTO/interface contract, keep the signature and add this comment above the method:

```java
// Compatibility path for the external data API. HydroCore baseline does not expose prediction as a default business capability.
```

3. Remove old default fixture path `testJson/forecast/queryCommonChartData.json`; use `testJson/common/queryCommonChartData.json` if Task 2 kept a neutral mock.

Expected: old forecast naming remains only where changing it would break an external API contract, and every such retention is explained.

- [x] **Step 6: Neutralize backend thread/constant names**

In `ThreadPoolConfig.java`, rename the prediction-specific bean and thread prefix if no caller requires the old bean name:

```java
@Bean(name = "backgroundTaskExecutor")
public Executor backgroundTaskExecutor() {
    executor.setThreadNamePrefix("Background-Task-");
}
```

If callers still require `algorithmPredictionExecutor`, keep the bean name and add the compatibility comment from Step 5.

In `ConstantBase.java` and `ConstantField.java`, delete old gas/prediction alarm constants that are not referenced. For referenced constants, rename to generic names such as `SAMPLE_SERIES` or move the exception into the validation record.

- [x] **Step 7: Update backend system config API docs**

In `hydrocore-be/docs/api/sys-config-api.md`, replace module examples with:

```markdown
| `SYSTEM` | 通用系统配置 |
| `INTEGRATION` | 外部集成占位配置 |
```

Use example config rows:

```json
{
  "scCode": "system_display_name",
  "module": "SYSTEM",
  "scName": "系统显示名称",
  "description": "HydroCore 基线显示名称"
}
```

Expected: no default API examples mention forecast, pressure, control, temperature prediction, or old point menus.

- [x] **Step 8: Backend compile and unit tests**

Run:

```powershell
Set-Location D:\project\HydroCore\hydrocore-be
mvn -q -DskipTests compile
mvn -q test
```

Expected: both commands exit 0. If Maven dependency resolution fails because the private Nexus is unavailable, record the exact failing artifact and rerun after dependency access is restored.

- [x] **Step 9: Record Task 3 completion**

Check these lines in `openspec/changes/clean-hydrocore-baseline/tasks.md`:

```markdown
- [x] 2.3 替换或移�?`SysConfigModuleEnum`、`SysConfigCodeConstants`、相关组装器和后端系统配�?API 文档中的旧系统配置模块与编码
- [x] 2.5 在不破坏保留的通用数据 API 前提下，删除或中性化后端预测/控制测试 JSON、服务注释和示例说明
- [x] 2.6 运行后端编译和测试，并修复后端清理造成的任何失�?```

### Task 4: 前端系统配置、主题变量、图表命名和环境模板清理

**Files:**
- Modify: `hydrocore-fe/src/models/system/config/config.enums.ts`
- Modify: `hydrocore-fe/src/views/system/config/index.tsx`
- Modify: `hydrocore-fe/src/views/system/config/components/config-form.tsx`
- Modify: `hydrocore-fe/src/styles/themes.css`
- Modify: `hydrocore-fe/src/styles/tailwind.css`
- Modify: `hydrocore-fe/src/utils/public.js`
- Modify: `hydrocore-fe/src/utils/constants.ts`
- Modify: `hydrocore-fe/src/components/base/charts/options.ts`
- Modify: `hydrocore-fe/src/components/charts/CommonXaxisChart.vue`
- Modify: `hydrocore-fe/src/components/charts/LineChart.vue`
- Modify: `hydrocore-fe/src/assets/iconfont/iconfont.json`
- Modify: `hydrocore-fe/.env`
- Modify: `hydrocore-fe/.env.development`
- Modify: `hydrocore-fe/.env.production`
- Modify: `openspec/changes/clean-hydrocore-baseline/tasks.md`

**Interfaces:**
- Consumes: Task 3 backend config module set: `SYSTEM`, `INTEGRATION`
- Produces: frontend default UI and theme naming with no old forecast/control/pressure/level/kiln surface

- [x] **Step 1: Search frontend residues**

Run:

```powershell
Set-Location D:\project\HydroCore
rg -n -i "kiln|forecast|level|pressure|GAS|temperament|predict|窑炉|预测|液位|压力" `
  hydrocore-fe\src hydrocore-fe\.env hydrocore-fe\.env.development hydrocore-fe\.env.production hydrocore-fe\docs hydrocore-fe\README.md -u
```

Expected: all hits in active UI, theme, models, constants, env and chart helpers are removed or renamed; iconfont-only hits are checked for actual usage before retention.

- [x] **Step 2: Replace frontend config enum**

Change `hydrocore-fe/src/models/system/config/config.enums.ts` to:

```typescript
export enum SysConfigModuleEnum {
  SYSTEM = 'SYSTEM',
  INTEGRATION = 'INTEGRATION',
}

export enum SysConfigTypeEnum {
  STRING = 'STRING',
  INTEGER = 'INTEGER',
  FLOAT = 'FLOAT',
  DOUBLE = 'DOUBLE',
  DECIMAL = 'DECIMAL',
  BOOLEAN = 'BOOLEAN',
  TIMESTAMP = 'TIMESTAMP',
}
```

Expected: frontend module enum exactly mirrors backend.

- [x] **Step 3: Replace config page tabs**

In `hydrocore-fe/src/views/system/config/index.tsx`, keep only:

```tsx
<NTabPane name={SysConfigModuleEnum.SYSTEM} tab="系统配置" />
<NTabPane name={SysConfigModuleEnum.INTEGRATION} tab="集成配置" />
```

Expected: UI no longer shows control or forecast tabs.

- [x] **Step 4: Replace config form module options**

In `hydrocore-fe/src/views/system/config/components/config-form.tsx`, use:

```typescript
const moduleOptions = [
  { label: '系统配置', value: SysConfigModuleEnum.SYSTEM },
  { label: '集成配置', value: SysConfigModuleEnum.INTEGRATION },
];
```

Expected: users cannot create default config rows under old modules from the UI.

- [x] **Step 5: Rename CSS variables**

In `themes.css`, replace all `--kiln-` prefixes with `--app-`, preserving color values:

```css
:root {
  --app-primary-1: oklch(0.9629 0.0182 250.5883);
  --app-primary-2: oklch(0.88 0.0595 255.4686);
  --app-success-1: oklch(0.9802 0.0326 146.7241);
  --app-warning-1: oklch(0.9733 0.0169 61.9627);
  --app-danger-1: oklch(0.9659 0.0172 35.3449);
  --app-link-1: oklch(0.6215 0.2039 261.9758);
  --app-bg-1: oklch(0.0712 0.0137 282.35);
  --app-grey-1: oklch(0.2059 0.0059 285.871);
}
```

Apply the same prefix rewrite to every variable in the file, not only the sample above.

- [x] **Step 6: Update Tailwind theme references**

In `tailwind.css`, replace `var(--kiln-...)` with `var(--app-...)` for every color mapping. Keep existing Tailwind token names such as `--color-primary1`, `--color-fill1`, and `--color-word1`.

Expected: `rg -n "--kiln-|kiln" hydrocore-fe\src\styles` returns no matches.

- [x] **Step 7: Neutralize chart helper names**

In `hydrocore-fe/src/utils/public.js`, change:

```javascript
let keyList = ['actual', 'multiForecast', 'singleForecast']
```

to:

```javascript
let keyList = ['actual', 'seriesUpper', 'seriesLower']
```

Update all related local variable names and display labels in the same function. In `utils/constants.ts`, replace the old "预测�? label with a generic label such as `"参考�?` only if that value is still used by a generic chart. Delete the option if it belongs only to old prediction UI.

- [x] **Step 8: Review iconfont usage before editing metadata**

Run:

```powershell
Set-Location D:\project\HydroCore\hydrocore-fe
rg -n "icon-[A-Za-z0-9_-]*" src
rg -n -i "forecast|control|pressure|level|kiln|predict|窑炉|预测|压力|液位" src\assets\iconfont
```

Expected: unused old business icon metadata is removed from `iconfont.json`, `iconfont.css`, and generated iconfont JS only if the font build process supports regenerating those files. If the font files cannot be regenerated, keep iconfont metadata as a documented third-party/generated exception and do not edit binary font files.

- [x] **Step 9: Normalize env templates**

Use localhost or explicit placeholders:

```text
VITE_API_BASE_URL=http://localhost:8080
VITE_WS_BASE_URL=ws://localhost:8080
```

For production, use deploy-time placeholders:

```text
VITE_API_BASE_URL=/api
VITE_WS_BASE_URL=/ws
```

Expected: env files do not point only to team intranet addresses.

- [x] **Step 10: Frontend typecheck and build**

Run:

```powershell
Set-Location D:\project\HydroCore\hydrocore-fe
pnpm exec vue-tsc --noEmit
pnpm run build
```

Expected: both commands exit 0.

- [x] **Step 11: Record Task 4 completion**

Check these lines in `openspec/changes/clean-hydrocore-baseline/tasks.md`:

```markdown
- [x] 3.1 �?`--kiln-*` 主题变量及对�?Tailwind 引用替换�?`--app-*` 或组件语义名�?- [x] 3.2 清理前端系统配置枚举、标签页、表单、标签和常量，使默认配置界面不再暴露旧预测或压力语义
- [x] 3.3 对保留为通用图表工具的组件和工具函数进行中性命名，处理 `singleForecast`、`multiForecast`、`GAS` 等旧业务�?- [x] 3.4 复查 iconfont 和组件残留，删除未使用的旧预�?控制图标，或记录通用图标元数据被保留的原�?- [x] 3.5 用安全示例和模板替换前端硬编码环境地址，并补充本地代理配置说明
- [x] 3.6 运行前端构建，并修复前端清理造成的任何失�?```

### Task 5: 文档与仓库策略更�?
**Files:**
- Modify: `README.md`
- Modify: `docs/README.md`
- Modify: `hydrocore-be/README.md`
- Modify: `hydrocore-be/docs/README.md`
- Modify: `hydrocore-be/docs/architecture.md`
- Modify: `hydrocore-be/docs/api/sys-config-api.md`
- Modify: `hydrocore-fe/README.md`
- Modify: `hydrocore-fe/docs/README.md`
- Modify: `openspec/changes/clean-hydrocore-baseline/tasks.md`

**Interfaces:**
- Consumes: Tasks 2-4 final behavior and verification commands
- Produces: docs that explain how to run the baseline, initialize data/config, verify builds, and extend future water-treatment modules without implying old business is current capability

- [x] **Step 1: Update root README**

Ensure `README.md` answers:

```markdown
# HydroCore

HydroCore is a clean baseline for building water-treatment applications. This repository keeps the generic system foundation: authentication, RBAC, menu management, organization/user management, system configuration, generic data query plumbing, and the frontend shell.

This baseline does not include real water-treatment process models, device control algorithms, prediction algorithms, reports, or production plant data. Add those through separate OpenSpec/Comet changes.
```

Add commands:

```powershell
cd hydrocore-be
mvn -q -DskipTests compile
mvn -q test

cd ..\hydrocore-fe
pnpm install
pnpm run build
```

- [x] **Step 2: Document backend setup**

In `hydrocore-be/README.md`, include:

```markdown
## Database

Import `src/main/resources/db/schema/hydrocore_schema.sql` for a fresh local baseline. The seed account is for local development only and must be changed or recreated before production use.

## Nacos

Use the templates in `src/main/resources/nacos/`. Production secrets, database hosts, Redis hosts, and external service URLs must be supplied by the deployment environment.
```

Expected: docs no longer direct users to old kiln/forecast/control defaults.

- [x] **Step 3: Document frontend env and proxy**

In `hydrocore-fe/README.md`, include:

```markdown
## Environment

Use `.env.development` for local development. The default API endpoint is `http://localhost:8080`; production deployments should provide API and WebSocket base URLs through environment-specific config.

## Build

Run `pnpm run build` before handing off changes.
```

- [x] **Step 4: Update architecture docs**

In `hydrocore-be/docs/architecture.md`, keep the statement that old kiln-specific modules were removed as historical context. Add:

```markdown
Current baseline capability is generic system foundation only. Forecasting, level/pressure control, process-specific monitoring, and plant reporting are not part of this baseline.
```

- [x] **Step 5: Clarify local agent/IDE config policy**

In root docs, add:

```markdown
OpenSpec and Comet artifacts live in this repository because they describe product changes. Local agent, IDE, and skill installations are expected to come from the user's global environment unless a repository-local config is explicitly documented here.
```

Expected: repository-local `.agents`, `.codex`, `.claude`, `.cursor`, and duplicate skills are either absent from delivery scope or documented if retained.

- [x] **Step 6: Record Task 5 completion**

Check these lines in `openspec/changes/clean-hydrocore-baseline/tasks.md`:

```markdown
- [x] 4.1 更新根目�?README �?`docs/`，说�?HydroCore 是水处理系统二开基线，并明确真实业务模块属于非目�?- [x] 4.2 更新后端文档，覆盖数据库初始化、Nacos/bootstrap 配置、验证命令和种子凭据策略
- [x] 4.3 更新前端文档，覆�?env 文件、代理设置、构建验证和二开扩展入口
- [x] 4.4 删除或忽略根目录、前端和后端中无用途的 `.claude`、`.cursor`、`.agents`、`.codex` 与重�?skills 配置；如保留，必须写明用�?- [x] 4.5 文档�?Comet/OpenSpec/skills 由全局安装调用，本仓库只保�?OpenSpec 产物
```

### Task 6: 最终验证与交付记录

**Files:**
- Modify: `openspec/changes/clean-hydrocore-baseline/tasks.md`
- Modify: docs or verification record path chosen by current Comet convention; if no dedicated verification file exists, add a section to `openspec/changes/clean-hydrocore-baseline/tasks.md` named `## 验证记录`

**Interfaces:**
- Consumes: Tasks 1-5 completed code/docs state
- Produces: passing compile/build/test evidence and explicit residual exception list

- [x] **Step 1: Re-run global keyword scan**

Run:

```powershell
Set-Location D:\project\HydroCore
rg -n -i "kiln|forecast|level|pressure|GAS|temperament|predict|窑炉|预测|液位|压力" `
  hydrocore-be\src hydrocore-fe\src hydrocore-be\docs hydrocore-fe\docs README.md docs -u
```

Expected: remaining matches are only:

```text
historical migration comments
historical architecture/design notes
external API compatibility comments
third-party/generated iconfont metadata that is not referenced by default UI
this plan or OpenSpec artifacts describing what was removed
```

- [x] **Step 2: Backend compile**

Run:

```powershell
Set-Location D:\project\HydroCore\hydrocore-be
mvn -q -DskipTests compile
```

Expected: exit code 0.

- [x] **Step 3: Backend tests**

Run:

```powershell
Set-Location D:\project\HydroCore\hydrocore-be
mvn -q test
```

Expected: exit code 0.

- [x] **Step 4: Frontend build**

Run:

```powershell
Set-Location D:\project\HydroCore\hydrocore-fe
pnpm run build
```

Expected: exit code 0.

- [x] **Step 5: Config enum consistency check**

Run:

```powershell
Set-Location D:\project\HydroCore
rg -n "CONTROL|FORECAST" `
  hydrocore-be\src\main\java\com\siact\hydrocore\module\system `
  hydrocore-fe\src\models\system\config `
  hydrocore-fe\src\views\system\config
```

Expected: no matches.

- [x] **Step 6: Theme variable consistency check**

Run:

```powershell
Set-Location D:\project\HydroCore
rg -n "--kiln-|kiln" hydrocore-fe\src\styles
```

Expected: no matches.

- [x] **Step 7: Record verification evidence**

Append to `openspec/changes/clean-hydrocore-baseline/tasks.md`:

```markdown
## 验证记录

- `rg -n -i "kiln|forecast|level|pressure|GAS|temperament|predict|窑炉|预测|液位|压力" hydrocore-be\src hydrocore-fe\src hydrocore-be\docs hydrocore-fe\docs README.md docs -u`: PASS; remaining exceptions are listed below.
- `mvn -q -DskipTests compile` in `hydrocore-be`: PASS.
- `mvn -q test` in `hydrocore-be`: PASS.
- `pnpm run build` in `hydrocore-fe`: PASS.
- Retained exceptions:
  - Historical migration comments in `hydrocore-be/src/main/resources/db/migration/hydrocore_menu_migrate.sql`.
  - Historical architecture/design notes under `docs/superpowers/` and backend architecture docs.
  - External API compatibility methods documented with compatibility comments.
  - Iconfont metadata retained only when not referenced by default UI and not safely regenerable.
```

If a command cannot run because of missing private dependencies or unavailable local services, record the exact command, exit code, and blocking artifact/service instead of marking PASS.

- [x] **Step 8: Record Task 6 completion**

Check these lines in `openspec/changes/clean-hydrocore-baseline/tasks.md`:

```markdown
- [x] 5.1 重新执行旧业务关键字盘点，确认仅保留允许的历史文档或第三方图标元数据例外
- [x] 5.2 �?`hydrocore-be` 中运�?`mvn -q -DskipTests compile`
- [x] 5.3 �?`hydrocore-be` 中运�?`mvn -q test`
- [x] 5.4 �?`hydrocore-fe` 中运�?`pnpm run build`
- [x] 5.5 更新 Comet/OpenSpec 验证记录，写明命令、结果、剩余例外和真实水处理模块的后续建议
```

## Spec Coverage

| Requirement | Task |
|-------------|------|
| 关键字盘点、分类、例外定�?| 1 |
| 后端 schema/seed 数据清理 | 2 |
| 后端 migration 历史标注 | 2 |
| Nacos/bootstrap 模板中性化 | 2 |
| 后端系统配置枚举/常量/API 文档中性化 | 3 |
| 通用数据 API forecast 命名兼容判断 | 3 |
| seed 账号策略文档�?| 5 |
| 前端 `--kiln-*` 主题变量迁移 | 4 |
| 前端系统配置 enum/tabs/form options 与后端一�?| 4 |
| 前端通用图表工具中性命�?| 4 |
| iconfont 残留复查与例外记�?| 4, 6 |
| 前端 env 模板安全�?| 4, 5 |
| 根目�?前后端文档覆盖启动、初始化、验证、扩展入�?| 5 |
| 最终后�?compile/test、前�?build、关键字复查 | 6 |

## Placeholder / Consistency Check

- 后端配置模块只使�?`SYSTEM` �?`INTEGRATION`，前�?`SysConfigModuleEnum` 与后端一致�?- 计划中的验证命令与设计文档要求一致：`mvn -q -DskipTests compile`、`mvn -q test`、`pnpm run build`、旧业务关键字复查�?- 计划没有要求编辑生成产物、锁文件或二进制字体文件；iconfont 二进制只允许通过生成流程自然刷新�?- 计划不新增水处理业务模型、算法、真实数据、报表或领域 API�?
