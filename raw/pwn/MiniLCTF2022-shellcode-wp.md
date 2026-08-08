# MiniLCTF2022 shellcode Writeup

## 题目简述

64 位程序把最多 `0x100` 字节输入映射为 RWX 后执行，但 seccomp 只允许系统调用号 `0`、`1`、`5`、`9`，在 x86-64 下分别是 `read`、`write`、`fstat`、`mmap`，缺少读取 flag 所需的 `open`。过滤器只比较系统调用号，没有检查体系结构；而 32 位 i386 的系统调用号 5 恰好是 `open`，因此可通过兼容模式切换复用白名单。

## 解题过程

第一阶段在 64 位模式调用 `mmap`，在低于 4 GB 的固定地址申请 RWX 页，例如 `0x40404000`，再用 `read` 写入第二阶段。低地址是必要条件：切到 32 位兼容模式后，`EIP/ESP` 只能寻址低 32 位。

构造远返回帧并用 `retfq` 切换：

```text
push 0x23              ; 32-bit user code selector
push stage32_address
retfq
```

程序会拒绝输入字节 `0xcb`，即 `retf/retfq` 的 opcode。绕过方式是在第一阶段运行时用异或或加减把 `0xcb` 写入低地址页，而不是把该字节直接放入原始输入。

32 位阶段把 `./flag` 压栈，执行：

```asm
xor ecx, ecx
mov ebx, esp
mov eax, 5
int 0x80              ; i386 open("./flag", O_RDONLY)
```

seccomp 看到的数字仍是 5，因而按白名单放行。拿到文件描述符后再构造 `CS=0x33` 的 `retf` 帧返回 64 位模式，调用允许的 `read(fd, buffer, 0x40)` 与 `write(1, buffer, 0x40)` 输出 flag。第二阶段布局可概括为：

```text
low RWX page:
  64-bit retfq stub -> 32-bit open -> retf -> 64-bit read/write
low stack:
  selectors, return addresses, "./flag\0"
```

当前源仓库未保留实际 `shellcode` ELF，只保留部署文件；禁用字节、seccomp BPF 与完整阶段结构由多份比赛脚本一致确认，具体低地址可按远程映射情况调整。

## 方法总结

seccomp 规则必须同时校验 `seccomp_data.arch` 和系统调用号。只看数字会把不同 ABI 中含义不同的调用混为一谈。本题还同时考查远返回模式切换、低地址代码与栈迁移，以及对禁用 opcode 的运行时自修改。调试时应分开验证三步：低地址映射成功、32 位 `open` 返回有效 fd、切回 64 位后 `read/write` 正常。
