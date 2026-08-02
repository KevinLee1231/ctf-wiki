# N1CTF 2020 FlagBot Writeup

## 题目简述

题目给出 7 组不同椭圆曲线上的 ECDH 公钥与一段 AES-CBC 密文。发送方在所有曲线上复用了同一个私钥 `secret`。曲线生成器只保证群阶含有一个较大的素因子，却没有清除其余小因子，也没有强制公钥位于统一的大素数子群。攻击者可在每条曲线的小子群中求出 `secret` 的模值，再用 CRT 拼回完整私钥。

## 解题过程

### 分解每条曲线的群阶

对第 $i$ 条曲线计算生成点阶 $n_i$ 并分解。选择一个可在合理时间内求离散对数、同时能为 CRT 提供足够信息的因子 $r_i$。若原生成点为 $G_i$、公开点为 $P_i=secret\cdot G_i$，令：

$$
h_i=\frac{n_i}{r_i},\qquad G_i'=h_iG_i,\qquad P_i'=h_iP_i.
$$

则 $G_i'$ 的阶为 $r_i$，且：

$$
P_i'=(secret\bmod r_i)G_i'.
$$

附件 solver 会逐步剥去过大的素因子，使剩余阶通常小于约 $2^{40}$，再用 Sage 的 Pohlig–Hellman/`discrete_log` 求解。

### CRT 恢复复用私钥

对 7 条曲线得到：

$$
secret\equiv s_i\pmod{r_i}.
$$

用广义中国剩余定理合并；若模数不互素，要先确认同余在最大公因数上相容，并用最小公倍数更新总模数：

```sage
secret = s[0]
modulus = r[0]
for i in range(1, 7):
    secret = crt(secret, s[i], modulus, r[i])
    modulus = lcm(modulus, r[i])
```

当累计模数超过私钥取值上界，例如 $2^{255}$，合并结果就是唯一的原始 `secret`。把它乘到接收方公钥，必须能在每条曲线上复现给定共享点，这是恢复成功的交叉验证。

### 派生 AES 密钥并解密

题目从共享点横坐标派生 32 字节摘要：

$$
digest=\operatorname{SHA256}(x_{shared}).
$$

按源码约定取前 16 字节作为 AES key，后 16 字节作为 IV，执行 CBC 解密并移除 PKCS#7 padding：

```python
digest = sha256(encoded_x).digest()
key, iv = digest[:16], digest[16:]
plain = unpad(AES.new(key, AES.MODE_CBC, iv).decrypt(ciphertext), 16)
```

`encoded_x` 的十进制、十六进制或定长大端编码必须与 `bot.sage` 完全一致，否则私钥正确也会得到错误摘要。

## 方法总结

跨曲线复用标量会把每条曲线的小子群泄露累积起来。单条曲线保留一个大素因子并不足够：必须验证生成点和对端公钥都属于指定的大素数子群，并进行 cofactor clearing；更根本的做法是固定经过审计的标准曲线，避免在不同群中复用长期私钥。
