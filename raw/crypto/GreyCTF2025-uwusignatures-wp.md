# Uwusignatures

## 题目简述

题目实现了 ElGamal 风格签名。服务每个会话只生成一次随机 nonce $k$，并对最多两条自选消息返回签名中的 $s$，不返回 $r$；公钥参数 $p,g,y$ 公开。由于两次签名复用相同 $k$，对应的 $r=g^k\bmod p$ 也相同，可以消去私钥项并恢复 $k$、$r$ 与私钥 $x$，再为目标消息伪造完整签名。

## 解题过程

签名方程为：

$$
s_i \equiv (h_i-xr)k^{-1}\pmod{p-1}.
$$

对两条不同消息作差，$xr$ 项消失：

$$
s_1-s_2\equiv(h_1-h_2)k^{-1}\pmod{p-1}.
$$

取消息整数 1 和 3 请求两次签名。只要相关差值在模 $p-1$ 下可逆，就能依次计算：

$$
k^{-1}\equiv(s_1-s_2)(h_1-h_2)^{-1}\pmod{p-1},
$$

$$
r=g^k\bmod p,
$$

$$
x\equiv(h_1-s_1k)r^{-1}\pmod{p-1}.
$$

核心求解代码为：

```python
modulus = p - 1
kinv = ((s1 - s2) * pow((h1 - h2) % modulus, -1, modulus)) % modulus
k = pow(kinv, -1, modulus)
r = pow(g, k, p)
x = ((h1 - s1 * k) * pow(r, -1, modulus)) % modulus

target = bytes_to_long(b"gib flag pls uwu")
target_hash = hash_m(target)
target_s = ((target_hash - x * r) * pow(k, -1, modulus)) % modulus
```

提交 JSON `{"m": target, "r": r, "s": target_s}` 后，验证式成立，服务返回：

```text
grey{h_h_H_h0wd_y0u_Do_tH4T_OMO}
```

若消息哈希差、$r$ 或恢复出的 $k^{-1}$ 不可逆，当前随机参数无法完成这些除法，重新建立会话即可。

## 方法总结

- 核心技巧：对复用 nonce 的两条 ElGamal/Schnorr 类签名作差，消去固定私钥项并恢复 nonce。
- 识别信号：同一会话只生成一次 `k`，随后对不同消息重复签名，是致命的 nonce reuse。
- 复用要点：所有线性关系都在模 $p-1$ 下成立；每一步求逆前必须检查最大公因数，不可逆不代表公式错误。
