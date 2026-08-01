# GlacierCTF2022 - ChaCha60

## 题目简述

附件把一个 64 位主密钥视为 $\mathbb F_2^{64}$ 上的列向量，分别乘以三个 $32\times64$ 二进制矩阵，得到三段 32 位子密钥。每段子密钥补 28 个零字节后作为 ChaCha20 密钥，以全零 nonce 连续加密同一张 PNG。题目给出三个矩阵和密文，目标是恢复图片中的 flag。

## 解题过程

把三个矩阵纵向拼接：

$$
M=\begin{bmatrix}M_1\\M_2\\M_3\end{bmatrix}\in\mathbb F_2^{96\times64},
\qquad K'=MK.
$$

单独看每个 $M_i$ 都满秩，但附件中的组合矩阵只有 32 秩。这意味着 64 位主密钥虽然有 $2^{64}$ 种，真正可能出现的 96 位展开密钥却只有 $2^{32}$ 种。用 Sage 对列空间做消元，可取得一个 $96\times32$ 的基矩阵：

```sage
with np.load("matrices.npz") as f:
    m1 = matrix(GF(2), f["m1"])
    m2 = matrix(GF(2), f["m2"])
    m3 = matrix(GF(2), f["m3"])

m = block_matrix(3, 1, [m1, m2, m3])
assert m.rank() == 32
basis = m.T.echelon_form()[:m.rank()].T
```

随后枚举 $v\in\mathbb F_2^{32}$，计算 `basis * v`，将 96 位结果拆成三个 4 字节子密钥。ChaCha20 是流密码，三层加密仍可用已知明文快速筛选；只需解密密文前 8 字节并检查 PNG magic `89 50 4e 47 0d 0a 1a 0a`。官方实现把枚举循环编译成 Cython，试运行约 9 分钟，最坏约 45 分钟。

找到候选后，将每段 4 字节值补零到 32 字节，按相反顺序应用三个全零 nonce 的 ChaCha20 实例，即可恢复完整 PNG。图片中的文本为：

```text
glacierctf{n3x7_7im3_mak3_sur3_7o_us3_ind3p3nd3n7_ma7ric3s}
```

恢复图本身只是带有上述可复制文字的梗图，不承担额外结构证据，因此正文直接转写 flag，不保留图片副本。

## 方法总结

密钥长度不能代替有效密钥空间。多个线性派生输出若彼此相关，应联合计算秩；本题三个看似独立的 32 位子密钥合起来仍只有 32 位熵。已知文件头又提供了廉价且可靠的候选过滤器，使 $2^{32}$ 的枚举可以实际完成。
