# rw.py

## 题目简述

脚本维护列表 `a = [{}, (), [], "", 0.0]`，命令 `r idx` 会打印元素，`w idx byte` 却把 `idx` 直接加到 `id(a)` 后作为字节地址写入。`idx` 没有边界限制，因此服务提供了相对 `PyListObject` 基址的任意单字节写；读接口只能打印列表元素，需要先把它升级为泄漏和任意读。

官方 solver 针对题目给定的 CPython/nsjail 环境编写。对象、类型对象和 GOT 的偏移会随解释器构建而变化，不能把其中的具体常数照搬到其他 Python 版本。

## 解题过程

### 由 `repr` 泄漏列表基址和解释器基址

列表第 0 项是字典。官方解法把该字典对象的 `ob_type` 低字节改到一个 `tp_repr` 为空的类型页，使打印退化为形如 `<... object at 0x...>` 的地址表示。结合已知 `a` 到 `a->ob_item[0]` 的相对距离，便可回推出 `a` 的绝对地址。

接下来覆写 `a->ob_item[0]`，让它指向 `PyDict_Type->tp_dict`。该字典内的已加载方法对象在 `repr` 中再次泄漏一个解释器内地址，官方 solver 据此得到 Python 可执行文件基址。

### 伪造 `PyByteArrayObject` 做任意读

有了列表基址后，单字节写可逐字节写到任意绝对地址。官方 solver 在列表附近的可写区伪造 `PyByteArrayObject`，其 `ob_type` 指向真实 `PyByteArray_Type`，并可控 `ob_size`、`ob_bytes`、`ob_start`。令后二者等于目标地址，然后执行 `r 0`，Python 会把该地址的内容按 `bytearray(...)` 打印，形成任意读。

```python
write_abs(fake + 16, p64(size))  # ob_size
write_abs(fake + 32, p64(addr))  # ob_bytes
write_abs(fake + 40, p64(addr))  # ob_start
read(0)
```

利用该 primitive 读取 Python 二进制的 `nice` GOT 项，减去已知 libc 偏移得到 libc 基址。官方脚本中的候选距离修正分支是为部署中观察到的小范围布局偏差准备的，不改变漏洞本质。

### 伪对象调用 `system`

最后在可写区建立假 `PyTypeObject`，将其 `tp_repr` 设为 `libc_base + system`；再建立假对象，令其 `ob_type` 指向该假类型。把假对象首字段写为 `.bin/sh\0`，利用打印导致的引用计数加一，将参数修正为 `/bin/sh\0`。读取第 0 个列表元素时即触发 `system("/bin/sh")`。

### 验证

官方 `solv.py` 的验证顺序为：打印异常字典得到地址、打印类型字典得到解释器泄漏、通过伪 bytearray 读取 GOT 得到 libc 泄漏、最后 `read(0)` 取得 shell 并执行读取 flag 的命令。本文只审阅源码和官方 solver，未执行其中的利用。

## 方法总结

- 核心技巧：把以 `id(a)` 为基准的无界单字节写，逐级升级为地址泄漏、任意读和伪类型对象函数调用。
- 识别信号：Python 题若将 `ctypes` 与 `id()` 结合、且索引没有单位或范围校验，应按 CPython 内存对象而非普通语言级数组越界分析。
- 复用要点：先取得稳定基址再构造伪对象；`repr`/`print` 既可作为地址泄漏触发器，也可作为 `tp_repr` 劫持的最终调用点。
