# super party computation

## 题目简述

服务模拟两方 ECDSA：Alice 提交自己的 P-256 公钥，Bob 生成私钥 $x_B$ 和 Paillier 加密的 $x_B$；共享 ECDSA 公钥来自 ECDH 点 $x_Ax_BG$。签名时，Alice 可提交 Paillier 加密的 partial signature，Bob 解密、以自己的 nonce share 相除、验证签名，并把“签名有效/无效”作为响应返回。

原协议应通过零知识证明等机制保证 partial signature 格式正确。这里完全缺失这一步，使签名是否 abort 成为关于 Bob 私钥的逐位 oracle；目标消息只禁止走签名接口，却可在 `get_flag` 中提交一份已验证签名。

## 解题过程

先提交私钥 $x_A$ 为奇数的 Alice 公钥，取得 Bob 公钥、Paillier 公钥 $(n,g=n+1)$ 与 $\operatorname{Enc}(x_B)$。第 $l$ 轮令 Alice nonce 为：

$$
k_A=2^l.
$$

从 Bob 返回的 nonce 公钥得到 $R=k_Ak_BG$ 及 $r=x(R)$。维护已恢复的低 $l-1$ 位 $y_B$，构造候选 $y'_B=y_B+2^{l-1}$，并以 Paillier 同态乘法形成加密 partial signature：

$$
c_3=\operatorname{Enc}(x_B)^{x_A r\,2^{-l}\bmod n}
       \cdot g^{\zeta+y'_B r(k_A^{-1}-2^{-l})x_A}\pmod {n^2},
$$

其中 $\zeta=k_A^{-1}m\pmod q$，$m$ 为测试消息哈希，$q$ 是 P-256 阶。官方 solver 在 Paillier 指数域使用 $2^{-l}\bmod n$，在 ECDSA 标量域使用 $k_A^{-1}\bmod q$；若 $r$ 为偶数则加 $q$，保证对 $2^l$ 可逆而不改变其模 $q$ 的含义。

Bob 解密后会仅在 $x_B\equiv y'_B\pmod {2^l}$ 时验签成功。因此 `sign_and_validate` 返回签名就置入该 bit，返回 `invalid signature parameters` 就保留为 0。循环 $l=1\ldots255$ 恢复 $x_B$；最高位可用已知 Bob 公钥比对确定。然后共享私钥为：

$$
x=x_Ax_B\pmod q.
$$

用任意自选 ECDSA nonce 为被拒绝的目标消息离线签名，再调用 `get_flag` 即可。`solve/solve.py` 与 `solve/solve-joseph.py` 分别给出同一 abort-oracle 思路的实现。服务返回：

```text
DUCTF{d0nt_w0rry_th3_3mus_w1ll_b3_0kay}
```

## 方法总结

门限签名的安全性不能由 Paillier 的语义安全单独保证。攻击者若可提交任意同态密文并观察验证结果，协议 abort 本身就可能泄露私钥位。应当对参与方输入做零知识范围/正确性证明，统一且不可区分地处理失败路径，并避免把有效性结果返回给恶意参与方。
