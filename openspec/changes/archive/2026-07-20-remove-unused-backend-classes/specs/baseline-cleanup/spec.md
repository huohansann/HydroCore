## ADDED Requirements

### Requirement: No confirmed unused backend production classes
HydroCore backend production source SHALL NOT retain Java classes that have been confirmed unused by static reference checks, framework entry-point checks, and backend build verification.

#### Scenario: Confirmed unused backend classes are removed
- **WHEN** 后端源码清理确认某个生产类不存在静态引用，且不属于 Spring、MyBatis、配置、序列化或反射加载入口
- **THEN** 该类 MUST 被删除，或在变更记录中说明保留原因

#### Scenario: Backend remains verifiable after class removal
- **WHEN** 删除确认无用的后端生产类后运行后端验证
- **THEN** Maven 测试 MUST 通过，或失败原因 MUST 被记录为阻塞并在完成前修复
