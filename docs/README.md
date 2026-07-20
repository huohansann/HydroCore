# HydroCore 文档

根目录下的 `docs/` 用于存放产品级和跨仓库文档。

## 目录结构

- `superpowers/`：Comet 生成的计划、设计和实现说明。
- `product/`：产品需求、工作流说明和验收上下文。
- `architecture/`：跨服务架构、集成方案和部署决策。

服务内文档保留在对应服务仓库中：

- `hydrocore-be/docs/`：后端 API、数据库、环境和部署文档。
- `hydrocore-fe/docs/`：前端组件、路由、环境和部署文档。

OpenSpec 和 Comet 产物属于仓库工作成果。agent、IDE、skill 的本地安装属于用户全局环境，不应复制到本仓库中。

## Git 提交备注纪律

所有 Git commit message 必须使用中文；这是协作纪律，不使用英文提交信息。
