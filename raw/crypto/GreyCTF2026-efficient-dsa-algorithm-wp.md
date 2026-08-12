# Efficient DSA Algorithm

## 题目简述

题目实现了 secp256k1 上的 ECDSA，但临时随机数 $k$ 由 32 位二进制 LFSR 产生。公开记录先泄露了连续 96 位 LFSR 输出，签名函数紧接着取后续 256 位拼成 $k$。同时给出消息、签名 $(r,s)$、公钥和一段以私钥派生流加密的密文。

ECDSA 的安全性要求每个 nonce 不可预测。LFSR 虽能产生看似变化的比特流，却是 GF(2) 上的线性递推；泄露长度超过线性复杂度两倍时，可以恢复递推并预测后续输出。

## 解题过程

从 `output.txt` 读取曲线阶 $n$、消息、96 位 `lfsr_output`、签名和密文。对泄露比特执行 Berlekamp–Massey，得到能够生成该序列的最短连接多项式。若递推系数为 $c_1,\ldots,c_L$，后续比特满足：

$$
b_t=\bigoplus_{i=1}^{L}c_i b_{t-i}
$$

用恢复出的递推继续生成 256 位，按照生成器相同的高位在前顺序转成整数并模 $n$，即可得到签名 nonce：

```python
bits = [int(bit) for bit in out["lfsr_output"]]
recurrence = berlekamp_massey(bits)
nonce_bits = predict(bits, recurrence, 256)
k = int("".join(map(str, nonce_bits)), 2) % out["n"]
```

ECDSA 签名关系为：

$$
z=\operatorname{SHA256}(m)\bmod n
$$

$$
s=k^{-1}(z+rd)\bmod n
$$

此时 $k,r,s,z,n$ 均已知，直接移项恢复私钥：

$$
d=(sk-z)r^{-1}\bmod n
$$

```python
z = int.from_bytes(sha256(message).digest(), "big") % n
d = ((s * k - z) * pow(r, -1, n)) % n
```

生成端用私钥的 32 字节大端表示计算 SHA-256，并循环使用这 32 字节作为 XOR 密钥流。因此按相同方式派生密钥即可解密：

```python
key = sha256(d.to_bytes(32, "big")).digest()
plaintext = bytes(c ^ k for c, k in zip(ciphertext, cycle(key)))
```

得到：

```text
grey{LFSR_i5_n0t_4_C5PRNG_at_aL1!!!}
```

## 方法总结

本题把两个常见错误串成了完整攻击链：先从裸 LFSR 输出恢复线性递推并预测 ECDSA nonce，再由单个已知 nonce 的签名求出长期私钥。公钥在这里主要用于校验 $dG$ 是否一致，并不能弥补 nonce 生成器的缺陷。密码协议中的“临时秘密”与长期私钥同样重要，必须由合格的 CSPRNG 或确定性 ECDSA 方案产生。
