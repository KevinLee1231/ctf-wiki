# 4bites

## 题目简述

WelcomeCTF2021 的 4bites 模拟了一次 BB84 风格的量子密钥分发。Alice 随机选择比特与测量基，Bob 的测量基却来自一个带隐藏状态的多项式伪随机发生器。服务允许中间人连续干预五轮：前四轮可以收集 Bob 的输出，第五轮必须准确预测 Bob 的测量基，既取得测量结果，又不能让 Alice 与 Bob 的共享密钥不一致。

## 解题过程

Bob 的内部状态满足

$$
u_{i+1}=a u_i+a+b+ab \pmod p,
$$

其公开测量基为

$$
y_i=F(u_i,a,b)^{23}\pmod p,
$$

其中

$$
F(u,a,b)=u^2+3bu+5ab+7a^3+13b^5+17.
$$

模数 `mod` 是素数，且 $23$ 在模 $p-1$ 下可逆。令

$$
d=23^{-1}\pmod{p-1},
$$

则可由公开值还原 $z_i=y_i^d=F(u_i,a,b)$。

前四轮统一提交 1024 位全零测量基。这样虽然会扰动量子态并触发共享密钥不一致，但服务仍会输出 Bob 的基，并允许继续下一轮。收集四个 $y_i$ 后，在 $\mathbb Z_p$ 上建立四个方程：

```python
a, u, b = gens(PolynomialRing(Zmod(mod), ["a", "u", "b"]))
d = pow(23, -1, mod - 1)

state = u
equations = []
for output in bob_outputs:
    equations.append(genOutput(state, a, b) - pow(output, d, mod))
    state = a * state + a + b + a * b

basis = Ideal(equations).groebner_basis()
```

官方脚本对该理想求 Gröbner 基，从三条简化后的关系中解出 $a$、$u$、$b$。按状态递推四次即可得到第五轮状态，再计算第五轮 Bob 基：

```python
for _ in range(4):
    u = (a * u + a + b + a * b) % mod

predicted_basis = pow(genOutput(u, a, b), 23, mod)
```

第五轮提交 `predicted_basis`。中间人与 Bob 使用同一测量基，因此 Bob 再测量时不会改变对应量子比特，Alice 与 Bob 的筛选密钥也会一致。服务随后给出 Alice 的基、截获的测量结果和密文。

对 Alice 基与预测的 Bob 基逐位比较，只保留二者相同位置的测量比特，并按题目从高位到低位的顺序拼成整数共享密钥。最后计算 `SHA-512(str(shared_key))`，取与密文等长的前缀并异或：

```python
digest = hashlib.sha512(str(shared_key).encode("ascii")).digest()
flag = bytes(c ^ k for c, k in zip(ciphertext, digest))
```

得到：

```text
greyhats{W1th_Gr03bn3R_B@s15_b@dPRNG+QKD=h@ck3d!!!}
```

## 方法总结

本题表面上是 QKD，中间人的真正突破口却是 Bob 可预测的测量基。先利用服务允许失败多轮的设计收集代数输出，再通过逆指数和 Gröbner 基恢复隐藏状态，最后在第五轮无扰动截获密钥。实现时必须保持状态递推次数、位序和密钥筛选顺序与服务源码完全一致。
