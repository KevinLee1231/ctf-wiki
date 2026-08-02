# TSGCTF2024 piercing_misty_mountain

## 题目简述

目标是一个静态链接、无 PIE、无栈保护的 64 位 ELF。`main` 先把最多 0x1000 字节的名字读入大栈缓冲区，然后进入 `auth()`；`auth()` 又在栈上分配 0x4000 字节随机密码区。选择菜单项 3 后，`profile()` 读取职业和年龄：

```c
int profile() {
  char job[0x8] = "Job:";
  char age[0x8];
  read_n(job + 4, 0x18 - 4);
  read_n(age, 0x8);
  return atoi(age);
}
```

从 `job + 4` 开始最多写 0x14 字节，会越过 8 字节的 `job`，最终控制保存的 `rbp` 与返回地址，但可控空间不足以直接放置常规 ROP。利用重点是把这 16 字节覆盖、8 字节 age 和 `main` 中的大缓冲区串起来。

## 解题过程

### 1. 在 `main` 缓冲区预置 ROP

第一次 `Name` 输入有 0x1000 字节空间。先放置一段 `ret` 滑板，再放完整的 `execve` 链。静态二进制中 gadget 足够多，链先把 `/bin/sh` 分两次写入 `.bss`，再设置寄存器：

```text
rdi = .bss + 0x50    # "/bin/sh"
rsi = 0
rdx = 0
rax = 59             # execve
```

最后跳到 `syscall; ret`。

### 2. 利用 `atoi` 返回现场把栈迁到 `age`

`profile()` 最后执行 `atoi(age)`。在本题使用的 libc/编译结果中，`atoi` 返回时 `rcx` 保留着 `age` 的地址。职业输入覆盖保存返回地址为：

```text
mov rsp, rcx; pop rcx; jmp rcx
```

于是 `profile()` 返回后先令 `rsp = &age`，再从 age 的 8 字节内容取出下一 gadget 地址并跳转。也就是说，虽然 age 看起来只是数字输入区，它实际上成为栈迁移后的第一个 ROP 槽位。

### 3. 用 `ret 0x4910` 跨过随机区

`age` 中放入官方 solver 搜到的 gadget：

```text
pop rdx; add eax, 0x83480000; ret 0x4910
```

其中前两条指令的副作用不重要，关键是 `ret 0x4910`。它从当前小栈区取出一个合法的 `ret` 地址作为下一跳，同时一次性把 `rsp` 增加 0x4910，跨过 `auth()` 的 0x4000 字节随机缓冲区及其他局部变量，落入先前的 `main` 名字缓冲区。职业溢出中保存的 `rbp` 槽位被设置为普通 `ret`，用于衔接这次大跨度返回。

由于落点需要精确对齐，名字缓冲区前部填入一串相同的 `ret` 地址作为滑板；官方 solver 在第 286 个 8 字节槽附近开始放真正的写 `.bss` 与 `execve` 链。控制流最终沿滑板进入 ROP，获得 shell。

读取 flag：

```sh
cat flag*
```

结果为：

```text
TSGCTF{7h3r3_wa5_571ll_w0rk_70_b3_d0n3_b3y0nd_7h3_pa7h_0f_hard5h1p_bu7_w3_l00k_f0rward}
```

## 方法总结

本题展示了极短溢出下的分阶段栈迁移：利用函数返回时残留的 `rcx` 把 `rsp` 指向 8 字节 age，再利用带立即数的 `ret` 一步跨越超大栈帧，最终接上早已写入 `main` 的长 ROP。分析这类题时不能只按“可覆盖多少字节”判断是否可利用，还要检查调用约定留下的寄存器、父栈帧中的可控缓冲区，以及 `ret imm16` 等能大幅调整 `rsp` 的非典型 gadget。
