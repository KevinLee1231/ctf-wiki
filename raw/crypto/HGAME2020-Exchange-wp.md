# Exchange

## 题目简述

题目模拟 Diffie–Hellman 密钥交换，但通信双方没有认证彼此的公钥。攻击者可以分别向 Alice 和 Bob 发送自己的公钥，建立两条不同的共享密钥，再对双方使用的乘法“加密”进行解密和重加密，完成中间人攻击并拼出 flag。

## 解题过程

服务端公开素数 $p$、生成元 $g$ 以及：

$$
A=g^a\bmod p,\qquad B=g^b\bmod p.
$$

攻击者选择两份私钥，并分别伪装成通信对端：

```python
my_a = 0x1337
my_b = 0x7331

my_A = pow(g, my_a, p)  # 发给 Bob，冒充 Alice
my_B = pow(g, my_b, p)  # 发给 Alice，冒充 Bob
```

于是攻击者与 Alice、Bob 分别拥有共享密钥：

$$
S_A=A^{my_b}\bmod p,
$$

$$
S_B=B^{my_a}\bmod p.
$$

题目的消息加密不是标准对称密码，而是：

$$
C=mS\bmod p.
$$

只要计算 $S$ 在模 $p$ 下的逆元，就能恢复明文：

$$
m=CS^{-1}\bmod p.
$$

中间人转发过程如下：

```python
from Crypto.Util.number import inverse, long_to_bytes

S_A = pow(A, my_b, p)
S_B = pow(B, my_a, p)

# 截获 Bob 发往 Alice 的半段 flag
m_b = C_b * inverse(S_B, p) % p
forward_to_alice = m_b * S_A % p

# 截获 Alice 发往 Bob 的半段 flag
m_a = C_a * inverse(S_A, p) % p
forward_to_bob = m_a * S_B % p

# 按协议中两段的原始顺序拼接
flag = long_to_bytes(m_a) + long_to_bytes(m_b)
print(flag)
```

向服务端转发时，应把 `forward_to_alice` 和 `forward_to_bob` 放回各自对应的交互位置，维持双方协议正常结束。公开脚本有一处容易误抄的变量：给 Bob 重加密时必须使用 `m_a`，不能再次使用 `m_b`。

原 PDF 只列出题名，交互流程由 [HGAME2020 Crypto Writeup](https://blog.soreatu.com/posts/writeup-for-crypto-problems-in-hgame-2020/) 补足；正文已经包含双方公钥替换、两条共享密钥、解密与重加密的全部关系，不依赖外链才能理解。

## 方法总结

- 核心漏洞：Diffie–Hellman 只建立共享秘密，不自动认证公钥来源，因而可被主动中间人拆成两条会话。
- 关键计算：分别使用 $S_A$、$S_B$ 解密，并用另一侧共享密钥重新加密后再转发。
- 修复方向：对 DH 临时公钥和握手上下文做数字签名，或使用已认证的密钥交换协议；不要自制 $C=mS\bmod p$ 这种裸乘法加密。
