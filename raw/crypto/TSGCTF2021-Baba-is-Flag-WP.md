# TSGCTF2021 Baba is Flag WP

## 题目简述

服务实现了一套被刻意改坏的 secp256k1 ECDSA。签名和验证都删掉了标准公式中的 $r$ 因子：

```ruby
# 标准形式应含 x * d
s = (z + @d) * inv(k) % @n

# 标准形式应含 Q * (x * inv(s))
x2, _ = (@G * (z * inv(s)) + @Q * inv(s)).xy
return x == x2
```

服务只允许为固定消息 `Baba` 取得签名，却会在攻击者为消息 `Flag` 提交有效签名时返回 flag。每次连接最多进行五次操作。

## 解题过程

先请求一份 `Baba` 的签名 $(r,s)$。令：

$$
z_B=\operatorname{SHA256}(\text{Baba}),\qquad R=[k]G.
$$

服务返回的 $r$ 是点 $R$ 的横坐标，而非标准签名中再模曲线阶后的其他结构。签名公式给出：

$$
s=(z_B+d)k^{-1}\pmod n.
$$

两边乘以点 $R=[k]G$：

$$[s]R=[z_B+d]G=[z_B]G+Q$$

仅知道横坐标 $r$ 时，曲线上有两个互为相反数的候选点 $R$。OpenSSL 可以分别用压缩点前缀 2 和 3 构造它们：

```ruby
p1 = OpenSSL::PKey::EC::Point.new(curve,
  OpenSSL::BN.new(r | (2 << curve.degree)))
p2 = OpenSSL::PKey::EC::Point.new(curve,
  OpenSSL::BN.new(r | (3 << curve.degree)))
```

验证 `Flag` 时选择 $s'=1$。被修改的验证公式会计算：

$$[z_F]G+Q$$

其中 $z_F=\operatorname{SHA256}(\text{Flag})$。利用上面的关系，可以直接从签名点构造这个目标点：

$$
[z_F]G+Q
=[s]R+[z_F-z_B]G
$$

因此分别对两个候选 $R$ 计算右式横坐标：

```ruby
z_b = Digest::SHA256.hexdigest('Baba').hex
z_f = Digest::SHA256.hexdigest('Flag').hex

x1, _ = (p1 * s + curve.generator * (z_f - z_b)).xy
x2, _ = (p2 * s + curve.generator * (z_f - z_b)).xy
```

真实的 $R=[k]G$ 只有一个，但服务允许继续尝试，所以依次提交 `(msg="Flag", x=x1, s=1)` 和 `(msg="Flag", x=x2, s=1)` 即可。正确候选通过验证后返回：

```text
TSGCTF{HACKER_IS_YOU._POINT_IS_MOVE._POINT_ON_CURVE_IS_HACKED._YOU_IS_WIN.}
```

## 方法总结

这题不需要恢复私钥 $d$。删除 ECDSA 中的 $r$ 因子后，一份签名直接泄露了点关系 $[s]R=[z]G+Q$；改变消息哈希只需再加一个公开的 $[z'-z]G$。横坐标造成的正负点歧义只有两个候选，可直接在线尝试。密码协议中的每一个乘法因子都承担明确的绑定作用，不能在签名端和验证端“对称删掉”后就认为安全性仍然成立。
