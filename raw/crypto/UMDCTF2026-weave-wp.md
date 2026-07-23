# weave

## 题目简述

题目在 $GF(2^{43})$ 上构造一个类似 Gabidulin 码的秩度量加密实例。参数为 $n=40$、$k=8$，原始错误的二进制秩不超过 5。公开输出中同时包含 Moore 生成矩阵所需的点、左右变换矩阵以及一个维数为 3 的二进制子空间。

密文可写为：

$$
\text{bolt}=\text{secret}\cdot\text{warp}+\text{frays},
$$

其中

$$
\text{warp}=\text{knot}\cdot\text{loom}\cdot\text{shuttle}^{-1}.
$$

陷门数据意外全部公开，使密文能够还原成标准 Gabidulin 解码问题。

## 解题过程

先在 Sage 中按题目给定的不可约多项式建立 $GF(2^{43})$，读取 `output.json` 中的全部域元素和矩阵。将密文右乘 `shuttle`：

$$
\text{bolt}\cdot\text{shuttle}=
(\text{secret}\cdot\text{knot})\cdot\text{loom}
+\text{frays}\cdot\text{shuttle}.
$$

`loom` 是由公开 pegs 生成的 $8\times40$ Moore 矩阵，所以第一项正是一个 $[40,8]$ Gabidulin 码字。`frays` 的二进制秩最多为 5，而 `shuttle` 的元素都落在公开的 3 维 $GF(2)$ 子空间中，因此变换后的错误秩至多为

$$
5\cdot3=15.
$$

该码的最小秩距离为

$$
d=n-k+1=33,
$$

唯一解码半径为 $\lfloor(d-1)/2\rfloor=16$，足以纠正秩 15 的错误。

用任意与题目域表示一致的 Gabidulin 解码器，对 `bolt * shuttle` 和 Moore 生成矩阵解码，得到消息

$$
u=\text{secret}\cdot\text{knot}.
$$

随后计算

$$
\text{secret}=u\cdot\text{knot}^{-1}.
$$

必须沿用生成器中的元素序列化顺序，将 `secret` 序列化后计算：

```python
key = sha256(serialized_secret).digest()[:16]
```

最后使用该密钥和输出中的 nonce、tag 对 `vault` 执行 AES-GCM 解密并校验 tag，得到：

```text
UMDCTF{l01dr34u_l4mbda3_brick5_th3_w34v3_but_th3_trapd00r_unsp00ls_1t}
```

## 方法总结

本题的突破口是先消去公开的坐标变换，再检查错误秩是否仍落在 Gabidulin 唯一解码半径内。公开“看似混淆”的陷门矩阵不仅没有增加安全性，反而把实例直接还原为标准码字加低秩错误。实现时最容易出错的是有限域基和序列化字节序，AES-GCM 的 tag 可作为最终一致性校验。
