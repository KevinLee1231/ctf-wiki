# fakeobj.py

## 题目简述

服务先泄漏一个 Python 字典对象 `obj` 的地址和 libc 中 `system` 的地址，然后将用户提供的 72 个字节逐字节覆写到 `id(obj)` 指向的 CPython 对象头，最后执行 `print(obj)`。这不是普通 Python 语义漏洞：攻击者可以伪造 `PyObject`/`PyTypeObject` 相关字段，控制对象被打印时的 C 层回调。

容器把 flag 放在当前工作目录，并通过 nsjail 执行解释器脚本；ASLR 不影响本题，因为服务直接给出了对象地址和 `system` 地址。

## 解题过程

### `print()` 经过 `tp_repr`

CPython 打印对象会沿对象的 `ob_type` 找到类型槽，并调用该类型的 `tp_repr`。在 64 位 CPython 对象头中，前 8 字节是 `ob_refcnt`，接着 8 字节是 `ob_type`；`PyTypeObject.tp_repr` 位于类型对象起始处的固定偏移。于是可以让 `obj` 的 `ob_type` 指回攻击者覆写的数据区域，并让伪类型的 `tp_repr` 槽等于 `system`。

`tp_repr` 的第一个参数原本是对象指针；官方 solver 将 `ob_refcnt` 字段填成 `.bin/sh\0`，打印前解释器会递增引用计数，首字节 `.` 随之变为 `/`，于是传给 `system` 的字节串恰好是 `/bin/sh\0`。

```python
payload = flat(
    [
        b".bin/sh\x00",             # print 前递增后成为 /bin/sh\0
        p64(addrof_obj - 88 + 16),   # 让 tp_repr 槽落在 obj+16
        p64(system),                 # 假类型的 tp_repr
    ],
    length=72,
    filler=b"\x00",
)
```

`addrof_obj - 88 + 16` 的作用是让 CPython 取 `ob_type + 88` 时读取到 `obj + 16` 的 `system` 指针。之后服务的 `print(obj)` 触发 `system("/bin/sh")`，获得交互 shell。

### 验证

官方 solver 在发送上述 payload 后进入交互模式；验证动作是从 shell 读取工作目录中的 flag。本文没有执行服务或 solver，验证链来自其源码中给出的 payload、地址计算和最终 `print(obj)` 调用。

## 方法总结

- 核心技巧：对可任意覆写的 CPython 对象头，伪造 `ob_type` 和 `tp_repr`，把对象打印转化为一次受控函数调用。
- 识别信号：题目泄漏 `id()` 与 libc 函数地址，并在原生 `ctypes` 写入后触发 `print`/`repr`，应检查 CPython 的类型槽而不是只分析 Python 代码。
- 复用要点：需要同时满足调用约定和引用计数副作用；本题用 `.bin/sh` 而非 `/bin/sh`，正是为了抵消打印前的 `ob_refcnt` 自增。
