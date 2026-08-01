# GlacierCTF2022 - CryptoShop

## 题目简述

商店初始余额为 5 欧元，购买商品后会返回一个“退款码”。程序生成 1024 位 RSA 密钥，并直接把价格 $p$ 的裸 RSA 私钥运算 $p^d\bmod n$ 当作退款码；验证时重新计算同一表达式。购买 flag 需要 1000 欧元，目标是利用已有低价商品的退款码伪造大额退款。

## 解题过程

登录后服务会公开 RSA 模数 $n$。依次购买价格为 2、3、5 的商品，并在每次购买后提交原退款码取回余额，即可得到：

$$
s_2=2^d\bmod n,\qquad s_3=3^d\bmod n,\qquad s_5=5^d\bmod n.
$$

裸 RSA 具有乘法同态性：

$$
s_a s_b\bmod n=(ab)^d\bmod n.
$$

因此只要目标金额能由 2、3、5 分解，就能用已知退款码相乘生成合法码。官方 solver 选择 $10^6=2^6\cdot5^6$：

```python
refund_amount = 1_000_000
forged = pow(refund_codes[2], 6, n)
forged = forged * pow(refund_codes[5], 6, n) % n
```

提交 `refund_amount=1000000` 与 `forged` 后，服务端计算出的参考值完全相同，于是余额增加一百万。代码中的 `prev_refunds` 集合从未写入，退款码重放也不会受到限制，不过伪造大额码本身已经足够。最后购买 `CTF-Flag`，得到：

```text
glacierctf{RsA_S1gnAtuRe_1ssu3}
```

## 方法总结

RSA 私钥运算不是可直接使用的签名方案。对业务整数做裸 RSA 会把乘法关系原样暴露给攻击者；应采用经过标准编码和哈希的签名方案，例如 RSA-PSS，并把交易编号、商品、金额、用户和一次性状态共同纳入签名与重放校验。
