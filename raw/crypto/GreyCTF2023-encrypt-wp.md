# GreyCTF2023 Encrypt

## 题目简述

题目把 60 字节 flag 平分为整数 $a$、$b$，并在同一个随机密钥 $k$ 下分别计算

$c(x,k)=px+qx^2+(x+p+q)k$。

已知 $p$、$q$、$c(a,k)$ 与 $c(b,k)$。决定性弱点不是某个参数太小，而是两份密文复用了同一个线性出现的 $k$，可以先消元，再利用两段明文都只有 30 字节这一界限做小根恢复。

## 解题过程

在 Sage 中建立

$f(a,k)=pa+qa^2+(a+p+q)k-c_1$，

$g(b,k)=pb+qb^2+(b+p+q)k-c_2$。

对 $k$ 求结式即可得到只含 $a,b$ 的多项式 $h(a,b)=0$：

```sage
var('a b k')
f = p*a + q*a^2 + (a+p+q)*k - c1
g = p*b + q*b^2 + (b+p+q)*k - c2
h = f.resultant(g, k).expand()
```

两段明文满足 $0<a,b<2^{240}$。官方解法把 $h$ 中的 $a^2b$、$a^2$、$a$、$b$ 与常数项按对应上界缩放，构造整数格后执行 LLL；最短向量中的目标坐标直接给出 $a$、$b$。恢复结果时按大端序拼接两段：

```python
flag = long_to_bytes(abs(a)) + long_to_bytes(abs(b))
```

得到：

```text
grey{shortest_crypto_challenge_in_this_ctf_srfrGRUEShP8FKwn}
```

## 方法总结

看到多个密文共享同一未知量时，应先检查该未知量在方程中的次数。这里 $k$ 只线性出现，结式能无损消去它；消元后剩下的明文长度约束，正好把问题转化为低维格上的小根恢复。LLL 并非直接“解密”，而是利用已知字节长度从多项式关系中筛出唯一短解。
