# wintertime 2

## 题目简述

题目实现了一个受 POKÉ 高维同源加密启发的方案。参数取：

$$
A=2^{127},\qquad B=3^{162},\qquad C=5^{18}
$$

并在满足 $p=4fABC-1$ 的超奇异椭圆曲线上构造乘积同源。私钥包含同源度数 `deg` 和用于掩码辅助点的标量 $\delta$。公开材料同时给出未掩码提示点 $(X_A,Y_A)$、公钥中的 $(\delta X_A,\delta Y_A)$，以及加密过程前后的两组 $2$ 幂扭点。目标是利用 Weil pairing 的双线性消去掩码，恢复 `deg` 与 $\delta$，再重建接收方共享点。

## 解题过程

先处理 $5^{18}$ 阶辅助点。公钥给出：

$$
X'_A=\delta X_A,\qquad Y'_A=\delta Y_A
$$

Weil pairing 满足 $e(aP,bQ)=e(P,Q)^{ab}$，因此：

$$
e(X'_A,Y'_A)=e(X_A,Y_A)^{\delta^2}
$$

在阶为 $C$ 的乘法子群中计算离散对数即可得到 $\delta^2\bmod C$：

```python
e0 = X_A.weil_pairing(Y_A, C)
e1 = dX_A.weil_pairing(dY_A, C)
delta_squared = discrete_log(e1, e0, ord=C, operation="*") % C
```

由于 $C=5^{18}$，从模 5 开始逐位提升平方根，每一层只需尝试 5 个新数字，即可恢复满足生成约束的 `rec_delta`。

第二处泄漏来自密文中的 $(P_{2,B},Q_{2,B})$ 与经过另一侧同源后的 $(P_{2,AB},Q_{2,AB})$。分别计算 $4A$ 阶 Weil pairing：

```python
e0 = P2_B.weil_pairing(Q2_B, 4 * A)
e1 = P2_AB.weil_pairing(Q2_AB, 4 * A)
deg_squared = -discrete_log(e1, e0, ord=4 * A, operation="*") % A
```

加密端对两点施加的随机缩放分别为 $\omega$ 与 $\omega^{-1}$，在 pairing 中相互抵消；剩余指数关系暴露 $-\mathrm{deg}^2\bmod A$。已知 `deg` 为满足约束的奇数，可从模 8 的根开始逐位 Hensel 提升到模 $2^{127}$。官方脚本保留两个与取值范围相容的候选，后续用同源构造本身验证。

对每个候选 `deg`，以公开点建立维数二乘积同源的核：

$$
\bigl(-\mathrm{deg}\,P_{2,B},P_{2,AB}\bigr),\qquad
\bigl(-\mathrm{deg}\,Q_{2,B},Q_{2,AB}\bigr)
$$

```python
kernel = (
    CouplePoint(-deg * P2_B, P2_AB),
    CouplePoint(-deg * Q2_B, Q2_AB),
)
Phi = EllipticProductIsogeny(kernel, 127)
```

把密文中的 $C$ 阶点 $X_B,Y_B$ 经该同源求像。`eval_dimtwo_isog` 通过一组 $C$ 阶基和 Weil pairing 解二维离散对数，修复只给 $x$ 坐标造成的符号歧义，并按 `A-deg` 调整像点。最后乘回已恢复的 $\delta$，得到与加密端一致的 $(X_{AB},Y_{AB})$。

生成端只取两点的 $x$ 坐标送入 SHAKE256：

```python
xof = shake_256(X_AB[0].to_bytes() + Y_AB[0].to_bytes())
plaintext = bytes(a ^ b for a, b in zip(ciphertext, xof.digest(len(ciphertext))))
```

错误的 `deg` 候选会在同源或点恢复阶段失败；正确候选输出：

```text
grey{th1s_1s_why_y0u_must_MASK_y0ur_valu3s!}
```

## 方法总结

本题并不是要求正面破解高维同源，而是利用辅助值掩码不完整导致的 pairing 泄漏。对两个点同时乘同一标量只会让 Weil pairing 的指数变成标量平方；互逆缩放则在 pairing 中直接消失。于是两个看似隐藏的私钥分量都退化成平滑阶子群中的离散对数与素数幂模平方根问题。恢复这些标量后，公开同源代码即可原样重建共享点并解密。
