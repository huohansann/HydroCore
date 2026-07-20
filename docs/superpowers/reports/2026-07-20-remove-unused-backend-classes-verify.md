# 验证报告：remove-unused-backend-classes

## 摘要

| 维度 | 状态 |
|------|------|
| 完整性 | 6/6 个任务已完成，1 个 delta requirement 已检查 |
| 正确性 | 已删除确认无用的后端类，精确引用扫描无残留 |
| 一致性 | 实现符合 tweak 设计；未改变 public API、数据库 schema 或事件契约 |

## 验证证据

| 检查项 | 结果 | 证据 |
|--------|------|------|
| OpenSpec 产物 | PASS | `openspec.cmd status --change "remove-unused-backend-classes" --json` 显示 proposal、design、specs、tasks 均已完成 |
| 任务完成度 | PASS | `openspec.cmd instructions apply --change "remove-unused-backend-classes" --json` 显示 6/6 个任务完成 |
| 删除类引用 | PASS | 在 `hydrocore-be/src` 下执行 `rg --word-regexp`，未发现已删除类名的残留引用 |
| 后端测试 | PASS | 在 `hydrocore-be` 下执行 `mvn.cmd -q test`，退出码为 0 |
| 范围控制 | PASS | 仅删除确认无用的后端类；未修改 API、schema、资源或配置契约 |

## 删除清单

本次删除 39 个后端生产类：

- `common/config/BigDecimalTrimmingConverter.java`
- `common/constant/ConstantBase.java`
- `common/constant/ConstantField.java`
- `common/dto/BaseSecApiRtnDTO.java`
- `common/dto/SecApiProjectDTO.java`
- `common/dto/SecApiSystemDTO.java`
- `common/entity/BaseEntity.java`
- `common/enums/DataTypeEnum.java`
- `common/enums/ResponseEnum.java`
- `common/enums/ShortCodeEnum.java`
- `common/enums/StatusEnum.java`
- `common/repository/BaseRepository.java`
- `common/repository/BaseRepositoryImpl.java`
- `common/utils/JepUtils.java`
- `common/utils/OkHttpUtil.java`
- `common/utils/SshUtils.java`
- `common/vo/ReportCellDataVo.java`
- `common/vo/ReportDataPageVo.java`
- `common/vo/ReportDataVo.java`
- `common/vo/ReportRowDataVo.java`
- `module/base/dto/EnergyFeeModelStyleDTO.java`
- `module/base/dto/FeeModelComputeMarkRtnDTO.java`
- `module/base/dto/FeeModelDTO.java`
- `module/base/dto/FeeModelValFeePriceRtnDTO.java`
- `module/base/dto/Load.java`
- `module/base/dto/ProjectPropDTO.java`
- `module/base/dto/SecPropBaseDTO.java`
- `module/base/vo/AlarmPointVO.java`
- `module/base/vo/HistoryConfigChartQueryVO.java`
- `module/base/vo/ProjectPropVO.java`
- `module/base/vo/WaterSystemVO.java`
- `module/enmus/ModelStatusEnum.java`
- `module/enmus/PublishStatusEnum.java`
- `module/system/constants/SysConfigCodeConstants.java`
- `module/system/enums/MenuTypeEnum.java`
- `sec/dto/CloumChartParmsDTO.java`
- `sec/dto/DevModelTypeDTO.java`
- `sec/dto/ParamsDto.java`
- `sec/dto/StpropInsDTO.java`

## Requirement 覆盖

### No confirmed unused backend production classes

- 静态引用扫描确认这些类没有外部 Java 引用。
- 资源目录和测试目录的字符串扫描未发现候选类名引用。
- 框架入口检查已排除 Spring、MyBatis、配置、序列化和事件契约类型。
- 删除后 `mvn.cmd -q test` 通过。

## 保留类型

`DomainEvent`、`BaseEvent`、`DomainEventBase` 和 `GenericEvent` 已保留。事件子系统仍通过 publisher、handler 和 interceptor 使用 `DomainEvent` 契约。`DomainEventBase` 与 `GenericEvent` 当前没有业务发布调用，但仍属于事件框架辅助类型，本次保守清理不删除。

## 无关工作区改动

`hydrocore-be/src/main/java/com/siact/hydrocore/core/event/config/EventProperties.java` 存在一个无关的注释空格改动。根据用户选择，该改动保留但不纳入本次 change 的验证评估。

## 问题

### CRITICAL

- 无。

### WARNING

- 无。

### SUGGESTION

- 无。

## 最终结论

本次后端无用类清理的范围内检查全部通过。完成归档前最终确认后，可以进入 archive 阶段。
