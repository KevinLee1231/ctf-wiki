# my array generator

## 题目简述

题目实现一个弱化的 MAG 风格流密码。32 字节 key 被按 4 字节一组扩展成 128 个寄存器：第 $i$ 个寄存器取 `key[(4*i) % 32 : (4*i) % 32 + 4]`。初始化虽做了 $2^{14}$ 次更新，但更新只有 XOR、移位和由比较结果决定的全 1 常量 XOR；没有把不同 key 字节扩散为不可分离的非线性关系。

附件给出 1280 字节已知明文和密文。又因 `random.seed(1234)` 固定，`get_keystream()` 每次选择的寄存器字节位置也是已知的。故可得到 1280 个 keystream 字节并追踪它们对八个 32 位初始 key word 的依赖。

## 解题过程

先计算已知 keystream：

$$
z_i=\mathrm{plaintext}_i\oplus\mathrm{ciphertext}_i.
$$

官方 Sage solver 按 bit 在 $\operatorname{GF}(2)$ 上追踪八个独立 key word 的线性依赖，并同时记录每次取出的 word byte 下标。忽略条件分支中的 `^ 0xffffffff` 后，状态更新可写作线性形式：

```python
carry += r1          # 在 GF(2) 中等价于 XOR
registers = registers[1:]
registers.append(registers[-1] + carry)
```

被忽略的分支只会把一个输出字节整体翻为 `^ 0xff`，不会改变它依赖于哪个 key 字节。用相同随机种子模拟 1280 次输出，筛选符号表达式恰为单项式的位置。每个此类样本给出某个 key 字节，或其按位补；以 flag/key 是 ASCII 为判据即可消除补码歧义。官方代码以 `possibility & 0x80 == 0` 选择 ASCII 候选。

等价的 `solve-z3.py` 直接以八个 32 位 BitVec 建模前 80 个输出。每个观测约束为：

```python
solver.add(Or(kz == (p ^ c), kz == (p ^ c) ^ 0xff))
```

并把每个 key byte 限定在可打印 ASCII 范围；求得的八个字按 big-endian 拼接即为 32 字节 key/flag：

```text
DUCTF{n0_d1ffus10n_m4k3s_m3_sad}
```

## 方法总结

大量预热并不等于安全扩散。线性移位寄存器若复用短 key，并公开已知明文，就会把 keystream 变成初始状态的线性观测；固定随机字节选择还进一步降低了不确定性。条件异或全 1 常量只带来“原字节或补码”的一位歧义，格式约束足以恢复。实际流密码应使用经过分析的设计，并使用独立 nonce，而不是自制寄存器更新函数。
