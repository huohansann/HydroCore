---
change: audit-project-baseline
design-doc: docs/superpowers/specs/2026-07-20-audit-project-baseline-design.md
base-ref: e0dbc9dea87198f0676a5db2860acef73dc375d4
---

# 项目基线审计实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: 使用 `subagent-driven-development`（推荐）或 `executing-plans` 按任务逐项实施。步骤必须使用 checkbox（`- [ ]`）跟踪。

**目标：** 完成 HydroCore 前后端与仓库级基线审计，输出可执行报告，并执行证据充分的低风险优化整改。

**架构：** 采用“报告优先 + 低风险清理”流程。先用固定扫描命令和人工核对建立证据，再对前后端源码、文档和配置做低风险整改；涉及 public API、数据库 schema、新业务能力或架构重构的发现只记录为后续 change。

**技术栈：** 后端 Spring Boot/Maven/Java 8，前端 Vue 3/Vite/pnpm/TypeScript，规范与状态管理使用 OpenSpec/Comet。

## 全局约束

- 所有新增 OpenSpec、Comet、Superpowers 产物必须中文主导；必要技术术语可保留英文。
- 不新增水处理业务模块、页面、算法、业务表或领域 API。
- 不修改 public API 契约，不修改数据库 schema，不做框架或依赖体系的大规模升级。
- 不手工编辑 `target`、`dist`、`node_modules`、缓存目录等生成物。
- 删除候选必须有引用扫描、生成物属性、策略冲突、用途重复或误导性语义证据。
- 不确定项归入“保留例外”或“后续 change 候选”，不得直接删除。

---

## 文件结构

- 创建：`docs/superpowers/reports/2026-07-20-audit-project-baseline-report.md`
  - 记录审计摘要、必须修复、建议优化、可删除候选、保留例外、后续 change 候选和验证结果。
- 修改：`openspec/changes/audit-project-baseline/tasks.md`
  - 每完成一组任务后勾选对应 checkbox。
- 可能修改：`README.md`、`docs/README.md`、`hydrocore-be/docs/**`、`hydrocore-fe/docs/**`
  - 仅当当前入口文档误导未来开发或缺少边界说明时修改。
- 可能修改：`hydrocore-be/src/**`
  - 仅处理低风险源码整改，例如无引用代码、重复工具、误导命名、未说明兼容代码。
- 可能修改：`hydrocore-fe/src/**`
  - 仅处理低风险源码整改，例如无引用入口、误导占位页、重复工具或轻量结构问题。

## Task 1：建立审计证据基线

**文件：**
- 创建：`docs/superpowers/reports/2026-07-20-audit-project-baseline-report.md`
- 修改：`openspec/changes/audit-project-baseline/tasks.md`

**接口：**
- 输入：Design Doc、OpenSpec delta spec、当前 Git 状态。
- 输出：报告初稿，包含扫描命令、仓库入口清单和初始发现表。

- [ ] **Step 1：记录当前 Git 状态**

运行：

```powershell
git status --short
git rev-parse HEAD
git branch --show-current
```

预期：只允许出现当前 change 的计划/报告修改；如果出现无关文件，先按 dirty worktree 协议归因。

- [ ] **Step 2：创建报告骨架**

创建 `docs/superpowers/reports/2026-07-20-audit-project-baseline-report.md`，内容结构如下：

```markdown
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

## 必须修复

## 建议优化

## 可删除候选

## 保留例外

## 后续 OpenSpec/Comet change 候选

## 前后端源码整改记录

## 验证记录
```

- [ ] **Step 3：记录仓库入口清单**

运行：

```powershell
Get-ChildItem -Force
Get-ChildItem -Force hydrocore-be
Get-ChildItem -Force hydrocore-fe
Get-ChildItem -Force docs
Get-ChildItem -Force openspec
```

把与审计相关的目录记录到报告的“扫描命令与证据”章节。

- [ ] **Step 4：提交 Task 1**

运行：

```powershell
git add docs/superpowers/reports/2026-07-20-audit-project-baseline-report.md openspec/changes/audit-project-baseline/tasks.md
git commit -m "审计：建立项目基线证据"
```

## Task 2：后端源码与配置审计整改

**文件：**
- 修改：`docs/superpowers/reports/2026-07-20-audit-project-baseline-report.md`
- 可能修改：`hydrocore-be/src/**`、`hydrocore-be/docs/**`、`hydrocore-be/pom.xml`

**接口：**
- 输入：后端扫描结果。
- 输出：后端发现分类、低风险整改、保留例外和后续 change 候选。

- [ ] **Step 1：扫描后端旧业务和可疑兼容入口**

运行：

```powershell
rg -n -i "kiln|forecast|pressure|predict|temperature|control|legacy|deprecated|TODO|FIXME|占位|兼容" hydrocore-be\src hydrocore-be\docs hydrocore-be\pom.xml -g "!target/**"
```

将结果分类写入报告。迁移脚本、外部兼容方法和日志级别中的普通 `level` 不直接删除，先记录保留理由。

- [ ] **Step 2：扫描后端重复工具和潜在无引用代码**

运行：

```powershell
rg -n "class ConvertUtils|class JacksonUtils|sourceToTarget|queryForecastIntervalVal|System\.out\.println|printStackTrace" hydrocore-be\src -g "!target/**"
```

对每个候选再运行按符号名的引用扫描。无足够证据时写入“建议优化”或“后续 change 候选”，不得删除。

- [ ] **Step 3：执行低风险后端整改**

只允许执行满足以下条件的修改：

- 引用扫描证明无运行时引用。
- 不改变 controller 返回类型、请求路径、数据库表结构和配置键语义。
- 修改后 `mvn.cmd -q test` 可验证。

如果没有满足条件的后端整改项，在报告中写“本轮无可安全直接修改项”。

- [ ] **Step 4：运行后端验证**

运行：

```powershell
mvn.cmd -q test
```

工作目录：`hydrocore-be`。将结果写入报告“验证记录”。

- [ ] **Step 5：提交 Task 2**

运行：

```powershell
git add hydrocore-be docs/superpowers/reports/2026-07-20-audit-project-baseline-report.md openspec/changes/audit-project-baseline/tasks.md
git commit -m "审计：完成后端基线整改"
```

如果 `hydrocore-be` 没有代码变更，只提交报告和 tasks。

## Task 3：前端源码与配置审计整改

**文件：**
- 修改：`docs/superpowers/reports/2026-07-20-audit-project-baseline-report.md`
- 可能修改：`hydrocore-fe/src/**`、`hydrocore-fe/docs/**`、`hydrocore-fe/package.json`

**接口：**
- 输入：前端扫描结果。
- 输出：前端发现分类、低风险整改、保留例外和后续 change 候选。

- [ ] **Step 1：扫描前端旧业务和占位入口**

运行：

```powershell
rg -n -i "kiln|forecast|pressure|predict|temperature|control|legacy|deprecated|TODO|FIXME|Placeholder|占位|调试" hydrocore-fe\src hydrocore-fe\docs hydrocore-fe\package.json -g "!node_modules/**" -g "!dist/**"
```

将结果分类写入报告。图标元数据、HTTP 状态码 deprecated 注释、兼容占位页必须先判断是否仍被路由或菜单引用。

- [ ] **Step 2：扫描前端无引用入口**

运行：

```powershell
rg -n "system/debug|views/system/debug|components/slider|components/setTable|icon-park-twotone:water-level" hydrocore-fe\src hydrocore-fe\iconify.ts -g "!node_modules/**" -g "!dist/**"
```

对每个候选再运行引用扫描。无引用且不属于公共组件库入口时，记录为可删除候选或执行低风险删除。

- [ ] **Step 3：执行低风险前端整改**

只允许执行满足以下条件的修改：

- 不改变已注册业务路由契约。
- 不新增页面或业务能力。
- 不手工编辑 `dist`、`node_modules`、lockfile。
- 修改后 `pnpm.cmd run build` 可验证。

如果没有满足条件的前端整改项，在报告中写“本轮无可安全直接修改项”。

- [ ] **Step 4：运行前端验证**

运行：

```powershell
pnpm.cmd run build
```

工作目录：`hydrocore-fe`。将结果写入报告“验证记录”。

- [ ] **Step 5：提交 Task 3**

运行：

```powershell
git add hydrocore-fe docs/superpowers/reports/2026-07-20-audit-project-baseline-report.md openspec/changes/audit-project-baseline/tasks.md
git commit -m "审计：完成前端基线整改"
```

如果 `hydrocore-fe` 没有代码变更，只提交报告和 tasks。

## Task 4：文档、OpenSpec 和本地配置审计

**文件：**
- 修改：`docs/superpowers/reports/2026-07-20-audit-project-baseline-report.md`
- 可能修改：`README.md`、`docs/README.md`、`hydrocore-be/docs/**`、`hydrocore-fe/docs/**`

**接口：**
- 输入：文档与配置扫描结果。
- 输出：入口文档整改、保留例外和后续 change 候选。

- [ ] **Step 1：扫描当前入口文档可读性和策略冲突**

运行：

```powershell
rg -n -i "乱码|TBD|TODO|kiln|forecast|pressure|predict|\\.claude|\\.cursor|\\.agents|\\.codex|node_modules|target|dist" README.md docs hydrocore-be\docs hydrocore-fe\docs openspec\specs -g "!docs/superpowers/**"
```

当前入口文档如存在乱码或误导性边界描述，修正文档。历史 `docs/superpowers/**` 默认作为历史记录，不批量重写。

- [ ] **Step 2：扫描本地配置目录**

运行：

```powershell
Get-ChildItem -Force .comet,hydrocore-be\.claude,hydrocore-be\.cursor,hydrocore-be\.idea -ErrorAction SilentlyContinue
git status --short -- .comet hydrocore-be/.claude hydrocore-be/.cursor hydrocore-be/.idea
```

把本地状态、IDE 配置和策略冲突记录到报告。删除前必须确认这些目录是否被 Git 跟踪、是否属于用户本地状态。

- [ ] **Step 3：运行 OpenSpec 校验**

运行：

```powershell
openspec.cmd validate audit-project-baseline --strict
openspec.cmd status --change audit-project-baseline --json
```

把结果写入报告。

- [ ] **Step 4：提交 Task 4**

运行：

```powershell
git add README.md docs hydrocore-be/docs hydrocore-fe/docs docs/superpowers/reports/2026-07-20-audit-project-baseline-report.md openspec/changes/audit-project-baseline/tasks.md
git commit -m "审计：完成文档与配置核对"
```

只添加实际存在且已修改的路径。

## Task 5：最终核对与完成 build 阶段

**文件：**
- 修改：`docs/superpowers/reports/2026-07-20-audit-project-baseline-report.md`
- 修改：`openspec/changes/audit-project-baseline/tasks.md`

**接口：**
- 输入：前四个任务的提交和验证结果。
- 输出：最终审计结论、全部 tasks 勾选、build guard 通过。

- [ ] **Step 1：汇总报告结论**

更新报告摘要表，填入最终数量和结论：

- 是否适合作为未来开发基线。
- 必须修复数量。
- 建议优化数量。
- 可删除候选数量。
- 保留例外数量。
- 后续 change 候选数量。

- [ ] **Step 2：运行完整验证**

运行：

```powershell
mvn.cmd -q test
pnpm.cmd run build
openspec.cmd validate audit-project-baseline --strict
git status --short
```

Maven 命令在 `hydrocore-be` 下运行；pnpm 命令在 `hydrocore-fe` 下运行；OpenSpec 和 Git 命令在仓库根目录运行。

- [ ] **Step 3：勾选 OpenSpec tasks**

确认 `openspec/changes/audit-project-baseline/tasks.md` 中所有任务都已完成后，将 checkbox 改为 `[x]`。

- [ ] **Step 4：提交 Task 5**

运行：

```powershell
git add docs/superpowers/reports/2026-07-20-audit-project-baseline-report.md openspec/changes/audit-project-baseline/tasks.md
git commit -m "审计：完成项目基线审计"
```

- [ ] **Step 5：运行 build guard**

运行：

```powershell
comet.cmd guard audit-project-baseline build --apply
```

预期：guard 通过并将 phase 推进到 `verify`。

## 自检

- `project-baseline-audit` 的覆盖、分类、报告和验证要求分别对应 Task 1 到 Task 5。
- 前后端源码优化整改分别对应 Task 2 和 Task 3。
- 高风险 API、schema、新业务能力和架构问题只记录为后续 change。
- 计划中没有 `TBD`、`TODO` 或需要实现者自行猜测的占位项。
