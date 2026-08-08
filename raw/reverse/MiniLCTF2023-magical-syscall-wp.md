# MiniLCTF2023 - magical_syscall

## 题目简述

程序在初始化数组中布置了 `/proc/self/status` 的 `TracerPid` 检查和 10 秒 `alarm`。主逻辑 `fork` 出父子进程：子进程调用 `PTRACE_TRACEME` 后不断执行由数组提供编号和参数的 `syscall`；父进程以 `PTRACE_SYSCALL` 拦截这些并把 `3901..3913` 当成自定义 VM opcode。

目标是恢复 VM 指令语义，识别其中的魔改 RC4，再按相同错误实现反推 38 字节输入。

## 解题过程

父进程通过 `PTRACE_GETREGS` 读取 `orig_rax`、`rdi`、`rsi`、`rdx`，把它们分别解释成 opcode 和三个操作数；执行完后再用 `PTRACE_POKEDATA` 修改子进程的 VM 状态。`pc` 每次按指令长度增加，是识别字节码边界的首要锚点。

对照成对出现的处理分支可以恢复主要指令：

```text
3901 ADD       3902 MOD       3903 GETCHAR
3904 PUSH      3905 POP       3906 CMP
3907 JE        3908 JNE       3909 INC
3910 XOR       3911 MOV       3912 XCHG
3913 RESET     8888 FAIL      9999 SUCCESS
```

反汇编字节码后，程序先读入 38 字节，初始化 256 字节 S 盒，再执行 RC4 KSA/PRGA 并逐字节比较密文。密钥是 `MiniLCTF2023`。关键缺陷位于 `XCHG`：实现没有临时变量，实际效果是

```python
s[i] = s[j]
s[j] = s[i]
```

而不是交换两个元素。因此必须复刻这个“假交换”，使用标准 RC4 库会得到完全错误的结果。官方密文和恢复脚本如下：

```python
def broken_ksa(state, key):
    j = 0
    for i in range(256):
        j = (j + state[i] + key[i % len(key)]) % 256
        state[i] = state[j]
        state[j] = state[i]


def broken_prga(state, length):
    i = j = 0
    stream = []
    for _ in range(length):
        i = (i + 1) % 256
        j = (j + state[i]) % 256
        state[i] = state[j]
        state[j] = state[i]
        stream.append(state[(state[i] + state[j]) % 256])
    return stream


key = list(b"MiniLCTF2023")
enc = [
    147, 163, 203, 201, 214, 211, 240, 213, 177, 26,
    84, 155, 80, 203, 176, 178, 235, 15, 178, 141,
    47, 230, 21, 203, 181, 61, 215, 156, 197, 129,
    63, 145, 144, 241, 155, 171, 47, 242,
]

state = list(range(256))
broken_ksa(state, key)
stream = broken_prga(state, len(enc))
plain = bytes(a ^ b for a, b in zip(enc, stream))[::-1]
print(plain)
```

输出并与仓库 `flag.txt` 一致：

```text
a_v1rtu@l_m@ch1ne_w1th_ma9ical_sy$call
```

按赛事 flag 格式写为 `miniLCTF{a_v1rtu@l_m@ch1ne_w1th_ma9ical_sy$call}`。

## 方法总结

ptrace 可以把系统调用边界改造成 VM 调度器：子进程负责触发事件，父进程充当解释器。恢复此类 VM 时先找 `pc`、比较标志和对称指令，再重建高级算法。即使认出 RC4，也必须逐行核对实现；一个缺少临时变量的交换就足以把标准算法变成完全不同的状态机。
