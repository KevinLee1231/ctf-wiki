# DownUnderCTF 2020 - Addition

## 题目简述

题目实现了一个 Flask 在线计算器，POST 参数 `user_input` 会被 Python `eval()` 求值。应用试图用关键词黑名单阻止 `import`、`open`、`globals`、`exec` 等危险表达式，但校验循环的控制流写错：每遇到一个不匹配的黑名单项就立即执行一次 `eval()`，命中项也不会 `break` 或阻止后续求值。flag 作为模块级变量保存在同一全局命名空间中。

## 解题过程

关键代码可以收敛为：

```python
blacklist = ["import", "os", "sys", ";", "print", "__import__",
             "SECRET", "KEY", "app", "open", "globals", "proc",
             "self", "read", "exec"]

maybe_this_maybe_not = "DUCTF{...}"

for item in blacklist:
    if item in user_input:
        out = "Nice try"
    else:
        out = eval(user_input)
```

正确写法应先完成全部检查，确认没有命中后再求值；这里却把危险操作放在循环的 `else` 中。以 `globals()` 为例，循环走到 `globals` 时虽然会设置拒绝文本，但下一个不匹配的 `proc` 又会执行表达式，最后一次 `exec` 也不匹配，因此响应最终仍是求值结果。

向计算器提交：

```python
globals()
```

返回的字典包含模块级变量，其中：

```text
'maybe_this_maybe_not': 'DUCTF{3v4L_1s_D4ng3r0u5}'
```

知道变量名后也可以直接提交：

```python
maybe_this_maybe_not
```

它完全不含黑名单词，同样直接返回 flag：

```text
DUCTF{3v4L_1s_D4ng3r0u5}
```

## 方法总结

- 核心技巧：利用用户输入进入 `eval()`，并识别黑名单循环没有形成真正的拒绝门。
- 识别信号：计算器接受 Python 表达式、服务端使用 `eval`、过滤逻辑在遍历每个关键词时分别执行危险操作，都是代码注入的强信号。
- 复用要点：安全修复不是增加黑名单，而是使用整数/运算符白名单解析器；即使必须校验，也应先计算 `any(item in input for item in blacklist)`，命中立即拒绝，绝不能在逐项检查的 `else` 中执行输入。
