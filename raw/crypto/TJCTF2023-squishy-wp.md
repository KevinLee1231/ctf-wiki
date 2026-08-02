# squishy

## 题目简述

服务实现教科书式 RSA 签名 $\sigma(m)=m^d\bmod n$。注册任意未占用用户名时会返回签名，而登录 `admin` 需要提交有效签名。服务只禁止直接注册字节串 `admin`，没有对签名消息做哈希或填充，因此保留了 RSA 的乘法同态。

## 解题过程

令 $M$ 为 `admin` 的整数表示，选择任意与 $n$ 互素的小整数 $f$。分别注册以下两个字节串：

$$
M_1=Mf\bmod n,\qquad M_2=f^{-1}\bmod n.
$$

它们都不等于 `admin`，服务会返回 $\sigma_1=M_1^d$ 与 $\sigma_2=M_2^d$。相乘即得：

$$
\sigma_1\sigma_2\equiv(Mf)^d(f^{-1})^d\equiv M^d\pmod n.
$$

```python
from Crypto.Util.number import bytes_to_long, long_to_bytes

admin = bytes_to_long(b"admin")
f = 2
while True:
    try:
        inv = pow(f, -1, n)
        sig1 = sign_oracle(long_to_bytes(admin * f % n))
        sig2 = sign_oracle(long_to_bytes(inv))
        admin_sig = sig1 * sig2 % n
        if pow(admin_sig, 65537, n) == admin:
            break
    except ValueError:
        pass
    f += 1
```

用 `name=admin` 和合成签名登录，服务输出：

```text
tjctf{sQuIsHy-SqUiShY-beansbeansbeans!!!!!!}
```

## 方法总结

- 裸 RSA 签名具有乘法可塑性，禁止签某个精确消息不能阻止攻击者由其他签名合成它。
- 安全签名应先按标准方案编码消息，例如 RSA-PSS，而不是直接对消息整数做私钥幂。
- 构造因子时要保证其在 $\mathbb Z_n$ 中可逆，并用公钥运算验证合成签名后再提交。
