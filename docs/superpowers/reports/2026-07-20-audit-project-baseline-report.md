# 项目基线审计报告：audit-project-baseline

## 摘要

| 维度 | 结论 |
|---|---|
| 是否适合作为未来开发基线 | 待验证 |
| 必须修复 | 0 |
| 建议优化 | 0 |
| 可删除候选 | 0 |
| 保留例外 | 0 |
| 后续 change 候选 | 0 |

## 扫描命令与证据

### Git 状态基线

| 项目 | 结果 |
|---|---|
| 当前分支 | `feature/20260720/audit-project-baseline` |
| 当前 HEAD | `ce10aa0cff99452fada528cc324f864bafcb035b` |
| 工作区状态 | 仅存在根目录本地 `.comet/` 未跟踪状态文件；不纳入提交 |
| 仓库纪律 | `hydrocore-be`、`hydrocore-fe`、文档/Comet 分别作为独立 Git 仓库管理 |

已运行命令：

```powershell
git status --short
git rev-parse HEAD
git branch --show-current
```

### 仓库入口清单

| 路径 | 类型 | 审计备注 |
|---|---|---|
| `.agents/` | 本地协作配置 | 仅记录，不作为产品源码整改对象 |
| `.comet/` | 本地 Comet 选择状态 | 未跟踪本地状态，不提交 |
| `docs/` | 仓库文档 | 需要核对入口说明和历史 Superpowers 产物边界 |
| `hydrocore-be/` | 后端项目 | Spring Boot/Maven 后端，包含 `src/`、`docs/`、`scripts/`、`target/` |
| `hydrocore-fe/` | 前端项目 | Vue/Vite/pnpm 前端，包含 `src/`、`docs/`、`public/`、`dist/`、`node_modules/` |
| `openspec/` | OpenSpec 规范 | 包含 active change 和主规格 |
| `README.md` | 根入口文档 | 需要核对是否准确表达当前基线边界 |

已运行命令：

```powershell
Get-ChildItem -Force
Get-ChildItem -Force hydrocore-be
Get-ChildItem -Force hydrocore-fe
Get-ChildItem -Force docs
Get-ChildItem -Force openspec
```

### 生成物和本地状态初判

| 路径 | 初判 | 处理原则 |
|---|---|---|
| `hydrocore-be/target/` | 后端构建生成物 | 不手工编辑，不纳入整改提交 |
| `hydrocore-fe/dist/` | 前端构建生成物 | 不手工编辑，不纳入整改提交 |
| `hydrocore-fe/node_modules/` | 依赖安装目录 | 不手工编辑，不纳入整改提交 |
| `hydrocore-be/.claude/`、`hydrocore-be/.cursor/`、`hydrocore-be/.idea/` | IDE/agent 本地配置候选 | 后续 Task 4 核对 Git 跟踪和策略冲突 |

### 后端扫描证据

已运行命令：

```powershell
rg -n -i "kiln|forecast|pressure|predict|temperature|control|legacy|deprecated|TODO|FIXME|占位|兼容" hydrocore-be\src hydrocore-be\docs hydrocore-be\pom.xml -g "!target/**"
rg -n "class ConvertUtils|class JacksonUtils|sourceToTarget|queryForecastIntervalVal|System\.out\.println|printStackTrace" hydrocore-be\src -g "!target/**"
rg -n "com\.siact\.hydrocore\.sec\.utils\.ConvertUtils|sec\.utils\.ConvertUtils|import .*ConvertUtils|ConvertUtils\." hydrocore-be\src\main\java -g "!target/**"
rg -n "JacksonUtils|ClassConvertor2DTO|UnitConversion|JepUtils|SshUtils|OkHttpUtil|MapUtils" hydrocore-be\src\main\java hydrocore-be\src\test\java -g "!target/**"
rg -n "class CachePreloader|CachePreloader|FiveBaseVo" hydrocore-be\src\main\java hydrocore-be\src\test\java
rg -n "TODO|FIXME" hydrocore-be\src\main\java hydrocore-be\src\test\java hydrocore-be\docs hydrocore-be\pom.xml -g "!target/**"
```

后端关键证据：

| 发现 | 分类 | 证据 | 当前处理 |
|---|---|---|---|
| `hydrocore-be/src/main/java/com/siact/hydrocore/config/CachePreloader.java` | 已清理 | 文件内全部业务代码均已注释，引用扫描只命中自身，并引用旧 `module.permission` 命名 | 删除该空壳源文件 |
| `hydrocore-be/src/main/java/com/siact/hydrocore/sec/vo/FiveBaseVo.java` | 已清理 | 存在已注释的 `@Pattern` 校验和对应未使用 import，容易误导后续维护者以为时间格式已启用严格校验 | 删除注释代码和未使用 import，不改变运行行为 |
| `hydrocore-be/src/main/java/com/siact/hydrocore/sec/utils/CommonHandle.java` | 已清理 | 存在旧聚合实现注释，当前逻辑使用首个时间点值；注释会误导后续维护者 | 删除注释代码，不改变运行行为 |
| `sec.utils.ConvertUtils` 与 `common.utils.ConvertUtils` | 建议优化 | 两个类名称和大部分方法重复，但引用扫描显示 `sec` 兼容服务仍直接引用 `sec.utils.ConvertUtils`，`common` 被基础模块引用 | 本轮不合并；建议后续 change 统一转换工具语义 |
| `queryForecastIntervalVal` | 保留例外 | `tdengine` 与 `sec` 服务接口和实现仍存在，并已有兼容说明；属于外部数据 API 兼容路径 | 不删除，不新增预测业务能力 |
| `hydrocore_menu_migrate.sql` 中 kiln 文案 | 保留例外 | 仅出现在迁移脚本注释和历史角色清理 SQL，用于旧安装迁移 | 不删除，保留历史迁移语义 |
| `hydrocore_schema.sql` 集成端点占位 | 保留例外 | 占位值为本地二开集成端点，不包含生产地址 | 不删除，后续如引入真实集成能力需独立 change |
| `System.out.println` / `printStackTrace` | 已验证 | 仅静态扫描测试的 forbidden 列表命中，运行时代码未命中 | 无需整改 |

### 前端扫描证据

已运行命令：

```powershell
rg -n -i "kiln|forecast|pressure|predict|temperature|control|legacy|deprecated|TODO|FIXME|Placeholder|占位|调试" hydrocore-fe\src hydrocore-fe\docs hydrocore-fe\package.json -g "!node_modules/**" -g "!dist/**"
rg -n "system/debug|views/system/debug|components/slider|components/setTable|icon-park-twotone:water-level" hydrocore-fe\src hydrocore-fe\iconify.ts -g "!node_modules/**" -g "!dist/**"
rg -n "SYSTEM_DEBUG|system\.debug|views/system/debug|debug\.vue|components/setTable|components/slider|SetTable|Slider" hydrocore-fe\src hydrocore-fe\components.d.ts -g "!node_modules/**" -g "!dist/**"
rg -n "water-level|windmill|control-filled|package-box" hydrocore-be\src hydrocore-fe\src hydrocore-fe\iconify.ts -g "!target/**" -g "!node_modules/**" -g "!dist/**"
pnpm.cmd run build
```

前端关键证据：

| 发现 | 分类 | 证据 | 当前处理 |
|---|---|---|---|
| `src/views/system/debug.vue` | 已清理 | 文件是移除后遗留占位页且内容乱码；系统路由未注册该页面；引用扫描未发现外部引用 | 删除占位页 |
| `RouteName.SYSTEM_DEBUG` | 已清理 | 仅常量定义处命中，无路由或菜单代码引用 | 删除遗留常量 |
| `src/components/slider/**` | 已清理 | 引用扫描只命中组件自身；组件含大量旧交互代码和注释，没有当前页面引用 | 删除遗留组件 |
| `src/components/setTable/**` | 已清理 | 引用扫描只命中组件自身；组件含明显乱码文案，没有当前页面引用 | 删除遗留组件 |
| `components.d.ts` 中遗留声明 | 已同步本地生成物 | 构建声明文件为忽略的生成物；删除组件后本地已移除 stale 声明，不强制纳入 Git | 不提交生成声明文件 |
| `iconify.ts` 中 `water-level` 等动态图标 | 保留例外 | 当前源码无静态引用，但图标转换插件会为动态菜单图标建立资源池，菜单图标可由后端数据驱动 | 不删除 |
| `HttpStatus.ts` deprecated 注释 | 保留例外 | 为 HTTP 语义常量注释，不代表旧业务功能 | 不删除 |
| `overview` 占位文案 | 保留例外 | 明确说明一期占位和后续工艺监控接入边界，不是误导性旧页面 | 不删除 |

### 文档、OpenSpec 和本地配置证据

已运行命令：

```powershell
rg -n -i "乱码|TBD|TODO|kiln|forecast|pressure|predict|\.claude|\.cursor|\.agents|\.codex|node_modules|target|dist" README.md docs hydrocore-be\docs hydrocore-fe\docs openspec\specs -g "!docs/superpowers/**"
Get-ChildItem -Force .comet,hydrocore-be\.claude,hydrocore-be\.cursor,hydrocore-be\.idea -ErrorAction SilentlyContinue
git status --short -- .comet
git status --short -- .claude .cursor .idea
openspec.cmd validate audit-project-baseline --strict
openspec.cmd status --change audit-project-baseline --json
rg -n -i "TBD|TODO|乱码" README.md docs hydrocore-be\docs hydrocore-fe\docs openspec\specs -g "!docs/superpowers/**"
```

文档和配置关键证据：

| 发现 | 分类 | 证据 | 当前处理 |
|---|---|---|---|
| `openspec/specs/baseline-cleanup/spec.md` Purpose 为归档 `TBD` | 已清理 | 文档扫描命中 `TBD - created by archiving...` | 改为中文目的说明 |
| `openspec/specs/runtime-contracts/spec.md` Purpose 为归档 `TBD` | 已清理 | 文档扫描命中 `TBD - created by archiving...` | 改为中文目的说明 |
| 根 README 对独立仓库边界表述不够强 | 已清理 | 原文为“可以保留各自独立 Git 历史” | 明确 `hydrocore-be`、`hydrocore-fe`、文档/Comet 是三个独立 Git 仓库 |
| 根目录 `.comet/` | 保留例外 | `git status --short -- .comet` 显示未跟踪本地状态 | 不提交 |
| 后端 `.cursor/`、`.idea/` | 保留例外 | 后端 `.gitignore` 已忽略；后端 Git 状态未显示跟踪变更 | 作为用户本地 IDE/agent 状态保留，不删除 |
| OpenSpec change 校验 | 已验证 | `openspec.cmd validate audit-project-baseline --strict` 通过 | 记录验证结果 |
| `TBD/TODO/乱码` 二次扫描 | 已验证 | 二次扫描无命中 | 当前入口文档无已知占位或乱码 |

## 必须修复

当前阶段未发现已确认必须修复项。

## 建议优化

- 后续统一 `sec.utils.ConvertUtils` 与 `common.utils.ConvertUtils` 的职责边界；当前两个类均有运行引用，本轮不做行为合并。
- 前端自动生成声明文件 `components.d.ts` 当前被 `.gitignore` 忽略；如团队希望提交该文件，应独立明确生成物纳入策略。

## 可删除候选

- `hydrocore-be/src/main/java/com/siact/hydrocore/config/CachePreloader.java`：已删除。证据为文件内无可编译类、引用扫描只命中自身、内容指向旧权限缓存预加载。
- `hydrocore-fe/src/views/system/debug.vue`：已删除。证据为路由未注册、引用扫描无外部引用、内容为乱码占位。
- `hydrocore-fe/src/components/slider/**`：已删除。证据为引用扫描无外部引用。
- `hydrocore-fe/src/components/setTable/**`：已删除。证据为引用扫描无外部引用且包含乱码文案。

## 保留例外

- `queryForecastIntervalVal`：作为外部数据 API 兼容路径保留，不作为 HydroCore 基线预测业务能力。
- `hydrocore_menu_migrate.sql` 中 kiln 迁移注释和角色清理 SQL：作为旧安装迁移语义保留。
- `hydrocore_schema.sql` 中本地集成端点占位：不包含生产地址，作为二开集成模板保留。
- `hydrocore-fe/iconify.ts` 中动态图标池：为后端菜单数据驱动图标预注册资源，源码无静态引用不等于可删除。
- `hydrocore-fe/src/views/overview/route.tsx` 中一期占位文案：表达当前基线边界，保留。
- 根目录 `.comet/`：Comet 本地选择状态，不提交。
- 后端 `.cursor/`、`.idea/`：后端仓库已忽略的用户本地配置，不提交、不删除。

## 后续 OpenSpec/Comet change 候选

- 转换工具统一：评估并收敛 `sec.utils.ConvertUtils` 与 `common.utils.ConvertUtils` 的空集合语义、异常日志和包边界。该事项可能影响兼容服务，需独立 change。
- 兼容数据 API 去留策略：评估 `sec` / `tdengine` 中外部数据 API 兼容路径的生命周期，涉及 public API 时必须独立 change。

## 前后端源码整改记录

### 后端

| 路径 | 动作 | 风险控制 |
|---|---|---|
| `hydrocore-be/src/main/java/com/siact/hydrocore/config/CachePreloader.java` | 删除空壳旧缓存预加载文件 | 无可编译类，引用扫描只命中自身 |
| `hydrocore-be/src/main/java/com/siact/hydrocore/sec/vo/FiveBaseVo.java` | 删除已注释的校验注解和未使用 import | 不改变字段、注解生效范围或 API 契约 |
| `hydrocore-be/src/main/java/com/siact/hydrocore/sec/utils/CommonHandle.java` | 删除旧聚合逻辑注释 | 不改变当前聚合实现 |

### 前端

| 路径 | 动作 | 风险控制 |
|---|---|---|
| `hydrocore-fe/src/views/system/debug.vue` | 删除未注册乱码占位页 | 系统路由未注册，引用扫描无外部引用 |
| `hydrocore-fe/src/constants/RouteName.ts` | 删除 `SYSTEM_DEBUG` 遗留常量 | 引用扫描只命中定义处 |
| `hydrocore-fe/src/components/slider/**` | 删除无引用遗留滑块组件 | 引用扫描无外部引用，构建通过 |
| `hydrocore-fe/src/components/setTable/**` | 删除无引用遗留设置表格组件 | 引用扫描无外部引用，构建通过 |

## 验证记录

| 时间 | 命令 | 目录 | 结果 | 备注 |
|---|---|---|---|---|
| 2026-07-20 | `git status --short` | 仓库根目录 | 通过 | 仅 `.comet/` 为本地未跟踪状态 |
| 2026-07-20 | `mvn.cmd -q test` | `hydrocore-be` | 通过 | 后端低风险清理后测试通过 |
| 2026-07-20 | `pnpm.cmd run build` | `hydrocore-fe` | 通过 | 前端遗留入口删除后构建通过；重复运行后仍通过 |
| 2026-07-20 | `openspec.cmd validate audit-project-baseline --strict` | 文档/Comet 仓库根目录 | 通过 | 当前 OpenSpec change 有效 |
| 2026-07-20 | `openspec.cmd status --change audit-project-baseline --json` | 文档/Comet 仓库根目录 | 通过 | planning artifacts 完整，`isComplete=true` |
| 2026-07-20 | `rg -n -i "TBD|TODO|乱码" ...` | 文档/Comet 仓库根目录 | 通过 | 二次扫描无命中 |
