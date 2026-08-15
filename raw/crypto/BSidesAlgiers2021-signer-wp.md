# Signer

## 题目简述

签名服务在 1024 位素数模 $p$ 下选择秘密整数 $k$。对消息哈希整数 $y$，它返回：

$$
r=(y^5+y+1)(k^3-1)\bmod p,
$$

$$
s=(y^3-y^2+1)(k^2+k+1)\bmod p.
$$

服务允许取得一个随机消息及其 $(r,s,p)$，而提交消息“shellmates”的正确签名即可作为 root 取得 flag。这里的问题不是哈希碰撞，而是两个签名分量共享了可约掉的代数因子。

## 解题过程

利用恒等式

$$
y^5+y+1=(y^3-y^2+1)(y^2+y+1),
$$

$$
k^3-1=(k-1)(k^2+k+1),
$$

可得

$$
\frac r s=(y^2+y+1)(k-1)\pmod p.
$$

所以从任意一张合法签名即可恢复：

$$
k=r\left[s(y^2+y+1)\right]^{-1}+1\pmod p.
$$

这里的 $y$ 必须由服务同时给出的随机消息计算。官方说明中把它误写成目标消息“shellmates”的哈希；那只能用于恢复秘密后伪造目标签名，不能用于消去随机签名中的因子。

~~~python
import hashlib

def hash_int(message):
    return int(hashlib.sha256(message.encode()).hexdigest(), 16)

def recover_secret(word, ticket):
    p = int(ticket["p"], 16)
    r = int(ticket["r"], 16)
    s = int(ticket["s"], 16)
    y = hash_int(word)
    denominator = s * (y * y + y + 1) % p
    return (r * pow(denominator, -1, p) + 1) % p

def forge(message, secret, p):
    y = hash_int(message)
    r = (y**5 + y + 1) * (secret**3 - 1) % p
    s = (y**3 - y**2 + 1) * (secret**2 + secret + 1) % p
    return {"s": hex(s), "r": hex(r), "p": hex(p)}
~~~

先请求一张随机 ticket，记录对应的 word，恢复 $k$；再调用 `forge("shellmates", k, p)`，把得到的 JSON 对象提交给 root 登录分支。代数公式已用独立小素数样例验证可精确恢复秘密，最终服务返回：

~~~text
shellmates{4lg3br4_15_5o_co0o0l}
~~~

## 方法总结

自制签名若把消息多项式和秘密多项式直接相乘，常会因公共因子而泄露秘密。分析这类协议时，应先对多项式做符号分解，再比较 $r/s$ 能消掉哪些项，而不是直接攻击 SHA-256。实现时还必须区分“用于恢复秘密的随机消息”和“最终需要伪造的目标消息”。
