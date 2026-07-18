# SuanP01y

## 题目简述

题目在商环

```text
S = GF(2)[X] / (X^16381 - 1)
```

中生成两个低重量多项式，将它们分别循环移位后，只公开商 `hint = h1 / h0`。flag 使用 `MD5(str(h0))` 作为 AES-CTR 密钥加密。由于未移位的两个秘密多项式次数都不超过 `floor(r/3)`，可以把求解建模成模 `X^r-1` 的多项式有理重构，再枚举不可避免的循环移位等价类。

## 解题过程

### 还原生成关系

源码参数为：

```python
r, d = 16381, 41

def sparse_poly():
    return sum(x^i for i in set(sample(range(r // 3 + 1), d)))
```

生成互素的 `t0,t1` 后，题目选择随机移位 `a,b`：

```text
h0 = t0 * X^a
h1 = t1 * X^b
hint = h1 / h0
```

所有运算都在 `S` 中进行，乘 `X^k` 等价于把系数向量循环移位。因此：

```text
hint = (t1 * X^(b-a)) / t0  mod (X^r - 1)
```

也就是说，已知一个模多项式意义下的商，分母是次数不超过 `r/3` 的稀疏多项式。源码还保证 `gcd(t0,t1)=1` 且 `h0` 在商环中可逆，满足有理重构所需条件。

### 用扩展欧几里得算法做有理重构

目标是找到 `N,D`，使：

```text
hint * D == N  mod (X^r - 1)
```

并满足分子、分母的次数边界。Sage 的 `rational_reconstruction()` 内部使用多项式扩展欧几里得算法，可以直接写成：

```python
r = 16381
R.<x> = PolynomialRing(GF(2))
mod_poly = x^r - 1

den_bound = r // 3
num_bound = r - den_bound
num, den = hint.rational_reconstruction(
    mod_poly, num_bound, den_bound
)
```

`den` 恢复出 `t0` 的一个循环移位代表。GF(2) 中不存在额外的非平凡常数倍歧义，但在模 `X^r-1` 的环里，同时给分子和分母乘 `X^k` 不改变商，所以仅靠 `hint` 无法确定绝对移位 `a`。

### 枚举移位并解密

取出 `den` 中系数为 1 的指数集合，然后枚举全部 `r` 个循环移位：

```python
den_exps = [m.degree() for m in den.monomials()]
ciphertext = bytes.fromhex(ciphertext_hex)

for shift in range(r):
    exps = [(e + shift) % r for e in den_exps]
    candidate = sum(x^e for e in exps)

    # 必须使用 Sage 对多项式的原始字符串格式
    key = md5(str(candidate).encode()).digest()
    cipher = AES.new(key, AES.MODE_CTR, nonce=b'suanp01y')
    plaintext = cipher.decrypt(ciphertext)

    if plaintext.startswith(b'RCTF{'):
        print(shift, plaintext.decode())
        break
```

实现时最容易忽略的是密钥输入不是系数向量，而是 `str(h0)`。若自行格式化多项式，必须与 Sage 保持完全一致：指数降序、`X`/`X^n` 的写法以及项之间的空格都会影响 MD5。直接在同一个 Sage 多项式环中重建候选并调用 `str()` 最稳妥。

## 方法总结

题目公开的是循环多项式商，但秘密分子、分母具有远小于模次数的结构边界，这正是有理重构的适用信号。扩展欧几里得算法负责从模同余中找回短分母，枚举负责消除商环中的循环移位等价。最后还要严格复现序列化到密钥的过程；数学对象恢复正确，不代表它的字符串编码也自动一致。
