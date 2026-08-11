# cheap logs

## 题目简述

服务把 2184 字节随机数据直接作为 GMP 大整数 `priv` 的 limb 后备存储，并公布 ElGamal 风格公钥 $pub=2^{priv}\bmod p$。菜单可重复打印公钥，也可提交一个十六进制猜测；服务检查 $2^{guess}\bmod p=pub$。

私钥的初始化写法存在 use-after-free：`mpz_roinit_n(priv, buf, ...)` 让 GMP 直接引用 `malloc` 得到的 `buf`，但函数无条件执行 `free(buf)`。后续 GMP 及输入转换的堆分配会覆写同一块“私钥”存储。完整主线是先用 GMP/ptmalloc 堆布局把私钥重塑成“绝大部分已知”的代数对象，再对剩余 36 bit 未知量做结构化 MITM 离散对数；最终仍以恢复密钥通过群关系校验，故主分类保留为 Crypto，UAF 是必要的 Pwn 前置原语。

## 解题过程

### 关键观察

初次和后续的 `print_public_key` 都会调用 `mpz_powm_sec`。计算 `pub` 时所需的约 $0x88$ 字节 limb 分配可回收已释放私钥块的开头，因此下一次读取 `priv` 时，低 1088 bit 已变成上一轮公钥。官方 `solv_joseph.py` 还说明该空闲块紧随其后的元数据包括 unsorted-bin 的 `fd`/`bk`，二者是同一 libc 指针。

更关键的是菜单选项 3：`mpz_inp_str` 会先为输入字符串分配缓冲区，再为 `guess` 的 limb 分配空间。发送长度为 $0x370\times2$ 个十六进制 `f`，会得到大小分别为 $0x480$ 和 $0x380$ 的两段分配，恰好覆盖原来大小 $0x800$ 的私钥 chunk。于是覆写后的私钥由以下部分组成：

- 两次打印得到的公钥（其中一次是低位 `pub_prev`）；
- 固定的 chunk 元数据、攻击者输入的 `0f`/`ff` 字节和已知的 `guess` 计算结果；
- 同一个 libc 指针的两处副本；
- 字符串缓冲区中一个未被完全填满的随机字节。

### 将残余未知量化为小范围 DLP

令 $q=(p-1)/2$。按小端字节序拼出上述已知布局后，官方 `solve.py` 将私钥在模 $q$ 下写成

$$
priv=aL_1+bL_2+cB+d\pmod q,
$$

其中 $L_1$ 是 libc 地址熵的低 10 bit，$L_2$ 是接下来的 18 bit，$B$ 是那一个未知字节；其余 $a,b,c,d$ 都由已知 heap 布局计算。第二个公钥 $P=2^{priv}\bmod p$ 给出

$$
2^{bL_2}=P\cdot2^{-d-cB-aL_1}\pmod p.
$$

这把原本的大整数离散对数降为 $10+8+18=36$ bit 搜索。对每组 $(L_1,B)$ 建表，再枚举 $L_2$ 即可做 meet-in-the-middle：

```python
right = {}
for l1 in range(1 << 10):
    for byte in range(1 << 8):
        right[P * pow(2, -d - c * byte - a * l1, p) % p] = (l1, byte)

for l2 in range(1 << 18):
    candidate = pow(2, b * l2, p)
    if candidate in right:
        l1, byte = right[candidate]
        recovered = a * l1 + b * l2 + c * byte + d
        assert pow(2, recovered, p) == P
        break
```

实际脚本把多项式系数先落在 `Integers(q)`，并用 Sage 的有限域元素作字典键；这样指数符号与群阶的约化一致。最后将 `recovered` 以十六进制送入选项 3。

### 验证

官方 solver 在命中后断言 `pow(g, privkey_val, p) == pubkey2`，随后提交该值并读取服务输出。题目配置给出的验证材料为 `DUCTF{fr4gm3nt3d_d1scr3t3_fr4gm3nt5_l0g5_m4ss_h34p_m4554g1ng_gr00m1ng}`。本归档没有运行该约 $2^{18}$ 规模的求解循环，结论来自题目 C 源码与两份官方 solver 的静态对照。

## 方法总结

- 核心技巧：`mpz_roinit_n` 不取得输入 limb 的所有权；释放其 backing buffer 后，GMP 大整数会变成可由堆分配重塑的 use-after-free 数据。
- 识别信号：看到“大整数直接借用 malloc 内存”与随后 `free` 时，不能只按 crypto 题做 DLP；应把后续每一次 `mpz_*`、字符串解析和打印分配纳入 heap layout。
- 复用要点：先把覆盖后的整数按端序精确表达成少数未知变量，再用公开群关系把 ASLR/残余字节做 MITM。布局依赖题目随附的 glibc、GMP 和分配尺寸，不能照搬到不同运行时。
