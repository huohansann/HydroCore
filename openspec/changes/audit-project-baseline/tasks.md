## 1. 审计盘点

- [ ] 1.1 记录仓库入口和当前 Git 状态，覆盖根目录文档、`hydrocore-be`、`hydrocore-fe`、`docs`、`openspec`、生成物目录和本地配置目录。
- [ ] 1.2 盘点后端依赖、构建插件、资源模板、数据库脚本、测试和后端文档，识别优化或删除候选项。
- [ ] 1.3 盘点前端依赖、构建脚本、环境模板、生成文件、路由、复用组件、公共资产和前端文档。
- [ ] 1.4 盘点 OpenSpec/Comet 产物、归档 change、根目录协作策略和本地 IDE/agent 配置目录。

## 2. 发现分类

- [ ] 2.1 将审计发现分类为必须修复、建议优化、可删除候选、保留例外或后续 change 候选。
- [ ] 2.2 为每个可删除候选记录引用扫描证据、生成物属性、策略冲突、用途重复或误导性基线语义。
- [ ] 2.3 为每个保留例外记录保留原因，以及未来开发者需要知道的风险。
- [ ] 2.4 将涉及 public API、数据库 schema、新业务能力或架构级调整的发现拆分为后续 change 候选，不在本 change 内实现。

## 3. 基线清理

- [ ] 3.1 只执行有审计证据和当前 specs 支撑的低风险清理。
- [ ] 3.2 当当前入口文档存在误导、不可读或缺少必要基线边界时，更新对应文档。
- [ ] 3.3 避免手工编辑 `target`、`dist`、`node_modules`、缓存和 lockfile 等生成物，除非包管理器或构建工具要求。

## 4. 审计报告

- [ ] 4.1 创建基线审计报告，记录影响路径、分类、证据、建议动作、风险等级和验证方法。
- [ ] 4.2 报告必须包含优化候选、删除候选、保留例外和后续 OpenSpec/Comet change 的独立章节。
- [ ] 4.3 将报告与 `runtime-contracts`、`baseline-cleanup` 和本 change 的 `project-baseline-audit` 需求交叉核对。

## 5. 验证

- [ ] 5.1 在 `hydrocore-be` 下运行后端验证：`mvn.cmd -q test`。
- [ ] 5.2 在 `hydrocore-fe` 下运行前端验证：`pnpm.cmd run build`。
- [ ] 5.3 对 `audit-project-baseline` 运行 OpenSpec status/validation，并在验证记录中保存结果。
- [ ] 5.4 确认最终 Git diff 只包含当前 change 的审计产物和已批准的清理/报告更新。
