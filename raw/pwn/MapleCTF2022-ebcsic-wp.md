# EBCSIC

## 题目简述

服务只接受 ASCII 大写字母和数字，随后不是按 ASCII 执行，而是用 IBM EBCDIC code page 037 解码为原始字节，再把这些字节放入一个 32 位 RWX ELF 映像运行。任务因此变成：只使用那些能由大写字母或数字映射得到的 x86 操作码，构造最终 shellcode。

## 解题过程

先枚举允许字符经 `cp037` 编码后的字节，并用反汇编器筛选可用指令。虽然直接访存和常见立即数受限，但集合中包含移位、循环移位、按位取反、`enter`、`leave` 与 `ret`，足以建立一个自修改载荷。

构造任意 32 位常量时，先用两次大移位清零寄存器，再 `not` 得到全 1；按照目标值的连续位段交替使用 `shl` 和 `rol`，每次只选可编码的移位立即数。官方生成器把这一过程封装为 `set_reg(reg, value)`。

随后从后向前处理普通 32 位 `/bin/sh` shellcode：把 `esp` 设到 RWX 地址 `0x8057000+i`，把四字节目标值放入 `ebp`，利用 `enter 0xc1c8,0xc1; leave` 的栈写副作用落盘。全部写完后把栈迁移到载荷地址并执行 `ret`：

```python
for offset in reversed(range(0, 32, 4)):
    set_reg("esp", 0x8057000 + offset + 4)
    set_reg("ebp", u32(shellcode[offset:offset + 4]))
    emit("enter 0xc1c8, 0xc1")
    emit("leave")
```

生成的机器码再以 `cp037` 反解为只含大写字母和数字的输入。执行后取得：

```text
maple{can't_4ddr3ss_mem0ry?_n0_pr0bl3m_with_EBCDIC_07321a5c}
```

## 方法总结

字母数字 shellcode 题的核心不是寻找“一条神奇指令”，而是评估受限指令集是否仍具备构造常量、写内存和转移控制流三种能力。本题多了一层 EBCDIC 映射，必须在过滤字符域和实际执行字节域之间转换；最终输入应重新编码校验，确保每个字符都在白名单且解码结果与生成机器码完全一致。
