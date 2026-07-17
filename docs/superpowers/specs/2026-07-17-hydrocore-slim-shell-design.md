# HydroCore 一期：去窑炉瘦身 + 水处理空壳 + 工程更名

**日期：** 2026-07-17  
**状态：** 已审阅通过（2026-07-17）  
**范围：** 就地改造现有前后端；不做水处理工艺业务实现；不接 3D 孪生场景

---

## 1. 背景与目标

现有仓库为窑炉智控系统（`kic-be` / `kic-fe`，原 kiln-intelligent-control）。目标是在此底座上二开为**水处理系统 HydroCore**。

一期目标（已确认）：

1. **去除窑炉业务与相关前端页面**（含 level / pressure）
2. **保留数字孪生数据交互**（`com.siact.sec` + TDengine/`dataCode`）；不接 3D/UE 场景
3. **搭水处理空壳入口**：总览 + 趋势 + 系统管理
4. **包名、服务名、工程名改为 HydroCore**
5. **旧 git 远程断开**；新远程建好后再 push（本地保留 `.git` 历史）

明确不做（一期）：

- 水处理工艺建模、控制下发、告警引擎
- 液位/压力业务（模块删除，不迁移）
- 新仓从零脚手架拷贝
- 向任何远程 push（直到提供新库地址）

---

## 2. 决策摘要

| 项 | 决定 |
|----|------|
| 交付形态 | B：瘦身 + 空壳（总览 / 趋势 / 系统管理） |
| 孪生深度 | A：后端孪生 API + 趋势页接 `queryCommonChartData`；无 3D |
| 删除策略 | 硬删窑炉专用 + level/pressure；保留 system / device / sec / tdengine / iot(底座) |
| 改造路径 | 就地改造（方案 1），不旁路留死代码、不新建空仓 |
| 命名 | HydroCore：`hydrocore-fe` / `hydrocore-be`，服务名 `hydrocore`，Java 根包 `com.siact.hydrocore` |
| Git | 删除旧 `origin` remote；保留本地 `.git`；新库就绪后再 add + push |

---

## 3. 目标架构

```
┌─────────────────────────────────────────────────────────┐
│  hydrocore-fe (Vue3)                                    │
│  /overview 空壳 │ /trends 图表 │ /system/* 已有管理      │
└─────────────────────┬───────────────────────────────────┘
                      │ HTTP / STOMP(WS 底座保留)
┌─────────────────────▼───────────────────────────────────┐
│  hydrocore-be (Spring Boot)                             │
│  保留: system | device | sec | tdengine | iot | core    │
│  删除: kiln / forecast / gas·wind / level / pressure 等 │
│  孪生: /api/data/*                                      │
└─────────────────────┬───────────────────────────────────┘
                      │
              数字孪生平台 / TDengine(dataCode)
```

技术栈不更换：Spring Boot 2.6 + Java 8 + MyBatis-Plus + Redis + TDengine + Nacos；Vue3 + Vite + Pinia + Naive/Element。

---

## 4. 命名与工程更名

| 项 | 原 | 目标 |
|----|----|------|
| 目录 | `kic-fe` / `kic-be` | `hydrocore-fe` / `hydrocore-be` |
| npm name | `kic` | `hydrocore` |
| Maven artifactId | kiln 相关 | `hydrocore` |
| Java 根包 | `com.siact.*`（业务混在 module 下） | 迁至 `com.siact.hydrocore.*`（实施计划中分步；外部 Feign 依赖名若无法改则保留调用侧适配） |
| spring.application.name / Nacos | kiln-intelligent-control 等 | `hydrocore` |
| 主启动类 | `KilnApplication` | `HydrocoreApplication` |
| 产品文案 | 窑炉智控 | 水处理 / HydroCore |

说明：若一次性改全 Java package 成本过高，允许「工程名/服务名/目录先改，package 在同阶段内用机械替换完成」；一期结束时**不得再保留 Kiln 主类名与 kiln 服务名**。

MySQL 物理库名：新库未建好时配置可用占位；库名建议后续 `hydrocore`，不在本期阻塞代码更名。

---

## 5. 删除 / 保留清单

### 5.1 前端删除

- 路由与页面：`monitor`、`forecast`、`control`、`model`、`config`（控制区间/约束）、窑炉相关 analysis/yield/energy 等
- 请求与模型：`request/control/*`、`request/forecast`、`request/monitor` 及对应 models
- 窑炉业务组件（天然气/助燃风、窑炉监控大屏等）

### 5.2 前端保留

- 登录鉴权、布局壳、HTTP/WS、Pinia、通用 base 组件
- `system` 全套（用户/角色/菜单/组织/点位/实时/配置）
- `request/charts` → `api/data/queryCommonChartData`

### 5.3 后端删除

- `KilnInfo*`、窑炉 Monitor 大屏、`forecast` 整包、`control`（含天然气/助燃风/KilnPublish/控制规则）
- `level`、`pressure`、`predicted`、`model`、`algorithm`、`monitoring`、`snapshot`、`process` 整包删除
- `base` 中窑炉专用类（Kiln*、温度告警若仅服务窑炉温度等）；通用 Dic/缓存等保留并迁入新包结构
- 表：`kiln_info`、level/pressure/temperature_predict 及相关配置表；菜单中窑炉项
- 例外：删除后若 `system`/`device`/`sec` 编译仍依赖某文件，将该文件抽到保留包，而不是整包回滚

### 5.4 后端保留

- `system`、`device`、`sec`（数字孪生）、`tdengine`、`iot`/`mqtt`（无窑炉耦合的底座）、`core`/`common`/`config`

实施验收以**编译通过**为准；删除过程中按编译/引用错误收敛。

---

## 6. 空壳页、路由、菜单

菜单由后端 `sys_menu` 驱动；路由 `name` 与 `menu_code` 对齐。

| menu_code / route name | path | 行为 |
|------------------------|------|------|
| `overview` | `/overview` | 总览空壳：标题「水处理总览」+ 说明 + 占位；无假数据、无窑炉组件 |
| `trends` | `/trends` | 点位 `dataCodes` + 时间范围 → 现有图表 API；失败/无数据真实提示 |
| `system.*` | `/system/...` | 保持现有 |

- 登录默认落地：`overview`（替代原 monitor）
- SQL：删除窑炉菜单；插入 overview/trends；admin 角色关联更新
- Logo/侧栏文案改为 HydroCore / 水处理

---

## 7. 数字孪生边界

**保留：**

- `sec` 包与 `/api/data/*`（含 `queryCommonChartData`）
- TDengine 按 `dataCode` 查询能力
- 设备点位映射（`device`）作为点位主数据入口

**不做：**

- 嵌入 UE/3D 场景、iframe 孪生容器
- 为水处理新造孪生业务 API（一期只用现有通用查询）

---

## 8. Git 策略

1. `hydrocore-fe`、`hydrocore-be` 分别 `git remote remove origin`（或等价去掉旧远程）
2. 保留本地 `.git` 与提交历史，本地可继续 commit
3. **不** push，直到用户提供新 Git 库地址后：`remote add` + `push -u`
4. 不采用删除 `.git` 清历史（除非用户后续明确要求）

工作区注意：曾出现前端 `src/` 被误删未提交；实施前确认 `src/` 完整（可从当前 HEAD 恢复）。

---

## 9. 验收标准

1. 后端 `mvn compile`、前端类型检查/构建通过，无已删模块引用
2. 登录后侧栏仅总览、趋势、系统管理（及下属），无窑炉菜单
3. `/overview` 为占位页，无窑炉业务、无编造工艺数据
4. `/trends` 在有效 `dataCode` 与环境下可出曲线；失败时真实报错
5. 系统管理（含点位映射/实时查询）可用
6. 工程目录/artifact/应用名/服务名为 HydroCore；无 `KilnApplication`、无 kiln 服务名
7. 旧 git remote 已移除；未向旧窑炉远程 push

---

## 10. 风险

| 风险 | 应对 |
|------|------|
| `base` 等包内窑炉与通用代码缠在一起 | 按编译错误拆分；通用留 common/core |
| Java 包全局重命名易漏 | 机械替换 + 全量编译；Feign 外部包名不可改则仅改本仓库 |
| 菜单/角色与代码不同步 | 提供清理+插入 SQL 并执行 |
| Nacos/DB 未就绪 | 配置占位；不阻塞更名与删代码；趋势页真实失败提示 |
| iot/mqtt 隐含窑炉耦合 | 扫引用；纯底座留，业务耦合删 |

---

## 11. 一期之后（非本期范围）

- 水处理工艺/控制/告警真实功能
- MySQL 新库落地与正式库名
- 新 Git 远程创建并首次 push
- 可选：3D 孪生场景接入
