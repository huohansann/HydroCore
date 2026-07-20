# Brainstorm Summary

- Change: audit-project-baseline
- Date: 2026-07-20

## 已确认事实

- 当前 change 处于 design 阶段，OpenSpec open artifacts 已完成并通过 strict validate。
- 目标是验证当前 HydroCore 仓库作为未来几个月二次开发基础时，哪些内容可优化、可删除、需保留或应拆为后续 change。
- 范围覆盖 `hydrocore-be`、`hydrocore-fe`、`docs`、`openspec`、数据库脚本、构建配置、生成物和本地协作配置。
- 不新增水处理业务能力、public API、业务 schema、页面或大规模架构重写。

## 候选技术方案

### 方案 A：报告优先 + 低风险清理（已确认）

先生成完整审计清单与证据，再只执行证据充分、低风险、与现有 specs 一致的清理或文档修正。涉及 public API、schema、新业务能力或架构调整的发现只记录为后续 change。

优点：能在本 change 内交付可用基线，同时避免误删。缺点：不是所有发现都会立即修完。

### 方案 B：只做审计报告（候选）

本 change 只产出审计报告和后续任务，不删除或修改应用/文档入口。

优点：风险最低。缺点：明显无用或误导性的内容会继续留在仓库，未来开发仍可能踩坑。

### 方案 C：积极清理（候选）

审计发现后尽量在本 change 内直接删除或修改，包括本地配置、文档入口、可疑依赖和旧命名。

优点：仓库更快变干净。缺点：容易扩大范围，可能误删外部兼容或部署依赖。

## 关键取舍与风险

- 删除候选必须有引用扫描、生成物属性、策略冲突或误导性语义作为证据。
- 历史 Comet/OpenSpec 产物默认作为历史记录保留，除非影响当前入口或流程。
- 私有依赖、外部兼容 API、数据库迁移脚本等需要保守分类，不能仅凭名称判断删除。
- 文档编码/可读性问题优先处理当前入口；大量历史文档修复应视情况拆分。

## 测试策略

- 后端：`mvn.cmd -q test`。
- 前端：`pnpm.cmd run build`。
- OpenSpec：`openspec.cmd validate audit-project-baseline --strict` 和状态检查。
- 审计报告：逐项核对分类、证据、建议动作、风险等级和验证方式。

## 确认的技术方案

采用方案 A：报告优先 + 低风险清理。

先生成完整审计清单与证据，再只执行证据充分、低风险、与现有 specs 一致的清理或文档修正。涉及 public API、schema、新业务能力或架构调整的发现只记录为后续 change。

## Spec Patch

暂无。当前 delta spec 足够表达审计覆盖、分类、报告和验证要求。
