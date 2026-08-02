# TSGCTF2021 This is DSA WP

## 题目简述

服务先生成 256 位素数 $q$，却让客户端自行提交 DSA 参数 $p,h$：

```python
q = getPrime(256)
p = int(input('p? '))
h = int(input('h? '))

g = pow(h, (p - 1) // q, p)
x = randrange(q)
y = pow(g, x, p)
```

随后服务用私钥 $x$ 构造 DSA，并要求客户端给出 `sha256("flag")` 的有效签名。题目使用的 PyCryptodome 私有分支没有充分验证攻击者提供的 $p$ 是否为符合 DSA 要求的素数，因此可以把离散对数放进易解的 $q$-进主单位群。

## 解题过程

选择：

$$p=q^8$$

并确保它恰好为 2048 位；若位数不足就重新连接等待下一组 $q$。模 $q^8$ 的单位群阶为：

$$\varphi(q^8)=q^7(q-1)$$

令：

$$h=2^{\varphi(q^8)/q}\bmod q^8$$

这样 $h$ 落在阶整除 $q$ 的子群中。服务端再按 $(p-1)/q$ 的整数商计算 $g$；对 $p=q^8$，该指数等于 $q^7-1$，模 $q$ 为 $-1$，所以非平凡的 $g$ 仍具有 $q$ 阶。官方 solver 的参数构造为：

```sage
n = 8
p = q^n
phi = q^(n - 1) * (q - 1)
h = pow(2, phi // q, p)
```

服务公开：

$$
y=g^x\pmod{q^8},\qquad 0\le x<q.
$$

此时 $g,y$ 都位于 $q$ 进单位群中。$q$ 进对数把乘法线性化：

$$\log(y)=x\log(g)$$

因此在足够精度的 $\mathbb{Q}_q$ 中直接计算：

```sage
K = Qp(q, 10)
Kg = K(g)
Ky = K(y)
x = ZZ((Ky.log() / Kg.log()).polynomial()[0][0])
assert pow(g, x, p) == y
```

强验证 `pow(g, x, p) == y` 很重要：它同时排除精度不足、平凡生成元和代表元选择错误。恢复私钥后，用同一组 $(p,q,g,y,x)$ 构造 DSA 私钥，对 `SHA256("flag")` 正常签名，并把签名字节做 Base64 编码：

```python
dsa = DSA.construct((y, g, p, q, x))
dss = DSS.new(dsa, 'fips-186-3')
signature = dss.sign(SHA256.new(b'flag'))
print(base64.b64encode(signature).decode())
```

服务验签通过后返回：

```text
TSGCTF{WOW_AMAZING_DSA_IS_TOTALLY_BROKEN}
```

## 方法总结

这题的漏洞是允许不可信客户端选择群参数，却没有验证 $p$ 为合格素数、$q\mid p-1$、$g$ 的精确阶等 DSA 前提。构造 $p=q^8$ 后，公钥离散对数落入可用 $q$ 进对数线性化的主单位群，私钥不再困难。接受外部密码参数时必须执行标准规定的完整域参数验证；更稳妥的做法是只允许使用预先批准的参数组，而不是根据用户输入临时建群。
