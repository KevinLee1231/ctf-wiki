# solid

## 题目简述

题目给出一个 Solidity 合约：六字节 PIN 必须满足六组加法、减法和异或约束，通过后用该 PIN 作为 RC4 密钥解密固定数据。官方求解脚本用 8 位 Z3 `BitVec` 得到预期 PIN `sagnat`，并解出 flag。不过合约声明为 Solidity 0.8.19，且没有 `unchecked`；官方模型把 `uint8` 加法当成模 $256$ 运算，与合约默认的溢出回退语义不一致。这是题目实现中的实际缺陷，不能把“脚本解密成功”写成“链上校验成功”。

## 解题过程

### 建立官方预期的 8 位约束

合约对六个字节 $b_0,\ldots,b_5$ 施加：

$$
\begin{aligned}
b_0\oplus b_5 &= 7 \\
b_1\oplus b_2 &= 6 \\
b_2+b_3 &= \mathtt{0xD5} \\
\operatorname{int8}(b_3)-\operatorname{int8}(b_4) &= 13 \\
b_0+b_1+b_2 &= \mathtt{0x3B} \\
b_4\oplus b_5 &= \mathtt{0x15}
\end{aligned}
$$

官方脚本把每个变量定义为 8 位位向量：

```python
from z3 import BitVec, Solver, sat

b = [BitVec(f"b{i}", 8) for i in range(6)]
s = Solver()
s.add(b[0] ^ b[5] == 7)
s.add(b[1] ^ b[2] == 6)
s.add(b[2] + b[3] == 0xD5)
s.add(b[3] - b[4] == 13)
s.add(b[0] + b[1] + b[2] == 0x3B)
s.add(b[4] ^ b[5] == 0x15)

assert s.check() == sat
m = s.model()
key = bytes(m[x].as_long() for x in b)
print(key)
```

得到：

```text
b'sagnat'
```

这组字节满足官方脚本的模运算。例如：

$$
(115+97+103)\bmod 256=315\bmod256=59=\mathtt{0x3B}
$$

### 用预期 PIN 执行 RC4

合约中的 RC4 是标准 KSA 加 PRGA。对固定密文执行相同过程：

```python
def rc4(data, key):
    state = list(range(256))
    j = 0
    for i in range(256):
        j = (j + state[i] + key[i % len(key)]) % 256
        state[i], state[j] = state[j], state[i]

    i = j = 0
    out = bytearray()
    for value in data:
        i = (i + 1) % 256
        j = (j + state[i]) % 256
        state[i], state[j] = state[j], state[i]
        stream = state[(state[i] + state[j]) % 256]
        out.append(value ^ stream)
    return bytes(out)

ciphertext = bytes.fromhex(
    "09f7578b222a4c5b1e58c544c821af39"
    "af8a1efe44da9a9f5ed9cadbac"
)
print(rc4(ciphertext, b"sagnat").decode())
```

预期明文为：

```text
shellmates{3Vm_Byt3C0de_duD3}
```

### 核对 Solidity 0.8 的溢出问题

[Solidity 0.8 的官方说明](https://docs.soliditylang.org/en/v0.8.19/control-structures.html#checked-or-unchecked-arithmetic)规定：整数算术默认检查上溢和下溢，只有放进 `unchecked` 块才会回绕。合约中的表达式：

```solidity
uint8(b[0]) + uint8(b[1]) + uint8(b[2])
```

没有 `unchecked`。对 `sagnat` 而言，前两项为 212，再加 103 得到 315，超出 `uint8` 上限 255，执行会以算术 `Panic` 回退，根本到不了与 `0x3B` 的比较。

进一步按非回绕整数语义枚举六组约束，没有任何解；因此不能换一个 PIN 绕开该问题。若要让合约实现官方预期，至少应把有关表达式放入 `unchecked`，或显式提升到更宽整数后再按设计取模。当前仓库只能可靠复现“官方脚本恢复预期明文”，不能证明 `checker` 在原合约上返回 `true`。

## 方法总结

约束求解必须复刻目标语言语义，不能只复刻公式外形。Z3 `BitVec(8)` 的加减天然模 $256$，而 Solidity 0.8 的窄整数算术默认检查溢出；两者在本题恰好产生决定性差异。预期 flag 可由官方模型和 RC4 数据恢复，但链上验证路径存在不可用缺陷。高质量题解应同时给出预期解法与实现边界，不能把离线解密结果包装成未发生的合约通过证据。
