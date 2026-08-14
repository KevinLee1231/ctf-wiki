# CakeCTF2021 Matrix Cipher

## 题目简述

题目把 flag 字节组成整数向量 $m$，用公开整数矩阵 $B$ 加密：

$$
c=mB+e,
$$

其中噪声向量 $e$ 的每个分量都只在 $[-50,49]$ 内。密钥生成先构造接近正交的好基 $R$，再左乘随机幺模矩阵把它变为 Hadamard 比很差的公开基 $B$。这正是格上的近似最近向量问题：$mB$ 是格点，$c$ 只是它附近的一个点。

## 解题过程

### 用 LLL 找回较好的格基

$B$ 与 $R$ 只差幺模变换，生成的是同一个格。虽然不知道原始好基 $R$，但可以对公开基执行 LLL：

```sage
B, c = open("distfiles/output.txt").read().strip().split("\n")
B = Matrix(ZZ, ast.literal_eval(B))
c = vector(ZZ, ast.literal_eval(c))
R = B.LLL()
```

LLL 不保证还原生成时完全相同的 $R$，但会给出足够短且较正交的基，使 Babai 最近平面算法能够处理这里很小的误差。

### Babai 近似求解 CVP

先对约化基做 Gram-Schmidt。然后从最后一个基向量向前，把目标向量在正交向量上的投影系数四舍五入，并逐项扣除：

```sage
def babai_rounding(w, basis):
    orthogonal, _ = basis.gram_schmidt()
    residual = w
    for i in reversed(range(len(basis))):
        coeff = round(
            orthogonal[i].dot_product(residual)
            / orthogonal[i].norm()^2
        )
        residual -= coeff * basis[i]
    return w - residual
```

返回值就是距离 $c$ 很近的格点 $v=mB$。由于公开矩阵可逆，随后直接计算

$$
m=vB^{-1}
$$

并把整数分量转回字符：

```sage
nearest = babai_rounding(c, R)
m = nearest * B.inverse()
print("".join(chr(Integer(x)) for x in m))
```

得到：

```text
CakeCTF{ju57_d0_LLL_th3n_s01v3_CVP_wi7h_babai}
```

## 方法总结

- 幺模变换只改变格基，不改变格本身；公开坏基仍可通过 LLL 获得更适合计算的约化基。
- 当密文满足“格点加小噪声”时，问题应优先建模为 CVP，而不是直接求矩阵逆并忽略误差。
- 本题的完整链条是 `LLL 约化 -> Babai 最近平面 -> 乘 B^{-1}`。
