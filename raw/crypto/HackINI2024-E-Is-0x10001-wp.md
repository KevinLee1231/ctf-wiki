# e_is_0x10001

## 题目简述

Flask 服务在启动时只生成一次 RSA 模数 `n`，但 `/encrypt_flag` 允许用户任意指定大于 1000 的公开指数 `e`。因此可以让同一 flag 在同一模数下分别使用两个互素指数加密，再实施 RSA 共模攻击，无需求解 `p`、`q`。

## 解题过程

### 获取两组共模密文

选择两个互素且均大于 1000 的指数，例如：

$$
e_1=1009,\qquad e_2=1013,\qquad \gcd(e_1,e_2)=1
$$

分别向 `/encrypt_flag` 发送 JSON：

```json
{"e": 1009}
```

```json
{"e": 1013}
```

两次响应给出相同的 `n`，并得到：

$$
c_1\equiv m^{e_1}\pmod n,\qquad c_2\equiv m^{e_2}\pmod n
$$

### 使用 Bézout 系数组合密文

扩展欧几里得算法可求出整数 $x,y$，使：

$$
x e_1+y e_2=1
$$

从而：

$$
c_1^x c_2^y\equiv m^{xe_1+ye_2}\equiv m\pmod n
$$

```python
from Crypto.Util.number import long_to_bytes
from requests import post

base = "http://host:port"
e1, e2 = 1009, 1013
r1 = post(base + "/encrypt_flag", json={"e": e1}).json()
r2 = post(base + "/encrypt_flag", json={"e": e2}).json()

n = r1["n"]
assert r2["n"] == n
c1 = int.from_bytes(bytes.fromhex(r1["cipher"]), "big")
c2 = int.from_bytes(bytes.fromhex(r2["cipher"]), "big")

def extended_gcd(a, b):
    if b == 0:
        return a, 1, 0
    g, x1, y1 = extended_gcd(b, a % b)
    return g, y1, x1 - (a // b) * y1

g, x, y = extended_gcd(e1, e2)
assert g == 1
m = pow(c1, x, n) * pow(c2, y, n) % n
print(long_to_bytes(m))
```

Python 的三参数 `pow` 在指数为负数时会计算模逆；这要求相应密文与 `n` 互素，对普通短文本几乎总能满足。输出为：

```text
shellmates{pleAs3_leT_RSA_PuBL1c_key_$tAt1c_4Nd_Us3_iT_FoR_ALL_y0uR_Mes$4G3$}
```

## 方法总结

- 核心技巧：让相同明文在固定模数下使用两个互素指数加密，通过 Bézout 恒等式恢复明文。
- 识别信号：RSA API 固定 `n` 却允许调用者控制 `e`，尤其还能重复加密同一秘密时，应立即检查共模攻击。
- 复用要点：两个指数必须互素；若 Bézout 系数为负，需要对对应密文求模逆。
