# no-syscalls-allowed.c

## 题目简述

程序先把 `/flag.txt` 读入全局数组 `flag[100]`，再分配一页可写、可执行内存并接收最多 `0x1000` 字节 shellcode。随后加载默认动作为 `SCMP_ACT_KILL`、没有任何放行规则的 seccomp 过滤器，最后跳转到 shellcode：

```c
char flag[100];

int fd = open("/flag.txt", O_RDONLY);
read(fd, flag, sizeof flag);

void *code = mmap(NULL, 0x1000,
                  PROT_WRITE | PROT_EXEC,
                  MAP_ANONYMOUS | MAP_PRIVATE, -1, 0);
read(STDIN_FILENO, code, 0x1000);

seccomp_load(seccomp_init(SCMP_ACT_KILL));
((void (*)())code)();
```

执行 shellcode 时，flag 已经在内存里，但任何系统调用都会杀死进程。因此不能使用 `write`、`sendfile`、ORW 或 `execve`；可利用的输出只剩进程退出时间。

## 解题过程

### 建立一位时间预言机

仓库 healthcheck 用 `ret` 验证快速结束，用 `jmp $` 验证连接会长时间保持，这已经证明可以把“短时间退出”和“明显延迟”编码为 0、1。针对地址 `target` 的第 `bit` 位，可生成如下结构的 shellcode：

```asm
movzx eax, byte ptr [target]
shr eax, bit
and eax, 1
test eax, eax
jz fast

mov ecx, 0x20000000
slow:
dec ecx
jnz slow

fast:
ud2
```

`ud2` 通过非法指令使进程立即终止，不需要系统调用；慢分支只执行用户态循环。客户端记录发送 payload 到连接关闭的时间，超过校准阈值记为 1，否则记为 0。对同一位置从低位到高位查询八次即可还原一个字节。实际网络存在抖动，稳健脚本应先分别用恒快、恒慢 payload 校准阈值，并对临界结果重复测量、取多数票。

### 从运行时返回地址定位 PIE

每次建立连接都会重新触发 ASLR，不能在一次连接中泄露绝对地址、再把该地址用于下一次连接。解决方法是让每个 payload 都从当前进程的运行时状态重新计算目标：间接 `call code` 会把 `main` 中调用点之后的返回地址压到 shellcode 入口处的 `[rsp]`，因此它天然是当前进程的 PIE 指针。

```asm
mov rbx, [rsp]          ; 当前进程 main 内的返回地址
and rbx, -0x1000       ; 对齐到页
sub rbx, page_delta    ; 候选 ELF 基址
movzx eax, byte ptr [rbx + offset]
```

逐页向低地址测试候选页开头的四个字节，直到时间预言机还原出 `7f 45 4c 46`，便确定当前执行对应的 ELF 基址。这里被客户端保存的是相对页数和相对偏移；下一条连接的 payload 仍从自己的 `[rsp]` 重算基址，所以 ASLR 不会破坏跨连接泄露。

### 定位全局 flag 并逐字节泄露

官方 `SOLUTION.md` 给出的路线是先从 `main` 附近开始泄露机器码，再反汇编函数，寻找访问全局 `flag` 的 RIP-relative `lea` 指令。对形如：

```asm
lea rsi, [rip + disp32]
```

的指令，目标地址为“下一条指令地址加有符号位移 `disp32`”。将它换算成相对 ELF 基址的固定偏移后，后续每个 payload 都执行：

```text
当前 [rsp] 中的 PIE 锚点
  -> 当前连接的 ELF 基址
  -> ELF 基址 + flag 相对偏移 + 字节下标
  -> 读取指定位并编码为运行时间
```

公开的[完整复现记录](https://ctftime.org/writeup/34779)使用了等价方法：从 `rbp` 附近的栈内容找 PIE 指针，验证 ELF 文件头，再在基址约 `0x4000` 后的全局数据区搜索 `uiuc` 前缀，最后逐位恢复字符串。这里保留链接用于交叉核对；时间位预言机、PIE 相对寻址和 flag 定位过程均已在正文说明。

按字节恢复到 `\x00` 为止，得到：

```text
uiuctf{timing-is-everything}
```

## 方法总结

- 核心技巧：在所有系统调用均被禁止后，用纯用户态忙等与快速崩溃构造一位时间侧信道，再以运行时 PIE 指针为锚点读取内存。
- 识别信号：秘密已在地址空间中、攻击者可执行代码、直接输出被 seccomp 阻断，但进程存活时间仍能从网络侧观察。
- 复用要点：侧信道 payload 必须在每次新进程中重算地址，不能复用 ASLR 后的绝对指针；还应校准阈值、处理网络噪声，并先验证已知字节或 ELF magic，避免在错误基址上长时间盲泄露。
