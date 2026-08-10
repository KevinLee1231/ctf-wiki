# exRSA

## 题目简述

题目仍以 $n=pq$ 和 $c\equiv m^{65537}\pmod n$ 加密，但另外生成了三个特殊整数：

$$
e_i\equiv d_i^{-1}\pmod{\varphi(n)},
$$

其中 $d_1,d_2,d_3$ 都是约 768 位的素数，而 $n$ 约为 2048 位。换言之，每组参数都满足：

$$
e_i d_i-k_i\varphi(n)=1,
$$

且未知的 $d_i$ 相对 $n$ 很小。单个样本未必足以直接使用经典 Wiener 攻击，但三个共享模数和欧拉函数的关系可以通过扩展 Wiener 格同时利用，进而恢复 $\varphi(n)$。

## 解题过程

### 将三组小私钥关系放入同一个格

令 $a=768/2048=3/8$，表示小私钥指数相对于模数位数的比例。题目附件给出了完整的 `e1`、`e2`、`e3`、`c` 和 `n`；这些数均为数百位整数，正文不重复粘贴无助于理解的整段数字，运行时应从题目输出原样填入脚本。

以下 8 维格来自三组关系 $e_i d_i-k_i\varphi(n)=1$ 的乘积组合。对各列施加与未知量预期上界对应的缩放后，LLL 更容易找到包含小 $d_i$、$k_i$ 的关系向量：

```python
from Crypto.Util.number import inverse, long_to_bytes

# 从题目输出原样填入五个大整数。
e1 = ...
e2 = ...
e3 = ...
c = ...
n = ...

a = 768.0 / 2048.0
D = diagonal_matrix(ZZ, [
    n^1.5,
    n,
    n^(a + 1.5),
    n^0.5,
    n^(a + 1.5),
    n^(a + 1),
    n^(a + 1),
    1,
])

L = matrix(ZZ, [
    [1, -n, 0, n^2, 0, 0, 0, -n^3],
    [0, e1, -e1, -n*e1, -e1, 0, n*e1, n^2*e1],
    [0, 0, e2, -n*e2, 0, n*e2, 0, n^2*e2],
    [0, 0, 0, e1*e2, 0, -e1*e2, -e1*e2, -n*e1*e2],
    [0, 0, 0, 0, e3, -n*e3, -n*e3, n^2*e3],
    [0, 0, 0, 0, 0, e1*e3, 0, -n*e1*e3],
    [0, 0, 0, 0, 0, 0, e2*e3, -n*e2*e3],
    [0, 0, 0, 0, 0, 0, 0, e1*e2*e3],
]) * D

B = L.LLL()
relations = matrix(ZZ, B) * L^(-1)

# 官方解法使用首个约化向量中的比例恢复 phi。
phi = floor(e1 * relations[0, 1] / relations[0, 0])
assert phi > 0

d = inverse(65537, int(phi))
m = pow(c, d, n)
plaintext = long_to_bytes(m)
print(plaintext)
assert pow(m, 65537, n) == c
```

这里的核心不是把 `e1` 当作普通 RSA 的小公开指数，而是利用它存在一个模 $\varphi(n)$ 的小逆元。LLL 返回的关系矩阵中包含与 $\varphi(n)$ 成比例的系数，故可由首行对应项的比值恢复 $\varphi(n)$。得到欧拉函数后，再计算真正加密指数 $65537$ 的逆元并正常解密。

输出为：

```text
hgame{Ext3ndin9_W1en3r's_att@ck_1s_so0o0o_ea3y}
```

官方 PDF 展示了格基和恢复公式，但未打印最终 flag；[参赛者题解](https://www.cnblogs.com/mumuhhh/p/18032304)给出了同一扩展 Wiener 格与最终结果，正文已经概括其必要信息。

## 方法总结

- 看到 $e\equiv d^{-1}\pmod{\varphi(n)}$ 且 $d$ 很小时，应把它改写为 $ed-k\varphi(n)=1$，再判断 Wiener 或多变量扩展攻击是否适用。
- 多个共享 $n$、$\varphi(n)$ 的小私钥样本比单个样本泄露更多信息，可以把各关系及其乘积嵌入同一个格。
- 格中的对角缩放用于平衡不同单项式的数量级；若省略，LLL 往往只会找到数值上短、却与目标关系无关的向量。
- 恢复 $\varphi(n)$ 后仍应重新加密验证明文，以排除格约化行选择或取整造成的伪解。
