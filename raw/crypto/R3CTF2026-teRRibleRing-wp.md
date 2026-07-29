# teRRibleRing

## 题目简述

题目对 flag 的每一位生成一组 512 次多项式样本。模数为

$$
p=\mathtt{0x8000000b}=2147483659,
$$

工作环为

$$
R=\mathbb F_p[x]/(f),
$$

其中 $f$ 是附件中给出的 512 次首一多项式。若当前 flag 位为 0，则输出

$$
a\leftarrow R,\qquad
s,e\leftarrow D_{\sigma=5}^{512},\qquad
b=as+e;
$$

若当前位为 1，则 $a,b$ 都从 $R$ 中均匀选取。附件一共给出 344 组样本，需要逐组判断其属于 RLWE 分布还是均匀分布，再每 8 位还原一个字节。

题名特意提示存在三个字母 “R”，真正的弱点也确实是 ring：多项式 $f$ 被构造成高度可约，并且它的若干因子还能组合成一个系数极小、极稀疏的 127 次因子。将样本投影到这个因子上，原本 512 维的格问题会降到 127 维，而且小误差在投影后仍然保持足够短。

## 解题过程

**寻找隐藏的稀疏因子**

直接在 $\mathbb F_p[x]$ 中分解 $f$，可以得到 75 个互异不可约因子，次数最高仅为 31。单独使用这些因子并不好，因为它们的系数看起来随机，取模时会把小秘密和小误差放大。

目标应是找若干因子的乘积，使其在整数中心提升后具有很小的系数。对一个候选不可约因子 $g$，构造由

$$
g,xg,x^2g,\ldots
$$

的系数向量以及 $pI$ 生成的格。格中的短向量对应一个模 $p$ 意义下能被 $g$ 整除、同时整数系数很小的多项式。下面是寻找短倍数的核心代码：

```python
def find_short_multiple(g, max_degree=128):
    coeffs = [ZZ(value) for value in g.list()]
    rows = []

    for shift in range(max_degree - len(coeffs) + 1):
        row = (
            [0] * shift
            + coeffs
            + [0] * (max_degree - len(coeffs) - shift)
        )
        rows.append(row)

    basis = block_matrix(
        ZZ,
        [
            [matrix(ZZ, rows)],
            [p * identity_matrix(ZZ, max_degree)],
        ],
    )
    reduced = basis.LLL()
    return RZ(list(reduced[0]))
```

对次数大于 12 的因子尝试该过程，会恢复出

$$
r(x)=x^{127}-x^{46}+x^{19}-x^8+1.
$$

在原始附件上可以直接验证

$$
f(x)\bmod r(x)=0.
$$

本地实际检查得到 `r_degree = 127`、`f_mod_r_is_zero = True`，商多项式次数为 385。这个极稀疏的 $r$ 才是题目埋入的后门；仅仅报告“$f$ 可约”仍不足以完成区分。

**把样本投影到 127 维**

对每组 $(a,b)$ 计算

$$
\bar a=a\bmod r,\qquad
\bar b=b\bmod r.
$$

若它是 RLWE 样本，则存在投影后仍较短的 $\bar s,\bar e$，满足

$$
\bar b=\bar a\bar s+\bar e\pmod{(p,r)}.
$$

令 $A$ 为“乘以 $\bar a$”在 $1,x,\ldots,x^{126}$ 基下的矩阵。也就是说，$A$ 的第 $i$ 行来自

$$
\bar a x^i\bmod r
$$

的中心提升系数。考虑行格

$$
B=
\begin{pmatrix}
I & A\\
0 & pI
\end{pmatrix}.
$$

其中的向量可以写成 $(u,uA+pv)$。以

$$
t=(0,\bar b)
$$

为目标做 CVP 时，RLWE 样本对应的距离向量本质上就是 $(-\bar s,\bar e)$，明显短于随机样本。使用 Kannan 嵌入，把 CVP 转成一次 SVP：

$$
B'=
\begin{pmatrix}
I&A&0\\
0&pI&0\\
0&\bar b&M
\end{pmatrix},
\qquad M=512.
$$

对 $255\times255$ 的 $B'$ 做强格基约化后，若最短行中绝大多数坐标的绝对值都小于 1024，就判为 0；否则判为 1。普通 LLL 或 `flatter` 强度不够，公开求解使用 BLASter 的 block size 42，每个样本约数秒。

下面给出整理后的完整求解框架。它从可信的本地附件提取 `p`、`f` 和样本，避免在 WP 中重复四百多行多项式系数；`BLASTER_DIR` 需要改成 BLASter 源码目录：

```python
#!/usr/bin/env sage -python
import ast
import re
import subprocess
import sys
from pathlib import Path

from Crypto.Util.number import long_to_bytes
from sage.all import *
from sage.repl.preparse import preparse


BLASTER_DIR = "/path/to/BLASter"

# 只执行 task.sage 中定义 p、PR、x、f 的前缀，不执行样本生成部分。
task_prefix = (
    Path("task.sage")
    .read_text(encoding="utf-8")
    .split("def F0():", 1)[0]
    .replace("from secret import flag", "flag = bytes()")
)
exec(preparse(task_prefix), globals())

sample_text = Path("samples.txt").read_text(encoding="utf-8")
output = ast.literal_eval(sample_text.split("=", 1)[1])

RZ.<y> = PolynomialRing(ZZ)
r_int = y^127 - y^46 + y^19 - y^8 + 1
r = PR(r_int)
d = r.degree()
assert f % r == 0


def centered_coefficients(poly):
    values = [ZZ(value.lift_centered()) for value in poly.list()]
    values += [0] * (d - len(values))
    return vector(ZZ, values)


def multiplication_matrix(poly):
    rows = [
        centered_coefficients((poly * x^i) % r)
        for i in range(d)
    ]
    return matrix(ZZ, rows)


def reduce_with_blaster(basis):
    payload = "[[" + "]\n[".join(
        " ".join(str(value) for value in row)
        for row in basis
    ) + "]]"

    command = [
        sys.executable,
        "src/app.py",
        "-b42",
        "-t1",
        "-P2",
    ]
    raw = subprocess.check_output(
        command,
        input=payload.encode(),
        cwd=BLASTER_DIR,
    )
    values = [int(value) for value in re.findall(rb"-?\d+", raw)]
    return matrix(ZZ, basis.nrows(), basis.ncols(), values)


def distinguish(sample):
    a_values, b_values = sample
    ar = PR(a_values) % r
    br = PR(b_values) % r
    A = multiplication_matrix(ar)

    base = block_matrix(
        ZZ,
        [
            [identity_matrix(ZZ, d), A],
            [zero_matrix(ZZ, d, d), p * identity_matrix(ZZ, d)],
        ],
    )

    target = matrix(
        ZZ,
        1,
        2 * d,
        [0] * d + list(centered_coefficients(br)),
    )
    embedded = block_matrix(
        ZZ,
        [
            [base, zero_matrix(ZZ, 2 * d, 1)],
            [target, matrix(ZZ, 1, 1, [512])],
        ],
    )

    shortest = reduce_with_blaster(embedded)[0]
    small_coordinates = sum(abs(value) < 1024 for value in shortest)
    return 0 if small_coordinates > 200 else 1


bits = [distinguish(sample) for sample in output]
value = int("".join(str(bit) for bit in bits), 2)
print(long_to_bytes(value))
```

344 次约化结束后，按高位在前的顺序拼接比特，恢复：

```text
R3CTF{h0W_To_d3t3cT_Vu1n3r4bi1ity_0F_RING?}
```

附件 README 中给出的 flag 长度为 43 字节，恰好与 $43\times8=344$ 组样本相符。上述后门因子的整除关系已在本地 Sage 环境复验；完整 344 次强约化属于约二十分钟量级的计算，没有在整理阶段重复执行。公开解法的原始实现与运行参数可参阅 [Sceleri 的 R3CTF 2026 题解](https://blog.sceleri.cc/posts/r3ctf-2026-writeup/)，其关键推导和依赖已经完整归纳在正文中。

## 方法总结

这道题不能停留在“分解 $f$”这一步。随机低次因子的系数很大，直接投影会破坏小误差结构；真正要找的是若干模 $p$ 因子的短整数倍数。利用移位倍数与 $pI$ 组成的格，可以反向恢复埋入的稀疏因子

$$
x^{127}-x^{46}+x^{19}-x^8+1.
$$

随后将 RLWE 样本降到 127 维，通过乘法矩阵、CVP 和 Kannan 嵌入判断是否存在短的秘密—误差对。设计环上密码参数时，模多项式不仅要满足次数要求，还必须检查不可约性、低系数因子以及从原始系数分布到商环各投影的几何行为；否则高维参数可能被隐藏因子直接降维。
