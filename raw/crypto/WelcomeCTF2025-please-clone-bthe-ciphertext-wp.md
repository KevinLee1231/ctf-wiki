# Please Clone Bthe Ciphertext

## 题目简述

服务公开随机明文 $P$ 和初始向量 $IV$，但隐藏 AES 密钥，并要求提交 $P$ 在该密钥和 $IV$ 下的自定义 PCBC 密文。用户可以选择另一组明文与 IV 调用同一密钥的加密 oracle，但服务禁止同时原样复用公开的 $P$ 与 $IV$。

每块的计算为：

$$
C_i=E_K(P_i\oplus V_i),\qquad V_{i+1}=P_i\oplus C_i
$$

## 解题过程

任选一个不同的 16 字节 $IV'$，令：

$$
\Delta=IV\oplus IV',\qquad P_i'=P_i\oplus\Delta
$$

首块进入 AES 的输入保持不变：

$$
P_0'\oplus IV'=(P_0\oplus\Delta)\oplus IV'=P_0\oplus IV
$$

所以 $C_0'=C_0$。继续看反馈值：

$$
V_1'=P_0'\oplus C_0'=V_1\oplus\Delta
$$

于是第二块也满足 $P_1'\oplus V_1'=P_1\oplus V_1$。该关系可归纳到全部块，因此 oracle 返回的整段密文正是目标密文。

官方 solver 的核心操作如下：

```python
from os import urandom


def xor(a, b):
    return bytes(x ^ y for x, y in zip(a, b))


new_iv = urandom(16)
delta = xor(iv, new_iv)
blocks = [plaintext[i:i + 16] for i in range(0, len(plaintext), 16)]
new_plaintext = b"".join(xor(block, delta) for block in blocks)

# 把 new_iv、new_plaintext 交给加密 oracle；
# 再把 oracle 返回的 ciphertext 原样提交给校验选项。
```

服务只比较输入字节是否与原值完全相等，无法识别这组等价输入。提交 oracle 密文后得到：

```text
grey{bthe_is_a_legit_word_i_bthink_6afc024a3e3b735cc3c59337e42c2b97}
```

## 方法总结

- 核心技巧：同时调整 PCBC 的 IV 和每个明文块，使每轮送入 AES 的异或结果保持不变。
- 识别信号：加密 oracle 与目标复用密钥，服务仅禁止复制原始参数，却没有禁止等价的内部状态。
- 复用要点：不能只修正第一块；必须证明反馈状态中的差分会持续保持，并把同一 $\Delta$ 应用于所有明文块。
