# EzQl

## 题目简述

题目是一个 Java HTTP 服务，`/ql` 接收 Base64 编码的 QLExpress 表达式并执行。Docker 使用 OpenJDK 8，启动 `ezql.jar`，flag 脚本将 flag 写入 `/this_is_flag`，权限为全用户可读。

反编译 `org.example.Main$1` 可确认核心逻辑：

```java
String express = getRequestBody(req);
express = new String(Base64.getDecoder().decode(express));
ExpressRunner runner = new ExpressRunner();
QLExpressRunStrategy.setForbidInvokeSecurityRiskMethods(true);
Set<String> secureMethods = new HashSet();
secureMethods.add("java.lang.Integer.valueOf");
QLExpressRunStrategy.setSecureMethods(secureMethods);
response = String.valueOf(runner.execute(express, context, null, false, false));
```

## 解题过程

### 关键机制

虽然开启了 `setForbidInvokeSecurityRiskMethods(true)`，但表达式仍可实例化对象、访问属性、触发 getter/setter。JAR 中包含 `ActiveMQ`、`Shiro`、`commons-collections` 等依赖，因此可以走 Java gadget 链。

外链 CTFCON 议题的关键内容是：ActiveMQ 的 `ActiveMQObjectMessage#getObject()` 会对 `content` 中的字节流做二次反序列化；结合 Shiro `IniEnvironment` 的配置属性解析，可以在设置/读取属性时触发任意 getter/setter，从而在不出网的情况下触发 commons-collections 反序列化链。

参考 URL：https://github.com/CTFCON/slides/blob/main/2024/Make%20ActiveMQ%20Attack%20Authoritative.pdf

### 求解步骤

有两条利用思路：

1. 非预期：利用 QLExpress 允许创建 `JdbcRowSetImpl` 并设置 `dataSourceName`，触发 JNDI lookup。
2. 预期：构造 Shiro INI 配置，创建 `ActiveMQObjectMessage`，设置 `content` 为序列化 payload，设置 `trustAllPackages=true`，最后访问 `object` 属性触发 `getObject()` 反序列化。

预期 payload 的结构如下：

```ini
[main]
byteSequence = org.apache.activemq.util.ByteSequence
byteSequence.data = <base64 serialized commons-collections payload>
byteSequence.offset = 0
byteSequence.length = <payload length>
activeMQObjectMessage = org.apache.activemq.command.ActiveMQObjectMessage
activeMQObjectMessage.content = $byteSequence
activeMQObjectMessage.trustAllPackages = true
activeMQObjectMessage.object.a = x
```

将上述 INI 作为 QLExpress 表达式中的字符串传入 `IniEnvironment`，表达式 Base64 后 POST 到 `/ql`。payload 命令读取 `/this_is_flag` 或将其外带即可。

## 方法总结

- 识别点：QLExpress 执行用户表达式、依赖中存在 ActiveMQ/Shiro/commons-collections。
- 核心绕过：安全方法白名单限制的是直接危险调用，但属性访问仍能触发 JavaBean getter/setter。
- 预期链：`IniEnvironment` 解析配置 -> 设置 `ActiveMQObjectMessage.content` -> 访问 `object` -> `getObject()` 二次反序列化。
