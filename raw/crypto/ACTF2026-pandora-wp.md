# Pand√ra

## 题目简述

题目把 45 字节 token 藏进二次域理想和 binary quadratic form。附件 `chal.sage` 会生成两个 512-bit 素数 $p,q$，令绝对判别式满足 $\lvert\Delta\rvert=pq^2$，再把 token 拆成两个整数并构造二次域理想。解法需要先从十进制小数和二次型泄漏恢复判别式与 $A$，再把 NICE 类结构转成多元 Coppersmith 小根问题。

## 解题过程

附件中 token 被拆成：

```python
x = int(token[:30].hex(), 16)
y = int(token[30:].hex(), 16)
```

随后在 $K=\mathrm{QuadraticField}(-p)$ 的 maximal order 中构造：

```python
OK.ideal(x + y*w)
```

并取对应 binary quadratic form $[a,b,c]$。远端把 form 转到判别式 $-pq^2$ 对应的 order 后输出：

```text
z = A >> 360
B
lambda = sqrt(p*q^2) 的 470 位小数
```

其中 $A=a$，$B$ 是乘上 $q$ 后在 $2A$ 下取的中心代表。Sage reduction 没有把前两项完全洗掉，所以泄漏中仍能看到 $A$ 的高位和完整 $B$。

第一步是从小数恢复 $\lvert\Delta\rvert$。远端给出 $\sqrt{\lvert\Delta\rvert}$ 的 470 位小数部分，记为 $F/Q$，其中 $Q=10^{470}$。若整数部分为 $T$，则：

$$
\sqrt{\lvert\Delta\rvert}=T+\frac{F}{Q}+\varepsilon
$$

展开后：

$$
\lvert\Delta\rvert-T^2\approx 2T\frac{F}{Q}+\frac{F^2}{Q^2}
$$

因此可以在二维格生成的点 $mQ+nF$ 中，用 CVP 找到让第一维抵消 $\lfloor F^2/Q\rfloor$ 的最近点，第二维给出 $2T$。拿到 $T$ 后平方补回即可得到 $\lvert\Delta\rvert$。

第二步恢复完整 $A$。二元二次型满足：

$$
\Delta=B^2-4AC
$$

因此：

$$
A \mid \frac{B^2+\lvert\Delta\rvert}{4}
$$

远端泄露了：

$$
A=(z\ll360)+low,\qquad 0\le low<2^{360}
$$

这是“因子高位已知”的一元 Coppersmith 场景。对：

$$
M=\frac{B^2+\lvert\Delta\rvert}{4}
$$

构造 $x+known$ 的小根格，恢复 $low$，并只接受同时满足：

```text
M % A == 0
A >> 360 == z
```

的候选。

第三步利用 NICE 类结构恢复 token。完整 $A$ 出来后：

$$
C=\frac{B^2+\lvert\Delta\rvert}{4A}
$$

原始 token 诱导的短向量 $(U,V)$ 会让二次型表示 $q^2$：

$$
AU^2+BUV+CV^2=q^2
$$

因为 $q^2 \mid \lvert\Delta\rvert$，可以构造模 $\lvert\Delta\rvert$ 的二元小根问题：

$$
AU^2+BUV+CV^2\equiv0\pmod{\lvert\Delta\rvert}
$$

$U,V$ 的规模来自 token 两段长度，约 120 bit，远小于模数。求解时使用二元 Jochemsz-May/Coppersmith 流程：生成 shift 多项式、按根界缩放格列、LLL 约化、重建消失多项式，再用 gcd/resultant 提取候选根。

候选 $(U,V)$ 需要通过题目结构过滤：

```text
q2 = A*U^2 + B*U*V + C*V^2
q2 > 0
disc_abs % q2 == 0
q2 是完全平方
sqrt(q2) 和 disc_abs / q2 都是素数
p % 4 == 3
```

最后由理想到二次型的关系还原 token：

$$
2AU+BV=\pm s q x,\qquad V=\pm s y
$$

缩放因子 $s$ 只需试少量情况。拿到合法 $x,y$ 后按原长度拼回：

```python
token = x.to_bytes(30, "big") + y.to_bytes(15, "big")
```

运行环境为 `sage -python`，需要 Sage、fpylll 和格约化工具。最终得到：

```text
actf{c0o0oppperrrsm1th_1n_3very_where}
```

Sage/Python 脚本最终输出明文 flag，验证了格规约与后续解密参数恢复。

## 方法总结

- 核心技巧：利用二元二次型关系 $\Delta=B^2-4AC$ 和高位泄漏恢复结构参数，再把 NICE 表示关系转成二元 Coppersmith 小根问题。
- 识别信号：Crypto 题出现 quadratic field、ideal、binary quadratic form、NICE 或 $\Delta$ 时，应检查判别式分解和小根攻击。
- 复用要点：先从十进制小数恢复判别式，再恢复 $A$，最后求 $(U,V)$；每一步都要用二次型整除、平方和素性条件过滤候选。
