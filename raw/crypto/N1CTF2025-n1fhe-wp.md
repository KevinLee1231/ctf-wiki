# N1CTF 2025 n1fhe

## 题目简述

题目使用 BFV 同态加密，环维数为 $4096$，明文模数由 `PlainModulus.Batching(4096, 18)` 生成。服务端把 16 字节 AES 密钥放到 4096 个槽位中的 16 个随机位置，其余槽位均为 0，再经 BatchEncoder 的逆 NTT 编码并加密。选手拿到这个密文后只有 32 次解密机会，而且每次只能看到解密多项式的第 0 个系数。

最终 flag 用同一密钥进行 AES-CTR 加密。问题因此分成两部分：用 32 个可选线性测量恢复一个汉明重量至多为 16 的稀疏槽位向量，再利用已知填充块恢复 PyCryptodome 自动生成但未显式给出的 CTR nonce。

## 解题过程

设槽位向量为 $e=(e_0,\ldots,e_{4095})$，编码后的明文多项式为

$$
M(x)=\sum_{i=0}^{4095}a_i x^i,
$$

则 BatchEncoder 的 NTT 关系可写成 $e_j=M(\omega_j)$。题目公开的是 $M$ 的 BFV 密文。BFV 允许把密文乘以已知明文多项式并相加，因此可以构造一个新密文，使其解密后的常数项等于任意指定的线性组合

$$
\langle h,e\rangle=\sum_j h_j e_j
=\sum_i a_i\left(\sum_j h_j\omega_j^i\right).
$$

官方脚本先生成 `key_enc * x^i`，再按 NTT 根 `omega` 计算组合系数。核心接口可概括为：输入一个长度为 4096 的向量 `h`，输出一个密文，其解密结果 `pt[0]` 就是 $\langle h,e\rangle$。这样，32 次解密不必逐槽旋转，而可以取得 32 个精心选择的校验和。

取有限域 $\mathbb F_p$ 上参数为 $(n,k)=(4096,4064)$ 的 Reed-Solomon 码，其校验矩阵 $H$ 有 32 行。依次查询 $H$ 的每一行，得到

$$
s=eH^T.
$$

在 Sage 中先求一个满足相同校验和的向量 $b'$，再把它解码到最近码字 $b$：

```python
C = codes.ReedSolomonCode(GF(p), 4096, 4064)
H = C.parity_check_matrix()

syndrome = []
for row in H.rows():
    ct = linear_cipher(row.change_ring(ZZ))
    syndrome.append(decrypt_constant_term(ct))

b_prime = H.T.solve_left(vector(syndrome))
b = C.decode_to_code(b_prime)
e = list(b - b_prime)
```

因为 $(b'-e)H^T=0$，所以 $b'-e$ 是码字；该码最小距离为 $33$，唯一译码半径正好是 16，可以恢复重量为 16 的误差向量。根据符号约定，恢复结果可能整体取负，官方脚本通过检查元素是否落在字节范围外来选择 `e` 或 `-e`。按槽位顺序收集非零元素即可还原 AES 密钥：

```python
if max(e) > 256:
    e = [-x % p for x in e]
key = bytes(x for x in e if x)
assert len(key) == 16
```

这里存在一个小概率边界：随机密钥字节可能等于 0，此时它与空槽不可区分，官方脚本的长度断言会失败；重新连接取得新实例即可。

flag 长度被固定为 48 字节，`pad(FLAG, 16)` 因而会额外添加一个完整的 `0x10` 填充块。默认 `AES.MODE_CTR` 使用 8 字节随机 nonce，最后一个计数器块满足

$$
\operatorname{AES}_K(\text{nonce}\parallel\text{counter})
=C_{\text{last}}\oplus(\texttt{10}^{16}).
$$

用 AES-ECB 逆运算即可得到整个 16 字节计数器输入，前 8 字节就是 nonce：

```python
from Crypto.Cipher import AES
from Crypto.Util.strxor import strxor

last_keystream = strxor(ciphertext[-16:], b'\x10' * 16)
counter_block = AES.new(key, AES.MODE_ECB).decrypt(last_keystream)
nonce = counter_block[:8]
plaintext = AES.new(key, AES.MODE_CTR, nonce=nonce).decrypt(ciphertext)
```

去掉 PKCS#7 填充后即可得到 flag。

## 方法总结

本题的关键不是逐槽解密，而是把同态运算视为“设计测量矩阵”的能力。32 次查询恰好对应一个冗余度为 32 的 Reed-Solomon 码，稀疏密钥向量则对应可唯一纠正的 16 个错误。恢复密钥后，还要注意 PyCryptodome 的 CTR nonce 虽未打印，却能由固定长度 flag 产生的已知完整填充块反推出。完整利用链为：构造 NTT 线性测量、收集 RS syndrome、唯一译码恢复稀疏槽位、由已知填充块恢复 nonce、CTR 解密。
