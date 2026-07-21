# 前后端联调整改与验证报告

## 摘要

| 维度 | 状态 |
| --- | --- |
| 后端统一响应 | 已适配：前端按 `ApiResponse` 解包，成功码使用 `200` |
| 登录与菜单跳转 | 已整改：登录 token 存储、当前用户/菜单解包、无权限兜底跳转修正 |
| 系统管理页面 | 已验证：用户管理、角色管理、组织管理、菜单管理查看接口通过 |
| CRUD | 已验证：系统配置直连后端和前端代理均通过创建、查询、更新、删除 |
| 中文编码 | 已验证：新写入中文经后端和前端代理读回正常，没有变成 `???` |

## 背景

后端完成统一返回结构改造后，接口返回由业务对象改为：

```json
{
  "success": true,
  "code": 200,
  "message": "操作成功",
  "data": {}
}
```

前端仍有部分代码按旧结构读取 `res.data`，导致页面在后端返回正确时仍报错，或显示成功但实际业务数据未正确落到前端状态中。同时用户反馈登录跳转、菜单名称问号、MySQL 中文 `???`、用户/角色/组织管理页面查看报错，需要做一次端到端联调验证和整改。

## 整改清单

### 前端

- `src/libs/http/envelope.ts`
  - `unwrapApiEnvelope` 成功码从旧的 `HttpStatus.SUCCESS` 调整为后端实际 `HttpStatus.OK`。
- `src/store/useAuthStore.ts`
  - 登录接口已返回解包后的 token，存储逻辑改为 `Bearer ${res}`。
- `src/store/useUserStore.ts`
  - 当前用户和菜单列表改为使用解包后的响应对象，不再读取 `res.data`。
- `src/plugins/router.ts`
  - 菜单权限检查前增加菜单数组归一化。
  - 无权限兜底路由从父级 `HOME` 调整到可访问的 `OVERVIEW`。
- `src/request/system/config.ts`
  - 配置刷新接口改为 `sysconfig/${scCode}/refresh`，与后端路径一致。
- `src/views/system/realtime/components/query-form.tsx`
  - 实时查询下拉选项改为使用解包后的数组。
- `src/views/trends/route.tsx`
  - 趋势查询参数调整为后端可识别的 `tsUnit: 'MIN'`、`calcType: 'AVG'`。
- `vite.config.ts`
  - 开发代理增加前缀重写，`/hydrocore/...` 正确转发到后端根路径。
- `scripts/verify-http-contract.mjs`
  - 扩展前端 HTTP 契约静态校验，覆盖 ApiResponse 解包、登录/菜单、系统配置刷新、趋势接口、后端图表路由暴露。

### 后端

- `src/main/java/com/siact/hydrocore/sec/controller/DataController.java`
  - 恢复活动的 `POST /api/data/queryCommonChartData` 接口，并统一返回 `ApiResponse<CommonChartResultDto>`。
- `src/main/java/com/siact/hydrocore/sec/service/DataChartService.java`
- `src/main/java/com/siact/hydrocore/sec/service/impl/DataChartServiceImpl.java`
  - 新增图表查询服务，封装 TDengine 查询、参数大小写归一化和空结果返回。
- `src/main/java/com/siact/hydrocore/module/system/service/impl/SysConfigServiceImpl.java`
  - 修复配置项 PATCH 更新：改用 `LambdaUpdateWrapper` 按 `scCode + scPath + version` 更新，避免 `@Version` 与 `updateById` 组合导致乐观锁条件不匹配。
- `src/test/java/com/siact/hydrocore/architecture/RuntimeContractStaticScanTest.java`
  - 增加运行时契约扫描，覆盖系统表字段、MySQL UTF-8 配置、趋势图接口暴露。
- `src/test/java/com/siact/hydrocore/module/system/service/impl/SysConfigServiceImplTest.java`
  - 增加配置项 PATCH 乐观锁更新测试。
- `src/test/java/com/siact/hydrocore/sec/service/impl/DataChartServiceImplTest.java`
  - 增加图表空结果返回测试。

## 联调接口矩阵

| 场景 | 方法 | 路径 | 结果 |
| --- | --- | --- | --- |
| 登录 | `POST` | `/auth/login` | 通过 |
| 当前用户 | `GET` | `/auth/current` | 通过 |
| 当前用户菜单 | `GET` | `/auth/menus` | 通过 |
| 用户管理列表 | `POST` | `/sysuser/list` | 通过 |
| 角色管理列表 | `POST` | `/sysrole/list` | 通过 |
| 组织管理列表 | `POST` | `/sysorg/list` | 通过 |
| 组织树 | `GET` | `/sysorg/tree` | 通过 |
| 菜单管理列表 | `POST` | `/sysmenu/list` | 通过 |
| 菜单树 | `GET` | `/sysmenu/tree` | 通过 |
| 系统配置模块列表 | `GET` | `/sysconfig/module/SYSTEM` | 通过 |
| 系统配置创建 | `POST` | `/sysconfig` | 通过 |
| 系统配置查询 | `GET` | `/sysconfig/{scCode}` | 通过 |
| 系统配置项查询 | `GET` | `/sysconfig/{scCode}/path/{scPath}` | 通过 |
| 系统配置项更新 | `PATCH` | `/sysconfig/{scCode}/path/{scPath}` | 通过 |
| 系统配置刷新 | `POST` | `/sysconfig/{scCode}/refresh` | 通过 |
| 系统配置删除 | `DELETE` | `/sysconfig/{scCode}` | 通过 |

## 验证证据

- 后端 focused 测试：
  - `mvn.cmd -q -DskipTests=false '-Dtest=RuntimeContractStaticScanTest,DataChartServiceImplTest,SysConfigServiceImplTest' test`
  - 结果：退出码 0。
- 前端 HTTP 契约：
  - `pnpm.cmd run verify:http-contract`
  - 结果：`frontend http contract verification passed`，退出码 0。
- 前端构建：
  - `pnpm.cmd run build`
  - 结果：`vue-tsc && vite build` 通过，退出码 0。
- 真实 HTTP 联调：
  - 后端直连：`http://localhost:9010`
  - 前端代理：`http://localhost:9712/hydrocore`
  - 结果：`auth=ok reads=ok crud=ok chinese=ok cleanup=ok`。

## 中文编码结论

本次通过真实 HTTP 写入临时中文配置并读回，验证当前接口链路和 MySQL 当前连接对新写入中文正常：

- 配置名：`联调配置`
- 描述：`中文往返校验`
- 配置值：`水务平台`
- PATCH 后配置值：`前端代理中文`
- refresh 后配置值：`刷新后中文`

如果数据库中已有历史记录已经保存为 `???`，原始中文已经丢失，需要重新导入或重新编辑；本次整改确认的是当前新写入链路不会继续产生 `???`。

## 提交边界

本报告对应前后端联调整改和验证。提交时只应纳入与本次整改相关的文件；后端仓库中存在其他既有改动，不能使用整仓 `git add -A` 混入无关变更。
