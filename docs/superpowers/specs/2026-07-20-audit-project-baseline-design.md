---
comet_change: audit-project-baseline
role: technical-design
canonical_spec: openspec
archived-with: 2026-07-20-audit-project-baseline
status: final
---

# audit-project-baseline 技术设计

## 背景

HydroCore 已完成运行时契约统一和基础清理，但接下来几个月的开发都会以当前仓库为扩展基础。此时需要做一次系统化基线审计，确认仓库中哪些内容应优化、删除、保留或拆为后续 change。

本设计遵循已确认方案：**报告优先 + 低风险清理**。本 change 不引入新业务能力、不修改 public API、不新增业务 schema，也不做大规模架构重写。

## 目标

- 形成可复核的审计证据链，而不是只给主观建议。
- 覆盖后端、前端、文档、OpenSpec/Comet、数据库脚本、构建配置、生成物和本地协作配置。
- 将发现分为必须修复、建议优化、可删除候选、保留例外、后续 change 候选。
- 将前后端源码优化整改纳入范围，但只处理低风险、证据充分、不会改变 public API、数据库 schema 或业务能力边界的问题。
- 只执行低风险清理和当前入口文档修正。
- 用后端测试、前端构建、OpenSpec 校验和审计报告一致性作为完成标准。

## 非目标

- 不新增水处理业务模块、页面、算法、业务表或领域 API。
- 不把私有依赖替换、框架升级、数据库 schema 调整等高风险工作混入本 change。
- 不手工编辑 `target`、`dist`、`node_modules`、缓存目录等生成物。
- 不批量重写历史 Comet/OpenSpec 文档；历史文档仅在误导当前入口或破坏流程时处理。

## 执行模型

审计采用四步闭环：

1. 盘点入口：按根目录、后端、前端、文档、OpenSpec/Comet 和生成物分层收集证据。
2. 分类判断：每个发现必须进入固定分类，并记录证据与风险。
3. 低风险处理：只处理引用清楚、风险低、符合现有 specs 的清理、源码整改或文档修正。
4. 验证收敛：运行后端测试、前端构建、OpenSpec 校验，并核对审计报告。

## 分类规则

### 必须修复

满足以下任一条件：

- 当前构建、测试、OpenSpec 校验或 Comet 状态机会失败。
- 当前入口文档或配置会明显误导后续开发。
- 当前仓库策略明确禁止，且不存在保留理由。

### 建议优化

满足以下任一条件：

- 不阻塞运行，但会增加后续开发理解成本。
- 结构重复、命名不清、依赖关系不透明。
- 可改善验证、启动、文档或协作体验。

### 可删除候选

删除前必须具备至少一种证据：

- 引用扫描无结果。
- 属于生成物或本地缓存。
- 与仓库协作策略冲突。
- 与当前基线语义冲突，并且不是历史迁移说明。
- 与其他文件职责重复，且没有独立用途。

### 保留例外

保留项必须记录原因。常见原因包括：

- 历史迁移脚本需要描述旧业务。
- 外部兼容 API 暂时不能改名。
- OpenSpec/Comet 归档产物属于生命周期记录。
- 缺少足够证据证明可以安全删除。

### 后续 change 候选

以下发现不得在本 change 内实现：

- public API 变更。
- 数据库 schema 变更。
- 新业务能力或真实水处理领域模型。
- 框架、依赖、构建系统的大规模升级。
- 需要跨模块架构重新设计的问题。

## 审计入口

### 根目录

- `README.md`
- `.gitignore`
- `.comet/` 本地状态与项目配置
- OpenSpec/Comet 当前 change 状态
- 本地协作配置策略

### 后端

- `hydrocore-be/pom.xml`
- `hydrocore-be/src/main/java`
- `hydrocore-be/src/main/resources`
- 数据库初始化和迁移脚本
- `hydrocore-be/docs`
- 后端测试与验证脚本

后端源码整改重点包括：

- 无引用或明显过时的类、方法、配置入口。
- 重复工具类或重复实现。
- 未说明的外部兼容方法。
- 误导后续开发的旧业务命名或占位逻辑。
- 不改变 API 契约和 schema 的轻量结构整理。

### 前端

- `hydrocore-fe/package.json`
- `hydrocore-fe/src`
- `hydrocore-fe/public`
- `.env*` 环境模板
- `vite.config.ts`
- `hydrocore-fe/docs`
- 生成物与类型声明策略

前端源码整改重点包括：

- 无引用页面、组件、路由或公共工具入口。
- 误导性的占位页面或旧业务残留实现。
- 重复配置、重复工具和可收敛的轻量组件结构问题。
- 环境模板与源码默认值不一致的问题。
- 不改变路由契约和用户可见业务能力的低风险整理。

### 文档与 OpenSpec

- `docs/README.md`
- `docs/product`
- `docs/architecture`
- `docs/superpowers`
- `openspec/specs`
- `openspec/changes/archive`

历史目录默认只作为上下文输入；除非影响当前入口，否则不批量清理。

## 审计报告结构

报告应写入 `docs/superpowers/reports/2026-07-20-audit-project-baseline-report.md`，包含：

- 摘要：总体结论、是否适合作为未来开发基线。
- 必须修复：路径、证据、处理结果、验证方式。
- 建议优化：路径、证据、建议动作、风险等级。
- 可删除候选：路径、引用证据、删除风险、建议处理阶段。
- 保留例外：路径、保留原因、未来风险。
- 后续 change 候选：建议 change 名称、范围、非目标、依赖关系。
- 验证记录：后端、前端、OpenSpec、Git diff 核对结果。

## 测试与验证

- 后端：在 `hydrocore-be` 运行 `mvn.cmd -q test`。
- 前端：在 `hydrocore-fe` 运行 `pnpm.cmd run build`。
- OpenSpec：运行 `openspec.cmd validate audit-project-baseline --strict`。
- Comet：设计阶段通过 `comet guard audit-project-baseline design --apply`，后续 build/verify 阶段按状态机推进。
- Git：最终 diff 只允许包含当前 change artifacts、审计报告、前后端低风险源码整改和已批准的文档/配置清理。

## 风险与缓解

- 误删外部部署依赖：删除前做引用扫描；不确定项归入保留例外或后续 change。
- 审计范围过大：只处理基线质量问题；架构或业务能力问题拆分。
- 历史文档旧业务描述造成误判：历史归档默认保留，当前入口优先。
- 文档编码问题扩散：只修当前入口和本 change 新增报告；历史文档另行评估。
- 私有依赖处理过度：本 change 只记录风险和后续建议，不直接替换依赖源。

## 后续计划输入

实施计划应围绕以下顺序展开：

1. 建立审计清单和扫描命令。
2. 执行后端、前端、文档、OpenSpec/Comet 分层盘点。
3. 生成报告并分类。
4. 处理证据充分的前后端低风险源码整改、配置清理或文档修正。
5. 运行验证并更新报告结论。
