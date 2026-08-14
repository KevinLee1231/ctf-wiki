# bi0sCTF 2024 - kowaiiVm

## 题目简述

题目实现一套自定义寄存器 VM，并在函数调用次数达到 10 且函数长度足够时将字节码 JIT 编译成 x86-64。解释器允许 `set` 指令修改 VM 映像中的字节，但 JIT 的安全假设建立在函数此前已被解释执行多次、字节码不会再变化之上。配合靠近映像末尾的函数和 PC 越界，可以让验证阶段与实际 JIT 解码看到不同的指令边界。

利用目标是让 JIT 编译器越过原函数的 `ret`，把后续不受限制的字节解释成指令；再利用 `mov` 立即数和 `push` 生成 ROP 链，在 seccomp 允许的 `open/read/write` 范围内读取 `flag.txt`。

## 解题过程

### 制造验证与编译的时间差

函数被调用时先由解释器运行。`retFunc()` 在 call count 达到 `JIT_CC = 0xa` 后调用 `jitGen()`，也就是说“已经安全执行过”并不代表即将编译的字节仍与之前相同。

官方 payload 在入口函数中重复调用位于 `0xbff0` 附近的函数。被调函数每轮执行 `get` 取出常量，再用 `set` 改写自身字节码。前几轮仍按原指令边界返回；达到阈值时，修改后的字节流才被 JIT 重新解码。

```text
entry:
    call target
    get  x1, mutation
    set  x1, 0
    ...重复到超过 JIT 阈值...

target @ 0xbff0:
    get  x1, mutation_source
    ...
    ret
    <原验证路径不可到达、但 JIT 会继续解码的字节>
```

函数贴近可写映像末端还使 PC 计算溢入相邻 RW 区。修改某条变长指令后，原先单独的 `ret` 字节会被当成前一指令的操作数，JIT 因而看不到预期终止点，继续编译其后的攻击者字节码。

### 用 JIT 输出拼出 ROP

JIT 后端会把 VM 寄存器操作直接编码为宿主 x86-64 指令。特别是 `mov reg, imm` 会把可控立即数原样写入 JIT 区，`push reg` 会把相应 64 位值压到宿主栈。官方 exploit 先用立即数字节在 JIT 区制造两个短 gadget：

- `pop rdi; pop rsi; pop rdx; ret`；
- `pop rax; syscall; ret`。

再从已知 JIT 映射关系计算这些 gadget 地址，按逆序发出 `push`，使栈上形成 ORW 链：

```text
open("flag.txt", O_RDONLY, 0)
read(fd, buffer, 0x40)
write(1, buffer, 0x40)
```

seccomp 明确允许 `open/openat/read/write`，但不允许任意进程执行，因此 ORW 比尝试 `execve` 更合适。payload 还把字符串 `flag.txt` 放在 VM 常量/BSS 区，并计算其运行时地址传给第一个 syscall。

### 触发宿主 `ret`

所有 ROP 值入栈后，JIT 生成的宿主 `ret` 从当前栈顶取出第一个 gadget 地址。此时控制流不再返回 VM 调度器，而是进入 ROP 链。最终写 syscall 将文件内容发送到标准输出。

复现时应分别检查三项：目标函数 call count 是否超过 10、修改后字节流的指令边界是否确实吞掉原 `ret`、JIT 区中立即数字节是否形成预期 gadget。仅观察解释器反汇编不足以确认最终 JIT 结果。

## 方法总结

漏洞本质是可变代码与 JIT 缓存假设冲突：解释执行历史被当作安全证明，但 `set` 允许在编译前自修改。通过改变变长指令边界，可以把终止字节变成操作数并暴露后续字节码。JIT 又把立即数和 push 映射成真实机器码与宿主栈操作，于是可构造受 seccomp 约束的 ORW ROP 链。
