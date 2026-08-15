# Champions Heap

## 题目简述

题目用球队净胜球数组 `gd` 管理一块堆内存。程序启动时直接打印该堆块地址，二进制无 PIE、GOT 仅受 Partial RELRO 保护，但启用了栈 Canary 和 NX。沙箱只禁止 `execve` 与 `execveat`。

漏洞链由两个边界错误组成：`reset_teams()` 会把 `gd[9]` 清零，而 `gd` 只有 9 项；随机对手又可能取到闭区间 `[0, 9]` 中的下标 9。`gd[9]` 恰好覆盖相邻 top chunk 的 size 字段，因此可先将其清零，再通过一场必胜比赛令其发生无符号下溢，构造 House of Force。

## 解题过程

`gd` 的申请大小为 `9 * 8 = 0x48` 字节，对齐后对应 `0x50` 字节 chunk。程序把用户区地址作为 `Gift` 输出，而 top chunk 的关键元数据紧随其后。

重置循环的终止条件多执行一次：

```c
for (i = 0; i <= NUM_TEAMS; i++) {
    gd[i] = 0;
}
```

随机函数也把上界包含在内：

```c
opteam_idx = randint(0, NUM_TEAMS);
```

选择 1 号球队 Real Madrid 后，己方得分固定为 5，而对手只有 0 至 4 分。若随机下标恰好为 9，下面第二次更新实际作用于 top chunk size；由于对手净胜球为负，原本为零的字段会下溢成极大的无符号值：

```c
gd[myteam_idx] += myscore - opscore;
gd[opteam_idx] += opscore - myscore;
```

为了准确等待下标 9，先泄露随机种子。全局 `name[0x20]` 后紧邻 `seed`，而 `read()` 读满 32 字节时不会补 `NUL`；随后 `printf("Hi %s", name)` 会继续输出 seed 字节。使用题目附带的同版本 libc 调用 `srand(seed)` 和 `rand()`，即可在本地预测随机对手出现的时刻。

```python
from ctypes import CDLL

libc_funcs = CDLL("./libc.so.6")
libc_funcs.srand(seed)

def randint(low, high):
    return libc_funcs.rand() % (high - low + 1) + low
```

触发 top size 下溢后，按 House of Force 计算超大申请尺寸，使 top chunk 指针按 64 位地址回绕到固定的 GOT 区域。官方利用先以 `getchar@got` 为落点，随后从球队名输出中取得 `main_arena` 指针，计算 libc 基址。

落到 GOT 后写入的组合数据完成三件事：

1. 把 `malloc@got` 改为 libc 的 `gets`；
2. 把 `num_teams` 清零，使后续仍可添加球队；
3. 把一个 `teams` 指针改为 `libc.sym.environ`，通过“显示球队”泄露栈地址。

`malloc` 被替换成 `gets` 后，`add_team()` 中原本的 `malloc(size)` 等价于以 `size` 作为第一个参数调用 `gets`。因此把菜单中的“大小”填写为目标地址，就可以把下一行输入写到任意地址。由 `environ` 泄露值减去官方脚本确定的 240 字节偏移，得到 `main` 的栈返回地址，并把 ROP 链写到该处。

沙箱会杀死 `execve`，所以最终链不能直接 `system("/bin/sh")`。官方方案先执行：

```text
mprotect(page(name), 0x1000, 7)
read(0, name, 0x100)
jump name
```

第二阶段 shellcode 使用允许的 `open`、`read`、`write` 系统调用读取 `/challenge/flag.txt`，最终输出：

```text
shellmates{r3ally_cUR10U$_to_sEe_HOw_U_S0lv3D_1t_U_c4N_dm}
```

## 方法总结

本题把数组越界与堆分配器元数据连成完整利用链。堆地址泄露确定 top chunk，越界清零和随机越界更新制造超大 top size，House of Force 再把分配位置推进到 GOT；之后通过 GOT 劫持建立任意地址写，逐步泄露 libc 与栈地址，最后在 seccomp 限制下改用 ORW shellcode。

修复至少要把重置条件改为 `i < NUM_TEAMS`，并让随机下标上界使用 `NUM_TEAMS - 1` 或实际的 `num_teams - 1`。此外，姓名读取应保留一个终止字节，生产构建还应启用 PIE 和 Full RELRO。seccomp 只减少最终利用方式，不能弥补此前的内存破坏。
