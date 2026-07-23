# Advantageous Adventures 1

## 题目简述

题目是一个 Flask 计算器。服务端把用户提交的两个操作数直接交给 Python `eval()`，再计算结果；同一进程中还存在保存敏感网络信息的 `secret_info` 变量。

## 解题过程

源码中的核心逻辑等价于：

```python
value1 = eval(request.form["value1"])
value2 = eval(request.form["value2"])
result = value1 + value2
```

`eval` 接受任意 Python 表达式，而不是只接受数字。因此可在任一输入框直接引用全局变量，或通过函数全局命名空间访问它。最简单的输入是：

```text
secret_info
```

另一个输入使用能与返回类型兼容的空值，例如空字符串。响应会回显变量中保存的 IP、用户名、密码和 Wi-Fi 信息，其中包含：

```text
UMDCTF-{pl3z_n0_3v4l}
```

如果应用只回显算术结果，也可构造 `globals()["secret_info"]`；关键不是命令执行，而是 Python 表达式注入已经突破了计算器的数据边界。

## 方法总结

`eval()` 不能用来解析不可信的数字输入。即使过滤括号或系统命令，表达式仍可读取全局对象并调用可达方法。安全实现应使用 `int()`、`float()` 或受限语法解析器，并在业务层明确允许的运算符和类型。
