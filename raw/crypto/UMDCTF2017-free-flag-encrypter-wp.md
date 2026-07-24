# Free Flag Encrypter

## 题目简述

压缩包中的 Python 2 工具实现了一个自定义背包密码。`gen_seq` 生成超递增私钥序列 $w$，再选取模数 $q$ 和乘数 $r$，公开权重为：

$$
a_i = r w_i \bmod q
$$

加密时把消息转换成比特 $x_i$，只输出一次子集和：

$$
C = \sum_i a_i x_i
$$

附件只留下 `umdctf.pub` 和密文，没有私钥。公开序列共有 290 项，密文规模约 391 bit；这类低密度子集和可以用格规约恢复。

## 解题过程

先去掉公钥文件的装甲头尾，依次 Base64 解码、zlib 解压和 JSON 解析。flag 的前缀 `UMDCTF-{` 与末尾 `}` 已知，可以从密文中扣除对应权重，只让 LLL 求中间未知比特。

对未知权重 $a_0,\ldots,a_{m-1}$ 和剩余目标 $S$，构造维数 $m+1$ 的整数格：

$$
B_i=(0,\ldots,2,\ldots,0,2Na_i)
$$

以及：

$$
B_m=(1,1,\ldots,1,2NS)
$$

其中 $N=\lceil\sqrt{m}/2\rceil$。若短向量最后一维为 0，且前 $m$ 维均为 $\pm1$，就可通过 $(v_i+1)/2$ 还原 0/1 选择向量。核心 Sage 代码如下：

```python
m = len(weights)
scale = ceil(sqrt(m) / 2)
basis = Matrix(ZZ, m + 1, m + 1)

for i, weight in enumerate(weights):
    basis[i, i] = 2
    basis[i, m] = 2 * scale * weight

for i in range(m):
    basis[m, i] = 1
basis[m, m] = 2 * scale * residual

for row in basis.LLL(delta=0.99).rows():
    if row[m] == 0 and all(abs(value) == 1 for value in row[:m]):
        bits = [(value + 1) // 2 for value in row[:m]]
        if sum(a * bit for a, bit in zip(weights, bits)) == residual:
            print(bits)
```

枚举合理的消息字节长度，把已知首尾比特补回并按每 8 bit 转成字节，得到：

```text
UMDCTF-{BetT3r_sticK_To_RsA_I_gueSs}
```

其 SHA-256 与 README 中的 `f0a7a7ee8c0bdcd10ed7fcf9a3a9c97ab611bcad6d6a21beb27e3ba1e762f6c2` 完全一致。

## 方法总结

Merkle–Hellman 的私钥超递增性质并不会自动让公开背包安全。这里的公开权重数量不大、数值尺度相对较小，子集和密度足以让 LLL 找到选择向量。利用已知 flag 格式先降维、再用子集和等式和 SHA-256 双重验证，可避免把偶然短向量误当成解。
