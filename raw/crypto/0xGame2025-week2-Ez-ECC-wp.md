# Ez_ECC

## 题目简述

题目使用 NIST P-256 曲线，公开点 $P$ 与 $Q=sP$，再以 `SHA256(str(s))` 作为 AES-ECB 密钥加密 flag。正常 P-256 离散对数不可行，但源码把私钥限制在 $1\le s\le2^{40}$，可用 Baby-step Giant-step 将复杂度降到约 $2^{20}$ 次点运算和同量级存储。

## 解题过程

令 $m=\lceil\sqrt{2^{40}}\rceil$，把未知量写成 $s=am+b$。预计算并保存 $Q-bP$，再枚举 $amP$；两者相等时：

$$
amP=Q-bP\Longrightarrow Q=(am+b)P
$$

以下是核心求解骨架；曲线参数、$P,Q$ 和密文替换为题目输出，点运算由附件 `utils.py` 的 `Curve` 类提供：

```python
from math import ceil, sqrt
from hashlib import sha256
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad
from utils import Curve

# p, a, b, P, Q, ciphertext = 题目输出与附件参数
C = Curve(a, b, p)
P = C(Px, Py)
Q = C(Qx, Qy)

def bsgs(base, target, bound=2**40):
    m = ceil(sqrt(bound))
    baby = {str(target - j * base): j for j in range(m)}
    step = m * base
    for i in range(m):
        hit = baby.get(str(i * step))
        if hit is not None:
            return i * m + hit
    raise ValueError('discrete log not found in bound')

s = bsgs(P, Q)
assert s * P == Q
key = sha256(str(s).encode()).digest()
plaintext = AES.new(key, AES.MODE_ECB).decrypt(ciphertext)
print(unpad(plaintext, 16))
```

恢复 $s$ 后必须用 `s * P == Q` 验证，再严格复现十进制字符串序列化与 SHA-256；直接对整数定长字节做哈希会得到不同密钥。

## 方法总结

- 核心技巧：利用异常小的椭圆曲线私钥上界执行 BSGS。
- 识别信号：标准强曲线本身没有问题，但标量只来自约 40 位空间。
- 复用要点：BSGS 的时间与空间都是 $O(\sqrt N)$；恢复离散对数后还要复现题目指定的密钥派生和填充方式。
