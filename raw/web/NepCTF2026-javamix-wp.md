# NepCTF2026 JavaMix Writeup

## 题目简述

JavaMix 将三项漏洞串联：

1. `/fetch` 只校验重定向前的 URL，可通过 302 跳转访问内部 `/services`；
2. `DataSyncService` 的外层 `SafeObjectInputStream` 只过滤第一次反序列化，可借 Hutool 触发第二次反序列化；
3. OpenRASP 只在 HTTP 线程或 Web 调用栈中阻断进程创建，可把命令执行移到新线程绕过。

## 解题过程

### SSRF 重定向发现内部服务

`/services` 只允许内网 GET，`/fetch` 又禁止常见的十进制、八进制、IPv6 等 IP 编码绕过，但 HTTP 客户端会自动跟随重定向，且只检查初始 URL。准备一个公网 URL：

```text
HTTP/1.1 302 Found
Location: http://127.0.0.1:<port>/services
```

把它交给 `/fetch`，即可取得内部 WebService 列表和接口文档。文档暴露 `DataSyncService` 的方法名和序列化字节参数。访问控制还有一个差异：外部不能直接 GET `/services`，但可直接 POST `/services/<method>`，因此后续利用包无需继续经过 SSRF。

### Hutool 二次反序列化

服务使用 `SafeObjectInputStream` 并在 `resolveClass` 过滤已知危险类，`SignedObject` 等常见二次反序列化载体也被封禁。不过依赖中同时存在 Guava、Vaadin 和 Hutool，可组合为：

```text
HashMap.readObject
  -> Equivalence.Wrapper.hashCode
  -> FunctionalEquivalence / ToStringFunction
  -> PropertysetItem.toString
  -> NestedMethodProperty.getValue
  -> 代理对象 getter
  -> Hutool MapProxy.invoke
  -> BeanConverter / ObjectUtil.deserialize
  -> ValidateObjectInputStream.readObject
  -> 内层 TemplatesImpl
```

外层对象只包含允许的 Guava、Vaadin、Hutool 与 JDK 类型，能够通过 `SafeObjectInputStream`。构造时让 `MapProxy` 的 `margin` 字段保存内层序列化字节，并让代理实现 `Layout.MarginHandler`；Vaadin 的属性访问会调用 `getMargin()`，Hutool 为满足返回类型而把字节数组二次反序列化。内层 `ValidateObjectInputStream` 没有继承外层黑名单，于是 `TemplatesImpl` 得以执行。

核心构造关系为：

```java
HashMap<String, Object> proxyMap = new HashMap<>();
proxyMap.put("margin", innerPayload);
MapProxy mapProxy = MapProxy.create(proxyMap);

Object proxy = Proxy.newProxyInstance(
    Exploit.class.getClassLoader(),
    new Class[]{Layout.MarginHandler.class, Serializable.class},
    mapProxy
);

Object propertysetItem = createPropertysetItem(proxy, "margin");
Object outer = wrapWithGuavaHashMap(propertysetItem);
```

题目作者公开的[完整 JavaMix 源码与构造代码](https://github.com/1diot9/CTFJavaChallenge/tree/main/2026/NepCTF/JavaMix)包含服务端类和 solve 工程；正文上面的调用链已经概括其关键机制，复现不依赖其他分析文章。

### 新线程绕过 RASP

直接从反序列化或 HTTP handler 调用 `Runtime.exec`、`ProcessBuilder.start` 会被拦截。先利用内层字节码注入一个仅做文件读取/下载的拦截器，下载 RASP JAR 后分析 `RaspChecker.shouldBlock()`。它的判定只有两类：

- 当前线程名包含 `http-nio-`、`catalina-exec-`、`qtp`、`XNIO-` 等模式；
- 当前调用栈含 Servlet、Tomcat、CXF、Spring Web、Jetty、Undertow 等类。

RASP 虽然 hook 了 `Runtime`、`ProcessBuilder`、`ProcessImpl/UNIXProcess`，但所有入口最终都调用这段上下文判断。创建一个新的 `Thread`/`CmdRunner`，在线程的 `run()` 中再调用 `ProcessBuilder`，此时线程名与栈均不含 Web 上下文：

```java
CmdRunner runner = new CmdRunner(
    new String[]{"/bin/bash", "-c", "/getflag"}
);
runner.start();
runner.join(10000);
String output = runner.getOutput();
```

根目录的 `flag.txt` 不可读，但 `/getflag` 可执行；通过新线程运行它并把标准输出写回 HTTP 响应，即可取得 flag。

## 方法总结

本题三处边界都存在“只检查第一层”的问题：SSRF 只检查重定向前地址，反序列化只过滤外层类，RASP 只依据当前 HTTP 线程和栈判断。分析复杂 Java 链时，应围绕数据从哪一层进入下一层建立触发图，并区分“危险类被禁”与“能否找到新的二次解释入口”；堆积无关 gadget 名称反而容易偏离主线。
