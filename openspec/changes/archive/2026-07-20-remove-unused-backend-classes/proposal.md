## Why

后端源码中可能保留了已经不参与编译路径或运行路径的类，增加维护成本，并容易误导后续二开判断可用边界。需要在不改变现有 API、数据库 schema 和运行时契约的前提下删除确认无用的类。

## What Changes

- 识别 `hydrocore-be` 生产源码中无引用、无 Spring/MyBatis/配置反射入口、无序列化或外部契约依赖的类。
- 删除确认无用的后端类及其直接关联的空包或无效引用。
- 保持现有 public API、数据库 schema、配置契约和测试行为不变。
- 运行后端 Maven 测试，确保清理后仍可编译并通过验证。

## Capabilities

### New Capabilities

- 无。

### Modified Capabilities

- `baseline-cleanup`: 增加后端生产类清理要求，约束确认无用的类不得保留，并要求清理后后端验证通过。

## Impact

- 影响范围：`hydrocore-be/src/main/java` 中确认无用的生产类，以及必要时同步清理关联测试或导入。
- 不影响：前端、数据库 schema、外部 API、运行时响应契约、Redis 管理契约。
- 验证：后端测试命令 `mvn.cmd -q test`。
