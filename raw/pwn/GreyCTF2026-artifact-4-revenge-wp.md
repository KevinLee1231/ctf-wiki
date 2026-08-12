# artifact-4-revenge

## 题目简述

服务实现一个只有 256 字节内存、8 个 8 位寄存器的自定义 VM。每条指令固定 4 字节，地址运算模 256；`r6` 充当栈指针，`r7` 是指令指针。程序把 32 字节 flag 放在 `MEM[0x90:0xb0]`，再运行一个“输入 32 字节、每字节减 `0x20`、与 flag 比较”的校验器。

`gets` 指令没有长度限制，只过滤出可打印 ASCII，因此可以覆盖 VM 栈上的保存帧。利用需要两次劫持 `scramble` 变换：第一次形成栈溢出并控制返回地址，第二次把预编码的可打印字节解码为 VM shellcode，再栈迁移到该 shellcode。决定性步骤是执行边界劫持，所以按 Pwn 归档。

## 解题过程

VM 中与利用相关的指令为：

```text
F1 x y z : r[x] = MEM[(r[y] + z) mod 256]
F3 x     : push r[x]
F4 x     : pop r[x]
F5 x y z : if r[x] != r[y], r7 += z
F6 x     : gets(MEM + r[x])
F7 x     : print r[x] and exit
F8 x     : call relative x
```

`main` 在栈上留出 0x20 字节，从 `rbp-0x20` 开始读入，却不限制输入长度。紧接着 `scramble(ptr,0x20)` 对前 32 字节逐个减 `0x20`。溢出的第 33、34 字节可覆盖保存的 `rbp` 和返回 IP。

第一次比较在首字节不等时进入失败路径，此时作为 flag 指针的寄存器仍指向 `0x90`。把保存返回地址改成 `vm.bin` 中 `bne rsi, zer, scramble_loop` 那条指令的位置，函数返回后就会重新进入 `scramble` 循环；把保存 `rbp` 改为 bytecode 中合适的可控 pivot，则第二次 `mov rsp,rbp; pop rbp; ret` 会把执行流迁移到输入缓冲区中的指令。

输入只能包含 `0x20..0x7e`，而 shellcode opcode 位于 `0xf0..0xf8`。目标 shellcode 所在部分会先后经历两次 `-0x20`，因此发送时对每个目标字节加 `0x40` 模 256。解码后的 8 字节 shellcode 为：

```text
F1 01 04 ii    ; r1 = MEM[r4 + ii]，r4 此时为 flag 基址 0x90
F7 01 00 00    ; 打印 r1 并退出
```

在编码 shellcode 后放一个空格。第一次 `scramble` 把空格从 `0x20` 变为 NUL；第二次 `scramble` 解码完前面的 shellcode后遇到该 NUL，避免继续破坏后方栈数据。

参考脚本先在 `vm.bin` 中定位分支字节序列：

```python
jmp = raw.index(b"\xf5\x04\x00\xe0")
```

再从现有 bytecode 中选择能让第二次函数尾声正确 pop/ret 的 pivot。对 flag 的 32 个偏移分别启动一次服务，构造对应 `ii`，解析 `F7` 打印的十进制字节并拼接：

```python
for i in range(32):
    value = run_one_payload(load_offset=i)
    flag.append(value)
```

最终得到：

```text
grey{vm_1s_r34lly_n0+_th4t_h4rd}
```

## 方法总结

这题先要求逆出 VM 指令和校验器栈布局，再利用无界 `gets` 完成两级控制流劫持。输入字符限制不是终点：既然合法程序会固定减 `0x20`，就可以把它当作自解码器；让 payload 经两次变换后恰好成为不可打印 opcode。每次只泄露一个字节虽然需要 32 次连接，却避免了在 8 位 VM 中构造复杂输出循环，稳定性更高。
