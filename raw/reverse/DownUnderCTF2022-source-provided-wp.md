# DownUnderCTF 2022 source-provided Writeup

## 题目简述

题目同时给出 64 位 ELF 和仅 50 字节校验逻辑的 NASM 源码。程序从标准输入读取 50 字节，对第 $i$ 个字节做加法、异或和截断，再与 `.data` 中的常量数组比较。目标是逐字节逆运算得到正确输入。

## 解题过程

开头的系统调用看起来有些反常：

```nasm
xor rax, rax
xor rdi, rdi
mov rdx, 0x32
sub rsp, 0x32
mov rsp, rsi
syscall
```

在 `main` 入口处，`rsi` 原本保存 `argv`。`mov rsp, rsi` 把栈指针改到这块可写内存，同时 `read(0, rsi, 0x32)` 将输入覆盖到同一地址；后续循环从 `[rsp+r10]` 读取输入。程序最终直接调用 `exit` 系统调用，因此破坏正常栈帧不会触发返回崩溃。

对每个输入字节，汇编执行：

```nasm
movzx r11, byte [rsp + r10]
add r11, r10
add r11, 0x42
xor r11, 0x42
and r11, 0xff
cmp r11, byte [c + r10]
```

若输入为 $x_i$、常量为 $c_i$，则校验关系是
$c_i=((x_i+i+0x42)\mathbin{\mathrm{xor}}0x42)\bmod256$。
逆运算为
$x_i=((c_i\mathbin{\mathrm{xor}}0x42)-0x42-i)\bmod256$。

把源码中的 50 字节数组直接代入即可：

```python
c = bytes([
    0xc4, 0xda, 0xc5, 0xdb, 0xce, 0x80, 0xf8, 0x3e, 0x82, 0xe8,
    0xf7, 0x82, 0xef, 0xc0, 0xf3, 0x86, 0x89, 0xf0, 0xc7, 0xf9,
    0xf7, 0x92, 0xca, 0x8c, 0xfb, 0xfc, 0xff, 0x89, 0xff, 0x93,
    0xd1, 0xd7, 0x84, 0x80, 0x87, 0x9a, 0x9b, 0xd8, 0x97, 0x89,
    0x94, 0xa6, 0x89, 0x9d, 0xdd, 0x94, 0x9a, 0xa7, 0xf3, 0xb2,
])

flag = bytes(((value ^ 0x42) - 0x42 - i) % 256
             for i, value in enumerate(c))
print(flag.decode())
```

输出为：

```text
DUCTF{r3v_is_3asy_1f_y0u_can_r34d_ass3mbly_r1ght?}
```

## 方法总结

这道题考查寄存器语义和可逆字节运算。`mov rsp, rsi` 不是常见的缓冲区设置方式，但它使 `read` 的目标地址与后续 `[rsp+i]` 完全一致；不能仅凭“不像正常函数序言”就把它当成无效代码。校验中的加法、异或和模 $256$ 截断都可逐字节逆转，按相反顺序运算即可恢复 flag。
