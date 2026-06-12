# d3icu

## 题目简述

题目组合了 Tomcat Redis session 反序列化和旧版 Chromium RCE。Tomcat 使用 Redis 保存序列化后的 JDK session，攻击者如果能向 Redis 写入任意二进制 session，就能通过伪造 `JSESSIONID` 触发 CommonsCollections 反序列化；随后改写 Tomcat 的 `index.jsp` 做 watering hole，让题目中的 headless Chromium 89 访问页面并触发 CVE-2021-21220，最终在浏览器容器侧拿 flag。

## 解题过程

Tomcat 有多种 session 持久化方案，其中一种是把 session 数据存入 Redis。

本题使用 `tomcat` - `cluster` - `redis` - `session` - `manager` 库。这个库的关键行为是：Tomcat session 对象会先经过标准 Java 序列化再写入 Redis；当同一个 `JSESSIONID` 再次被加载时，数据会被反序列化。

阅读源码可以发现，session 数据写入 Redis 前会使用 JDK 提供的序列化功能，读取时则会进行反序列化。

Redis 中的存储格式是 `session_id: serialized_binary_data`。

会话 ID 的值来自 `JSESSIONID` cookie。

如果攻击者能向 Redis 写入任意数据，就可以发起反序列化攻击。

为了让攻击可行，题目额外加入了 `CommonsCollections 3.1` 依赖。

同时，缓存程序提供了把 HTTP 响应内容缓存到 Redis 的功能。

后续会有一个 headless browser 访问 Tomcat 中的 `/demo/index.jsp`，并截图返回给用户。

这是一个 Node.js 应用。如果阅读比较仔细，会注意到

`package.json` 中的 `puppeteer` 版本很旧。每个 `puppeteer` 版本只

兼容特定版本的 Chromium。当前使用的 `puppeteer` 版本是 6.0.0，对应 Chromium 89。

这个版本的 Chromium 存在远程代码执行漏洞 CVE-2021-21220。

具体解题步骤如下：

1. 使用 cache 服务向 Redis 写入恶意构造的二进制数据。

2. 修改 cookie 触发反序列化，在 Tomcat 容器中获得 RCE。

3. 修改 `index.jsp` 实现 watering hole 攻击。

4. 让 Chromium 访问 `index.jsp`，触发浏览器 RCE 并获取 flag。

另外，题目配置了负载均衡，共有三个 Tomcat 容器提供 HTTP 服务。为了稳定触发 Chromium RCE，需要把三个 Tomcat 容器中的 `index.jsp` 都改写掉。

复现所需的关键信息如下：

- `tomcat-cluster-redis-session-manager` 会在 Redis 中保存 `session_id -> Java serialized session bytes`，因此任意 Redis 写入加可控 `JSESSIONID` 就构成 Java 反序列化 primitive。
- Apache Commons Collections 3.1 提供经典的 transformer/invoker gadget 构造块，到达 Java 反序列化点后足以完成命令执行。
- CVE-2021-21220 影响 Chromium 89，也就是 Puppeteer 6.0.0 绑定的 Chromium 版本。利用方向是让浏览器访问攻击者控制页面后触发浏览器 RCE。
- Chromium exploit 文章的价值在于提供了可落地的页面侧触发方式；在本题中，这个页面通过覆盖负载均衡后三个 Tomcat 的 `index.jsp` 来投递。

## 方法总结

- 核心技巧：Redis session 任意写、Java 反序列化、CommonsCollections gadget、JSP watering hole、旧版 Chromium RCE。
- 识别信号：看到 Tomcat session 落 Redis 且值是 JDK serialized bytes，应立即检查是否能写 Redis；看到 Puppeteer 旧版本，应根据其绑定 Chromium 版本查浏览器漏洞。
- 复用要点：负载均衡有三个 Tomcat 容器，稳定触发浏览器漏洞需要把三个 `index.jsp` 都改掉，否则 headless browser 可能访问到未被污染的节点。
