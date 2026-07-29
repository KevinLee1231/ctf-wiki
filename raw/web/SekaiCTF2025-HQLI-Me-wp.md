# hqli-me

## 题目简述

题目包含两个 Java/Hibernate 服务：

- 订单服务监听外部请求；
- 认证服务仅监听本机，保存用户和会话。

订单接口把可控 `fields` 拼到 HQL 投影：

```java
var sql = "select %s from Order o where o.username=\"%s\""
    .formatted(fields, authnUser);
```

过滤器错误地使用：

```java
Pattern.matches("\\W", token)
```

它只会拒绝“整个 token 恰好是一个非单词字符”的情况，复杂 HQL 表达式完全可以通过。Hibernate HQL 支持 `new ClassName(...)` 动态实例化，因此投影注入可调用 JDK 类构造器，最终借 JShell/JDI 达到代码执行。

## 解题过程

### 1. 登录并获得普通会话

先用默认账户：

```text
guest / guest
```

经订单服务 `/login` 获取合法 session ID。后续 `/orders` 只要求 session 能在认证服务查到用户，不限制投影字段必须来自 `Order` 实体。

### 2. 用 HQL 实例化 `JdiInitiator`

官方投影形如：

```text
new jdk.jshell.execution.JdiInitiator(
  0,
  new java.util.ArrayList(0),
  "...",
  true,
  "localhost",
  3000000,
  new map(... as main, ... as includevirtualthreads)
)
union select ...
```

Hibernate 会真实调用该构造器。`JdiInitiator` 用给定的 main class 和调试参数启动 Java 子进程，成为从“构造任意对象”到“启动 JDK 工具”的桥梁。

第一阶段启动：

```text
jdk/tools/jlink/internal/Main --save-opts /tmp/lol
```

并通过 `-p` 参数中的换行把一段 Java 表达式写进 `/tmp/lol`，使该文件同时可作为 JShell 输入。

### 3. 第二次构造触发 JShell

再次请求 `/orders`，实例化另一个 `JdiInitiator`，这次主类为：

```text
jdk/internal/jshell/tool/JShellToolProvider /tmp/lol
```

JShell 读取第一阶段生成的文件并执行：

```java
Runtime.getRuntime().exec(new String(new byte[]{...}));
```

命令按字节数组构造，避免 HQL、Java 字符串和 shell 三层引号互相干扰。

### 4. 从订单服务横向请求认证服务

题目禁止出网，但不妨碍访问 `127.0.0.1:8000`。执行的 shell 命令向认证服务 `/login` 发送恶意 password，直接绕过订单服务对用户名和密码的前置规范化。

认证服务同样把参数拼入 HQL。载荷调用 H2 函数：

```text
function('CSVWRITE', ...)
```

让 H2 执行包含多条语句的查询，创建 Java alias `SHELLEXEC`。该 alias：

1. 用 `ProcessBuilder("/flag")` 执行只可执行的 flag 程序；
2. 读取其标准输出；
3. 把输出嵌入新用户的用户名；
4. 创建攻击者指定的会话 ID `a`。

载荷还在用户名末尾追加引号与连接运算符，使它被订单服务再次拼入 HQL 时仍能形成可控表达式。

### 5. 通过新会话读回

最后以：

```text
sessionId = a
fields = 1||'
```

请求 `/orders`。订单服务先从认证服务取得包含 flag 的用户名，再把它拼入查询，结果字符串中出现 flag。

整条链没有依赖外部网络：

```text
HQL 投影注入
→ JdiInitiator
→ jlink 写 JShell 脚本
→ JShell 执行 Runtime.exec
→ localhost 认证服务 HQL 注入
→ H2 CREATE ALIAS
→ 执行 /flag
→ 会话用户名回显
```

仓库正式认证服务中的 `/flag` 程序输出：

```text
SEKAI{hql1nj3ct10n_unh4rm0n1zed}
```

## 方法总结

本题说明 HQL 注入不止能读数据库。Hibernate 的动态实例化语法会把投影变成 Java 构造器调用，而 JDK 自带类中存在能启动调试 VM 和工具主类的高危 gadget。

字段白名单必须逐 token 做完整匹配，例如只接受预定义列名集合，不能用“是否包含非字母数字”的宽松正则。内部认证服务也必须把 HQL 参数化；仅靠监听 localhost 不能防止已被攻陷的同容器服务。
