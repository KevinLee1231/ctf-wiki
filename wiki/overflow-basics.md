---
type: family
tags: [pwn, family, overflow, stack, oob, canary]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/overflow-basics.md
  - ../raw/pwn/D3CTF2023-d3op-wp.md
updated: 2026-07-06
---

# Overflow Basics

## 作用边界

本页是用户态基础内存破坏 family，不是单一 technique。它负责把栈溢出、ret2win、canary 泄露/爆破、长度字段越界读、signed index、单字节返回地址覆盖和简单结构体指针覆盖分流到更具体的 exploit 页面。

如果当前题的主要障碍已经是 glibc heap 状态、FILE/FSOP、seccomp ORW、ret2dlresolve、内核对象或浏览器/JIT primitive，应尽快转到对应 technique，不要继续把所有 crash 都按基础溢出处理。

## 识别信号

- `gets`、`read`、`scanf("%s")`、`memcpy`、协议长度字段、CSV/解析器字段或菜单输入能覆盖固定长度缓冲区。
- crash 点靠近返回地址、函数指针、GOT、结构体指针、canary 或相邻全局变量。
- 保护组合仍是路线核心：Canary 决定先泄露还是爆破，NX 决定 shellcode/ROP，PIE 决定是否需要代码地址泄露。
- 题目给出的 primitive 还比较初级，尚未稳定成任意读写或堆布局控制。

## 最小证据

- 已记录 `file`、`checksec`、架构、PIE/Canary/NX/RELRO、libc 版本和一次可复现 crash。
- 已用 cyclic、反汇编或调试确认覆盖距离，而不是只凭输入长度猜测。
- 对 canary、PIE 或 libc 的绕过方式有证据：泄露输出、fork server 行为、格式串、OOB read、协议回显或可重复的部分覆盖。
- 若是 OOB/index/stride 类 bug，已量化元素大小、有效边界和目标对象相对位置。

## 首轮路由

| 证据形态 | 先做什么 | 下一跳 |
|---|---|---|
| 固定偏移覆盖 RIP，No Canary 或 canary 已泄露 | 用 cyclic/GDB 固定 offset，检查 16 字节栈对齐，选择 ret2win、ret2libc 或基础 ROP。 | [ret2csu-dynelf-and-shellcode.md](ret2csu-dynelf-and-shellcode.md)、[stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md) |
| 有 win/magic 参数函数 | 找 `pop rdi; ret`、必要的 alignment `ret`，确认 payload 未触发过滤。 | [ret2csu-dynelf-and-shellcode.md](ret2csu-dynelf-and-shellcode.md) |
| canary 前后可读或字符串函数能打印越界 | 先泄露 canary，再返回 main 或二阶段输入；不要在 canary 损坏后直接继续返回。 | [runtime-protection-and-tls-exploits.md](runtime-protection-and-tls-exploits.md) |
| fork server 每次子进程继承 canary | 逐字节爆破 canary，使用连接行为、timeout 或错误响应区分 crash。 | [runtime-protection-and-tls-exploits.md](runtime-protection-and-tls-exploits.md) |
| 协议 length、stride、parser record 造成越界读/写 | 先把越界范围和每次读写粒度画清楚，再决定泄露地址、写 GOT 或构造 ROP。 | [cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md) |
| signed index、负数数量、单字节 RIP/函数指针覆盖 | 确认类型转换和目标地址同页/相邻关系；优先找低熵 pivot。 | [interpreter-jit-canary-and-integer-exploits.md](interpreter-jit-canary-and-integer-exploits.md) |
| 结构体指针覆盖后可间接写 GOT/函数指针 | 先选不会被目标函数再次调用的 GOT 项，再把写 primitive 变成控制流。 | [format-string.md](format-string.md)、[heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md) |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [D3CTF2023-d3op-wp](../raw/pwn/D3CTF2023-d3op-wp.md) | ubus RPC 的 base64 服务输出长度计算和索引检查不一致，最终是协议入口触发的 AArch64 栈溢出；先量化溢出长度和保护，再转跨平台 ROP。 |

## 合并与拆分结论

- 保留为 `family`：raw 中的条目共享“基础覆盖或越界先量化 primitive”的首轮判断，但后续路线明显分叉。
- 不拆成 ret2win、canary brute force、stride OOB 等多个页面：这些模式目前在本库中多数作为相邻 technique 的入口信号存在，单独成页会重复 `runtime-protection`、`ret2csu`、`interpreter-jit` 的内容。
- 不并入 [pwn-first-pass-red-flags-and-protections.md](pwn-first-pass-red-flags-and-protections.md)：首轮页负责全 Pwn 保护与漏洞族选择，本页负责基础 overflow/OOB/canary 这一族的二级分流。

## 常见陷阱

- 把“能 crash”当成“能控 RIP”，没有确认 saved RIP、saved RBP、callee-saved register 或 canary 的相对位置。
- 看到 canary 就放弃栈题；很多 raw 案例通过 null-byte 泄露、stride OOB、fork brute force 或 scanf skip 继续利用。
- 只看输入过滤字符串，忽略 payload 中地址字节也可能触发 `memmem`/黑名单。
- 单字节覆盖只有在同页函数、低熵返回点或可重复调用时才稳定，不能默认替代完整 ROP。

## 关联技巧

- [pwn-first-pass-red-flags-and-protections.md](pwn-first-pass-red-flags-and-protections.md)
- [ret2csu-dynelf-and-shellcode.md](ret2csu-dynelf-and-shellcode.md)
- [stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md)
- [runtime-protection-and-tls-exploits.md](runtime-protection-and-tls-exploits.md)
- [format-string.md](format-string.md)
- [interpreter-jit-canary-and-integer-exploits.md](interpreter-jit-canary-and-integer-exploits.md)
- [heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md)
- [cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md)

## 原始资料

- [overflow-basics.md](../raw/pwn/overflow-basics.md)
- [D3CTF2023-d3op-wp](../raw/pwn/D3CTF2023-d3op-wp.md)
