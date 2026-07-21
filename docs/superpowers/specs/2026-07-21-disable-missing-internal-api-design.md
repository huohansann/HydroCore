# 禁用缺失内部 API 功能链设计

## 背景

后端基线中仍保留一组依赖内部包的功能链，但当前依赖集中没有这些内部类型：

- `com.siact.api.common.*`
- `com.siact.api.feign.*`
- `com.siact.ins.*`

受影响代码集中在 `sec` 控制器、服务、DTO、转换器、工具类，以及 TDengine 数据服务。Maven 现在可以解析项目依赖，但编译会停在这些缺失的内部 API 类型上。

## 决策

将受影响的 Java 源文件整体注释掉，并保留原始内容，方便后续恢复。禁用范围包括：

- `sec/controller/DevController.java`
- `sec/controller/PropController.java`
- `sec/controller/SecInsController.java`
- `sec/convertor/ClassConvertor2DTO.java`
- `sec/dto/PropValDTO.java`
- `sec/sevice/DataService.java`
- `sec/sevice/DevService.java`
- `sec/sevice/PropInsService.java`
- `sec/sevice/SecInsService.java`
- `sec/sevice/impl/DataServiceImpl.java`
- `sec/sevice/impl/DevServiceImpl.java`
- `sec/sevice/impl/PropInsServiceImpl.java`
- `sec/sevice/impl/SecInsServiceImpl.java`
- `sec/utils/SiactSecApiFeignUtil.java`
- `tdengine/service/TaosDataService.java`
- `tdengine/service/TaosDataServiceImpl.java`

这些文件继续保留在原路径中，每一行源码用行注释保留，避免以后恢复时需要重新构造实现。不修改 POM、Nacos、数据库配置，也不混入无关用户改动。

## 运行影响

在内部 API 依赖恢复前，以下路由和能力不可用：

- `/api/dev`
- `/api/prop`
- `/api/ins`
- 依赖禁用服务的数据查询能力
- 使用禁用内部查询类型的 TDengine 数据服务能力

其余认证、系统、Redis、Nacos、MQTT、WebSocket 和其他基线代码保持不变。

## 验证

注释禁用完成后：

1. 在 `hydrocore-be` 中运行 `mvn -q -DskipTests compile`。
2. 运行现有 Maven 测试套件。
3. 确认活跃源码中不再导入 `com.siact.api.*` 或 `com.siact.ins.*`。
4. 确认当前无关工作区改动被保留。

预期结果是后端基线可以编译，同时内部 API 功能链被明确禁用。
