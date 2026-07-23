# Shellcodia 1

## 题目简述

服务允许提交一段 x86-64 shellcode，并在执行后检查 `RAX` 是否等于 7。目标不是打开 shell，而是用最短的机器码满足寄存器后置条件并正常返回。

## 解题过程

汇编只需要给 `RAX` 赋值并执行 `ret`：

```asm
mov rax, 7
ret
```

用 pwntools 组装并发送：

```python
from pwn import *

context.arch = "amd64"
payload = asm("""
    mov rax, 7
    ret
""")

io = remote("host", 1234)
io.send(payload)
io.interactive()
```

检查通过后输出：

```text
UMDCTF{R_U_@_Tim3_tR@v3ll3r_OR_Ju$t_R3a11y_Sm@rT}
```

仓库 README 的哈希已过期，以上 flag 与公开二进制中嵌入的检查结果一致。

## 方法总结

Shellcode 题应先读清验收条件。本题不要求系统调用或代码执行持久化，只要求返回时的一个寄存器值；最小化指令既更稳定，也减少了字符限制和执行环境差异带来的问题。
