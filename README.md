# HydroCore

HydroCore 是用于二次开发水处理系统的干净基础工程。本仓库保留通用系统底座：认证、RBAC、菜单管理、组织与用户管理、系统配置、通用数据查询管道，以及前端应用壳。

当前基线不包含真实水处理工艺模型、设备控制算法、预测算法、报表或生产现场数据。这些能力应通过独立的 OpenSpec/Comet 变更逐步加入。

## 仓库结构

- `hydrocore-be/`：Spring Boot 后端。
- `hydrocore-fe/`：Vue 3 前端。
- `openspec/`：OpenSpec 与 Comet 变更产物。
- `docs/`：产品级与跨仓库文档。

`hydrocore-be/`、`hydrocore-fe/`、根目录文档/Comet 是三个独立 Git 仓库。根仓库只管理协作文档以及 OpenSpec/Comet 产物，不替代前端或后端仓库的源码提交边界。

## 验证

```powershell
cd hydrocore-be
mvn -q -DskipTests compile
mvn -q test

cd ..\hydrocore-fe
pnpm install
pnpm run build
```

## Agent 与 IDE 规范

本项目只使用全局 Comet 作为 agent 工作流入口。OpenSpec 和 Comet 产物保存在本仓库中，因为它们描述的是产品变更。agent、IDE、skill 的本地安装应来自用户的全局环境。

除非未来变更明确说明原因，否则不要在仓库内新增 `.agents/`、`.codex/`、`.claude/`、`.cursor/` 或 `.github/skills/` 的本地副本。

## Git 规范

在远程仓库和分支策略准备好之前，不要求提交代码。Git 准备完成后，根仓库、后端仓库、前端仓库中同一个产品变更应使用一致的分支名称。

所有 Git commit message 必须使用中文；这是协作纪律，不使用英文提交信息。
