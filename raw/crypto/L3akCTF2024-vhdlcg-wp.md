# L3akCTF 2024 VHDLCG Writeup

## 题目简述

题目用 VHDL 实现了一个 28 位线性同余生成器（LCG），每次只输出状态的最高 8 位。连续 40 个输出字节组成密钥流，与 40 字节 flag 按位异或：

$$
x_{i+1}=(Ax_i+C)\bmod M,
$$

$$
y_i=\left\lfloor\frac{x_i}{2^{20}}\right\rfloor,\qquad
c_i=p_i\oplus y_i.
$$

VHDL 中的参数为：

```text
A = 73067557
C = 111837721
M = 0x10000000 = 2^28
```

虽然载体是 VHDL，决定性障碍是从截断 LCG 输出恢复完整状态，因此按密码分析归入 Crypto。

## 解题过程

flag 固定以 `L3AK{` 开头。将密文前 5 字节与该前缀异或，就得到 5 个连续的 LCG 高 8 位输出：

```python
known = [ciphertext[i] ^ b"L3AK{"[i] for i in range(5)]
```

每个完整状态都可写成：

$$
x_i=2^{20}y_i+z_i,\qquad 0\le z_i<2^{20}.
$$

把 LCG 递推代入后，未知量只剩各状态的低 20 位 $z_i$。这些未知量相对模数 $2^{28}$ 较小，可以按 Frieze 等人的截断同余状态恢复方法构造格并用 LLL 找到近向量。官方脚本采用的实现来自 [crypto-attacks 的 truncated state recovery](https://github.com/jvdsn/crypto-attacks/blob/master/attacks/lcg/truncated_state_recovery.py)；本题所需的完整参数和调用方式如下：

```python
from sage.all import QQ, ZZ, matrix, vector


def recover_states(outputs, state_bits, output_bits, modulus, a, c):
    missing_bits = state_bits - output_bits

    # 把已知高位移回状态位置，并消去每一步的常数增量。
    delta = c % modulus
    y = vector(ZZ, outputs)
    for i in range(len(y)):
        y[i] = (y[i] << missing_bits) - delta
        delta = (a * delta + c) % modulus

    # 齐次 LCG 对应的格基。
    basis = matrix(ZZ, len(y), len(y))
    basis[0, 0] = modulus
    for i in range(1, len(y)):
        basis[i, 0] = a**i
        basis[i, i] = -1
    basis = basis.LLL()

    # 求与已知高位向量最接近的模 modulus 向量。
    target = basis * y
    for i in range(len(target)):
        target[i] = round(QQ(target[i]) / modulus) * modulus - target[i]

    low = list(basis.solve_right(target))

    # 加回高位和非齐次增量，得到完整连续状态。
    delta = c % modulus
    states = []
    for i in range(len(low)):
        states.append(int(y[i] + low[i] + delta))
        delta = (a * delta + c) % modulus
    return states


A = 73067557
C = 111837721
M = 0x10000000

ciphertext = bytes.fromhex(
    "50E9A87F3B317119319E286313520AFD"
    "E00A710D156B75482373F4332473A876"
    "E2BAD778B67FD5B4"
)

known_prefix = b"L3AK{"
outputs = [
    ciphertext[i] ^ known_prefix[i]
    for i in range(len(known_prefix))
]

states = recover_states(outputs, 28, 8, M, A, C)
for previous, current in zip(states, states[1:]):
    assert (previous * A + C) % M == current

key_stream = bytearray(state >> 20 for state in states)
state = states[-1]
while len(key_stream) < len(ciphertext):
    state = (state * A + C) % M
    key_stream.append(state >> 20)

flag = bytes(c ^ k for c, k in zip(ciphertext, key_stream))
print(flag.decode())
```

运行官方 Sage solver 已验证输出：

```text
L3AK{Y0u_C4N_d0_M4NY_Th1ngS_w1Th_VHDL!!}
```

当前仓库快照存在一处必须说明的版本错配：`src/output.txt` 与 `dist/output.txt` 记录的是以 `CBC17C...` 开头的另一组密文，而官方 solver 硬编码的 `50E9A8...` 与当前 `flag.txt`、`seed.txt` 和 VHDL 递推逻辑相符。复现官方结果时不要混用这两组 artifact；处理实际赛题输出时，则应把脚本中的 `ciphertext` 替换为同一实例给出的密文。

## 方法总结

已知明文前缀把密文字节直接转成了 LCG 的截断输出。只泄露高 8 位并不意味着每轮独立缺少 20 位：连续状态受同一个线性递推约束，多个样本可通过格规约联合恢复低位。

遇到“LCG + 只输出高位 + XOR 密钥流”时，应记录状态位宽、输出位宽、模数、乘数、增量和输出时序，再检查是否存在已知明文。与此同时要核对 solver、生成源码与输出文件是否来自同一版本；仓库内文件名一致并不等于数据一定一致。
