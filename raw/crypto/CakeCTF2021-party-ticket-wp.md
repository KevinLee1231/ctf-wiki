# CakeCTF2021 Party Ticket

## 题目简述

每张邀请函都会生成新的 1024 位模数 $n=pq$ 和随机数 $b$，但消息始终相同：

$$
m=\operatorname{bytes\_to\_long}(\text{flag}\parallel\operatorname{SHA512}(\text{flag})).
$$

密文为

$$
c=m(m+b)\bmod n.
$$

于是 $m$ 是模 $n$ 上首一二次多项式 $f(x)=x^2+bx-c$ 的根。消息虽然带有 64 字节摘要，却仍显著小于模数；两张票据还提供了两个互素模数下的同一个小根，可以通过 CRT 合并后再使用 Coppersmith 方法。

## 解题过程

### 为两张票据建立多项式

从 `output.txt` 读出 $(c_1,n_1,b_1)$ 和 $(c_2,n_2,b_2)$，分别建立

$$
f_i(x)=x(x+b_i)-c_i\pmod {n_i}.
$$

取 CRT 幂等元 $k_1,k_2$，使它们分别满足

$$
k_1\equiv(1,0),\quad k_2\equiv(0,1)\pmod{(n_1,n_2)}.
$$

则

$$
f(x)=k_1f_1(x)+k_2f_2(x)\pmod N,\qquad N=n_1n_2
$$

在两个模数下都以真实消息 $m$ 为根。把它归一化为首一多项式后，可在更大的模数 $N$ 上寻找小根。

### 使用 Coppersmith 恢复消息

仓库双票据解法的核心如下：

```sage
import ast

with open("distfiles/output.txt") as f:
    c1, n1, b1 = ast.literal_eval(f.readline())
    c2, n2, b2 = ast.literal_eval(f.readline())

N = n1 * n2
PR.<x> = PolynomialRing(Zmod(N))

k1 = crt([1, 0], [n1, n2])
k2 = crt([0, 1], [n1, n2])
f1 = x * (x + b1) - c1
f2 = x * (x + b2) - c2
f = (k1 * f1 + k2 * f2).monic()

X = ceil(0.5 * N^RR(1 / 2 - 0.05))
for root in f.small_roots(X=X, beta=1, epsilon=0.05):
    raw = int(root).to_bytes((int(root).bit_length() + 7) // 8, "big")
    print(raw)
```

输出由 flag 和 64 字节 SHA-512 摘要拼接。分离末尾 64 字节并重新计算摘要，即可排除伪根并确认：

```text
CakeCTF{710_i5_5m4r73r_7h4n_R4bin_4nd_H4574d}
```

由于本题实际消息长度已经低于单个 $n^{1/2}$，仓库另一份官方脚本只用第一张票据也能调用 `small_roots()`；合并两张票据会扩大可接受的小根界限，成功条件更宽松。

## 方法总结

- 非线性表达式 $m(m+b)$ 仍然给出了关于未知消息的低次首一多项式，不能因为它不是标准 RSA 就忽略小根攻击。
- 同一小消息在多个互素模数下出现时，可以用 CRT 把多个同余根条件合为一个更大模数上的条件。
- 附带摘要没有增加消息的未知熵；它只扩大了消息长度，并在恢复后充当完整性校验。
