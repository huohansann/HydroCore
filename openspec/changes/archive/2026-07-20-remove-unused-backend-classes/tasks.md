## 1. Identify Candidates

- [x] 1.1 扫描 `hydrocore-be` 生产源码，列出无引用或疑似遗留的类候选。
- [x] 1.2 对候选类检查 Spring/MyBatis/配置/序列化/反射入口，确认删除风险。

## 2. Remove Unused Backend Classes

- [x] 2.1 删除确认无用的后端类，并清理直接关联的无效导入或空引用。
- [x] 2.2 保持现有 API、schema、配置和运行时契约不变。

## 3. Verify

- [x] 3.1 运行 `mvn.cmd -q test` 验证后端编译和测试通过。
- [x] 3.2 记录删除清单与验证结果。

## Implementation Record

### Deleted Backend Classes

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

### Retained Backend Event Types

- `DomainEvent`, `BaseEvent`, `DomainEventBase`, and `GenericEvent` are retained because the event subsystem still uses the `DomainEvent` contract, and the concrete/base event helpers are part of that framework surface even though current business code does not publish them directly.

### Verification

- Exact class-name scan with `rg --word-regexp` found no remaining references to deleted class names under `hydrocore-be/src`.
- `mvn.cmd -q test` passed in `hydrocore-be`.
