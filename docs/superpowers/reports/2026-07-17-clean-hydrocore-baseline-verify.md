# 验证报告：clean-hydrocore-baseline

## 摘要

| 维度 | 状态 |
|---|---|
| 完整性 | 通过：OpenSpec 任务 25/25 已完成，Superpowers 计划步骤已勾选 |
| 正确性 | 通过：代码审查发现的问题已修复，基线清理要求已重新验证 |
| 一致性 | 通过：实现符合 HydroCore 干净二开基线范围 |

## 验证证据

| 检查项 | 结果 |
|---|---|
| 在 `hydrocore-be` 执行 `mvn -q -DskipTests compile` | 通过 |
| 在 `hydrocore-be` 执行 `mvn -q test` | 通过 |
| 在 `hydrocore-fe` 执行 `pnpm.cmd run build` | 通过 |
| 对源码和文档执行旧业务关键词扫描 | 通过，仅剩已记录例外 |
| 扫描前后端系统配置枚举中的 `CONTROL` / `FORECAST` | 通过，无匹配 |
| 扫描前端样式中的 `--kiln-` / `kiln` | 通过，无匹配 |
| 扫描旧预测、控制常量和 fixture 路径 | 通过，无匹配 |
| 针对 fresh schema 复查弱 seed、报表/工艺 schema、无路由菜单 | 通过，无阻塞残留 |

## 审查问题处理

### Critical

当前无遗留 Critical 问题。

本轮 verify 已修复：

- 将 fresh schema 用户 seed 改为一个本地开发专用管理员账号，使用 BCrypt hash，并设置 `account='admin'`。
- 删除孤儿 `sys_user_role` 和 `sys_user_organization` 关联数据。
- 新增 `sys_config` 表结构，以及中性的 `SYSTEM` / `INTEGRATION` 初始配置。

### Important

当前无遗留 Important 问题。

本轮 verify 已修复：

- 删除 fresh schema 中的报表和模型参数字典 seed。
- 删除 fresh schema 中未作为当前基线能力交付的 `report` 表。
- 删除 fresh schema 中超出基线范围的 `process_log` 工艺日志表。
- 删除 seed 中的 `system.debug` 菜单和角色绑定，因为前端没有注册对应路由。

### Minor

已修复：

- 前端 `SysConfigTypeEnum` 新增 `OBJECT` 和 `ARRAY`，与后端系统配置类型保持一致。

## 保留例外

- `docs/superpowers/` 下历史 Comet/OpenSpec 文档会描述已移除的旧业务，作为项目历史保留。
- `hydrocore-be/src/main/resources/db/migration/hydrocore_menu_migrate.sql` 保留旧业务词，用于旧安装迁移清理。
- `queryForecastIntervalVal(...)` 保留在外部兼容接口中，注释已说明预测不是当前基线能力。
- `level` 出现在 Logback 和 Nacos 的日志/配置键中，与液位业务无关。
- `AccessLevel` 出现在 Lombok 注解中，与液位业务无关。
- `hydrocore-fe/src/assets/iconfont/iconfont.json` 可能保留生成的历史图标元数据，因为本次未重新生成字体资产。

## 结论

可以进入 archive 和提交。当前基线已经通过后端编译、后端测试和前端构建；fresh schema 保留通用 HydroCore 系统底座，不再携带旧窑炉、预测、控制、液位、压力业务默认项。
