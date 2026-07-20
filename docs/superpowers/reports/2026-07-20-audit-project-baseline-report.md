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

## 必须修复

当前阶段未发现已确认必须修复项。

## 建议优化

待后端、前端和文档扫描后补充。

## 可删除候选

待引用扫描和生成物属性核对后补充。

## 保留例外

待兼容入口、本地配置和历史产物核对后补充。

## 后续 OpenSpec/Comet change 候选

待审计发现分类后补充。涉及 public API、数据库 schema、新业务能力或架构级调整的事项只进入本章节，不在当前 change 内实现。

## 前后端源码整改记录

待 Task 2 和 Task 3 执行后补充。

## 验证记录

| 时间 | 命令 | 目录 | 结果 | 备注 |
|---|---|---|---|---|
| 2026-07-20 | `git status --short` | 仓库根目录 | 通过 | 仅 `.comet/` 为本地未跟踪状态 |
