# bi0sCTF 2025 - ഓണസദ്യ 题解

## 题目简述

这是一道分三层校验的 Linux 逆向题。主程序先检查用户名前 12 字节，再借助 `SIGILL` 处理器释放并执行内嵌的 `payasam`；后者继续校验密码和用户名剩余 14 字节。源码还会临时解密函数、检查调试器并执行一套自定义 VM，但这些机制最终都可以还原为独立的小约束，不需要对整个输入做指数级爆破。

最终输入为：

```text
username = ShrlkCheetzul1kesT4llWom3n
password = l4d1eS-m4n-2i6-fr
```

仓库 `admin/README.md` 给出的官方 flag 是：

```text
bi0sctf{ShrlkCheetzul1kesT4llWom3n_l4d1eS-m4n-2i6-fr}
```

## 解题过程

先看 `part_3.c`。程序读取用户名和密码，用固定种子 `srand(1234)` 生成共享随机字节，并把用户名逐字节映射成 8 个计数值。`check()` 对前 12 个字符逐字节检查该映射，随后再计算这 12 字节的 SHA-256：

```c
unsigned char correct_hash[] = {
    0x92, 0x29, 0x6a, 0x48, 0xf0, 0xfd, 0xef, 0x24,
    0x17, 0x08, 0xbf, 0x2a, 0xa7, 0xe6, 0x2e, 0x38,
    0x42, 0x0d, 0x8c, 0x01, 0x47, 0xed, 0x6b, 0x18,
    0xde, 0x9e, 0x8a, 0xcc, 0xd8, 0x90, 0x50, 0x7d
};
```

因为每个字符的 8 个映射值只依赖该字符和对应的共享随机字节，所以每一位只需枚举 256 个候选，而不是枚举整段字符串。得到候选 `ShrlkCheetzu` 后，可以直接用最终哈希常量确认：

```python
import hashlib

prefix = b"ShrlkCheetzu"
assert len(prefix) == 12
assert hashlib.sha256(prefix).hexdigest() == (
    "92296a48f0fdef241708bf2aa7e62e38"
    "420d8c0147ed6b18de9e8accd890507d"
)
```

校验成功时，`check()` 执行 `ud2` 并紧跟字节 `0x02`。`SIGILL` 处理器据此进入第二分支：把内嵌 ELF 写成 `payasam`，然后将父进程 PID、原密码和 `username[12:]` 作为参数执行它。这个信号处理器不是装饰性的反调试代码，而是通往后两层检查的真实控制流。

在 `password_verfication.c` 中，密码先经过 ChaCha20，再与 17 字节 `final_flag` 比较。源码已经给出 32 字节 key、12 字节 nonce、counter 0 和密文，因此直接执行同一条流密码即可恢复明文：

```python
from Crypto.Cipher import ChaCha20

key = bytes([
    44, 155, 95, 3, 242, 195, 36, 209,
    223, 101, 236, 84, 13, 210, 66, 158,
    146, 86, 187, 95, 233, 24, 240, 191,
    40, 21, 215, 122, 71, 88, 6, 119,
])
nonce = bytes([64, 97, 242, 19, 37, 51, 56, 220, 2, 78, 39, 137])
ciphertext = bytes.fromhex("3e02682025a97ef0db4a85b51dbf1b79c2")

password = ChaCha20.new(key=key, nonce=nonce).decrypt(ciphertext)
assert password == b"l4d1eS-m4n-2i6-fr"
```

密码通过后，`setup()` 把用户名后 14 字节、`add_val`、`mul_val` 和目标数组装入 VM。把 VM 指令展开后，每一位都是下面的独立等式：

$$
(u_i + a_i)\times m_i=t_i
$$

所以有：

$$
u_i=\frac{t_i}{m_i}-a_i
$$

用源码中的三组常量计算即可得到后缀：

```python
target = [1048, 616, 714, 872, 630, 94, 455, 402, 1152, 560, 408, 798, 603, 1190]
add =    [23,   28,  12,   8,  11, 10,  13,  26,   20,  25,  25,  24,  16,    9]
mul =    [8,     8,   6,   8,   5,  1,   7,   3,    9,   5,   3,   6,   9,   10]

suffix = bytes(t // m - a for t, a, m in zip(target, add, mul))
assert suffix == b"l1kesT4llWom3n"
```

把 12 字节前缀和 14 字节后缀拼起来就是完整用户名。再按官方格式将用户名与密码用下划线连接，即得到 flag。

## 方法总结

这题的关键不是硬啃所有混淆，而是沿真实控制流把检查拆开：确定性逐字节映射和 SHA-256 负责用户名前缀，`SIGILL` 处理器释放第二阶段，ChaCha20 常量恢复密码，自定义 VM 最终化简为 14 个一次方程。本文对 SHA-256、ChaCha20 明文和 VM 方程结果都用当前源码常量做了本地复算；未把 Linux 附件整体跑一遍，因此运行时反调试与自修改代码只作静态源码确认。
