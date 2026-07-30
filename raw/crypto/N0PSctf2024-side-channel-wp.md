# Side Channel

## 题目简述

附件提供 AES-128-ECB 实现 `aes.c`、头文件和 20,000 组采样数据 `ta.dat`。每行依次记录 16 字节明文、16 字节密文和整次加密耗时。目标不是破解 AES 数学结构，而是从实现的输入相关运行时间中恢复 128 位密钥。

源码的 `mixColumns` 会调用以下有限域乘法函数：

```c
uint8_t gf256_time(const uint8_t a, const uint8_t b) {
    uint8_t r, x, y;

    for (r = 0, x = a, y = b; y; x = gf256_xtime(x), y >>= 1) {
        if (y & 1) {
            r ^= x;
        }
    }
    return r;
}
```

循环次数取决于右操作数 `b` 的最高置位位置，造成可测量的计时侧信道。

## 解题过程

### 建立首轮计时模型

AES 首轮中，第 $i$ 个明文字节 $p_i$ 先与密钥字节 $k_i$ 异或，再经过 S-box：

$$
b_i=S(p_i\oplus k_i)
$$

`shiftRows` 只移动位置，不改变字节值。随后 `mixColumns` 把该值多次作为 `gf256_time` 的右操作数。函数对 `b=0` 不进入循环，其余值的循环次数等于二进制位长，因此可使用：

$$
\operatorname{TM}(b)=
\begin{cases}
0,&b=0\\
\lfloor\log_2 b\rfloor+1,&b>0
\end{cases}
$$

作为相对计时模型。一个字节只有 256 个密钥候选，可以逐字节猜测，而不必枚举完整的 $2^{128}$ 密钥空间。

### 用 Pearson 相关系数选择密钥

对每个字节位置和候选 $k$，在全部采样上计算模型值
`TM(S(p_i ^ k))`，再与实际耗时 $T$ 计算 Pearson 相关系数：

$$
\rho(X,T)=
\frac{\sum_j(X_j-\bar X)(T_j-\bar T)}
{\sqrt{\sum_j(X_j-\bar X)^2}\sqrt{\sum_j(T_j-\bar T)^2}}
$$

正确候选会与总耗时中的首轮局部贡献产生微弱正相关；错误候选通常没有稳定相关性。下面的脚本从附件 `aes.c` 直接提取 S-box，避免另行抄写常量：

```python
import re
from pathlib import Path

import numpy as np


source = Path("aes.c").read_text(encoding="utf-8")
match = re.search(
    r"const uint8_t sbox\[256\]\s*=\s*\{(.*?)\};",
    source,
    flags=re.S,
)
assert match is not None
sbox = np.array(
    [int(value, 16) for value in re.findall(r"0x[0-9a-fA-F]+", match.group(1))],
    dtype=np.uint8,
)
assert len(sbox) == 256

plaintexts = []
timings = []
for line in Path("ta.dat").read_text(encoding="ascii").splitlines():
    plain_hex, _cipher_hex, timing = line.split()
    plaintexts.append(bytes.fromhex(plain_hex))
    timings.append(float(timing))

plain = np.frombuffer(b"".join(plaintexts), dtype=np.uint8).reshape(-1, 16)
observed = np.asarray(timings, dtype=np.float64)
observed -= observed.mean()
observed_norm = np.linalg.norm(observed)

time_model = np.array([value.bit_length() for value in range(256)])
recovered = bytearray()

for byte_index in range(16):
    scores = []
    column = plain[:, byte_index]

    for candidate in range(256):
        model = time_model[sbox[np.bitwise_xor(column, candidate)]].astype(float)
        model -= model.mean()
        denominator = np.linalg.norm(model) * observed_norm
        score = float(model @ observed / denominator) if denominator else 0.0
        scores.append(score)

    recovered.append(int(np.argmax(scores)))

print(recovered.hex().upper())
```

输出密钥：

```text
740279C13A16787F9E9BF14C2F61D669
```

### 用已知明密文对复核

不能只依赖“相关系数最高”就宣告成功。用恢复的密钥加密 `ta.dat` 第一行明文：

```python
from Crypto.Cipher import AES

first_plain, first_cipher, _ = (
    Path("ta.dat").read_text(encoding="ascii").splitlines()[0].split()
)
key = bytes.fromhex("740279C13A16787F9E9BF14C2F61D669")

assert AES.new(key, AES.MODE_ECB).encrypt(
    bytes.fromhex(first_plain)
) == bytes.fromhex(first_cipher)
```

验证通过后，flag 为：

```text
N0PS{740279C13A16787F9E9BF14C2F61D669}
```

## 方法总结

- 核心技巧：把有限域乘法的可变循环次数建模为字节位长，并对首轮中可独立枚举的 16 个密钥字节做相关计时分析。
- 识别信号：加密算法本身标准，但实现含数据相关分支或循环；题目同时提供大量明文和耗时样本。
- 复用要点：计时模型只需与真实局部耗时近似线性相关，不必精确预测整次加密时间。候选选出后必须用附件中的明密文对重新加密验证，防止采样噪声或模型偏差造成误判。
