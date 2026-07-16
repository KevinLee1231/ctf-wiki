# spring

## 题目简述

应用使用 Spring Boot Actuator，并把全部管理端点暴露到 Web。`/actuator/env` 中的 `app.username` 提示 flag 位于 `app.password`，但含 `password` 关键字的配置值会被 Actuator 脱敏显示为星号，需要从 JVM 堆转储中恢复原值。

## 解题过程

访问 `/actuator/env`，可以看到：

```text
app.username = flag_is_the_password
app.password = ******
```

星号只是响应阶段的脱敏结果，真实字符串仍保存在 Spring `Environment` 对应的堆对象中。题目配置将 `management.endpoints.web.exposure.include` 设为 `*`，只排除了 `shutdown`、`refresh` 和 `restart`，因此可以直接下载 HPROF：

```text
GET /actuator/heapdump
```

将响应保存为 `heapdump` 后，可用 JDumpSpider 搜索 Spring 配置源。该工具会解析 `MapPropertySource`、`OriginTrackedMapPropertySource` 等常见对象并提取其中的配置键值；项目地址保留为工具获取入口：[whwlsfb/JDumpSpider](https://github.com/whwlsfb/JDumpSpider)。

```powershell
java -jar ".\JDumpSpider-full.jar" ".\heapdump"
```

关键输出为：

```text
OriginTrackedMapPropertySource
app.password = 0xGame{1abbac75-e230-4390-9148-28c71e0098b9}
app.username = flag_is_the_password
```

也可以使用 Eclipse MAT，在堆中查询键为 `app.password` 的 `LinkedHashMap.Entry`：

```sql
SELECT *
FROM java.util.LinkedHashMap$Entry entry
WHERE toString(entry.key).contains("app.password")
```

仓库内 `application.yml` 进一步确认该配置的明文值与堆中结果一致。

## 方法总结

Actuator 的配置脱敏只保护 `/env` 的展示结果，并不会擦除 JVM 中的原始对象；一旦 `/actuator/heapdump` 未受保护，密码、令牌和连接信息仍可能从堆中恢复。本题的完整链路是识别 Actuator、确认管理端点暴露、下载堆转储，再按 Spring 配置源对象定位 `app.password`。
