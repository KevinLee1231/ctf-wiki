# ezCC

## 题目简述

`ezCC_Official.pdf` 记录了一个 Java Commons Collections 反序列化接口。用户输入长度限制为 6000，服务端对输入字符串和 `resolveClass` 的类名分别做 `contains` 黑名单，阻止多种系统执行类、Jackson 与 `ChainedTransformer`，并试图防御 UTF-8 overlong encoding 和原生 Jackson 链。

PDF 给出的预期路线是组合 Commons Collections gadget：以 CC6 的 `HashMap`/`TiedMapEntry`/`LazyMap` 触发框架，直接布置 `InvokerTransformer("newTransformer")`，由 `TemplatesImpl` 定义攻击者 bytecode；字节码再安装 Tomcat Filter 内存马，以 HTTP header 回显命令结果。

## 解题过程

### 反序列化触发链

PDF 第 1 页给出的调用链为：

```text
HashMap.put/hash
  -> TiedMapEntry.hashCode/getValue
  -> LazyMap.get
  -> InvokerTransformer.transform
  -> TemplatesImpl.newTransformer
  -> defineTransletClasses / defineClass
```

构造时先让 `LazyMap` 使用无害 `ConstantTransformer(1)` 完成 `HashMap.put`，随后移除 `templates` 键并反射把 `factory` 改为 `InvokerTransformer("newTransformer")`，避免在生成端提前触发。`TemplatesImpl._bytecodes` 放入继承 `AbstractTranslet` 的最小类，`_name` 设为非空值。PDF 第 2--3 页记录的序列化 Base64 长度为 5372，仍低于 6000 上限。

### 内存马与回显

第 4--6 页的 translet 静态块从当前 Spring/Tomcat 请求获取 `request`、`response`，解码请求参数 `c` 的 class bytes，以反射 `ClassLoader.defineClass` 加载 Filter。Filter 注册到 `/*`，当请求带 `cmd` header 时以 `ProcessBuilder(cmd.split("\\s+"))` 执行，并把标准输出置于 `result` 响应头。第 7 页的视觉与文本层一致：先对 `/deserialize?data=<Base64>` POST，再以 `cmd: env` 请求触发 Filter，响应头存在 `OK: OK` 与 `result`。

这类黑名单没有阻止已序列化对象图中现有的 CC 类；也不能因为没有 `ChainedTransformer` 就认为 `InvokerTransformer` + `TemplatesImpl` 不可达。

### 验证

PDF 逐页核验结果：第 1--6 页为代码与调用链文本，已转写；第 7 页的两张抓包截图可由文本层确认请求路径、`cmd` header 与 `result` 回显，因此未保留截图。当前源范围未提供服务端源码、依赖版本或可运行 generator，故不声称在本地重新生成或验证该 payload。

## 方法总结

- 核心技巧：用 CC6 的 map 触发结构承接 `InvokerTransformer` 与 `TemplatesImpl`，将反序列化转为类加载和内存 Filter 注册。
- 识别信号：Java 对象反序列化、`resolveClass` 字符串黑名单、Commons Collections，以及对 `ChainedTransformer` 的单点封禁。
- 复用要点：拒绝不可信原生反序列化；类名黑名单不能替代类型允许列表，也不能防止已允许类的危险方法组合。
