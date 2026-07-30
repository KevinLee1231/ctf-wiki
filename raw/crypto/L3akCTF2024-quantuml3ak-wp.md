# L3akCTF 2024 QuantumL3ak Writeup

## 题目简述

服务生成一个 8 量子位噪声电路：随机选择 4 个控制位，对每个控制位施加 Hadamard 门，再分别通过 CNOT 与其余 4 位配对。这会形成 4 对测量结果始终相关的量子位。每次测量最终由同一个 Python `random.Random` 实例调用 `choices` 取样；退出时，该 PRNG 又生成 128 位 AES 密钥和 IV 来加密 flag。

关键问题不是攻击量子算法本身，而是通过可控电路提高测量输出的信息量，恢复 MT19937 状态，再预测后续 AES 参数。

## 解题过程

### 识别并抵消噪声电路

不上传电路时连续测量 32 次。属于同一 Bell 对的两个输出位在每次测量中都相同，因此可以按整列比较找出 4 对量子位。

噪声对每一对执行：

```text
H control
CX control dependent
```

逆变换需要逆序执行。由于 H 和 CNOT 都是自身的逆，官方 solver 为每一对上传：

```text
CX control dependent
H control
```

抵消噪声后，再对全部 8 位施加 H：

```python
for i in range(8):
    gates.append(f"H {i}")
```

这样测量结果在 256 个 8 位状态上近似均匀，每次结果都直接泄漏 PRNG 取样值的 8 个有效位。相比原始电路只有 16 种相关状态，这一步“最大化熵”显著增加了可用于恢复状态的信息。

### 恢复 MT19937 状态

CPython 的 `random()`/`choices()` 会连续消耗 MT19937 的 32 位输出。官方脚本把每次测得的 8 位放在一个 32 位字的已知位置，其余 24 位标为未知，并把另一个被消耗的 32 位字整体标为未知：

```python
bits = [None] * 24
for bit in reversed(result):
    bits.append(bit - 0x30)

solver.submit(32, bits)
solver.submit(32, None)
```

`MT19937Solver` 将 tempering 和状态转移表示为 $\mathrm{GF}(2)$ 上的线性方程。持续测量直至约束足够后，Sage 解出 624 个 32 位内部状态字，并将它们装回 Python RNG：

```python
state_words = list(map(int, solver.solve()))
rng = random.Random()
rng.setstate((3, tuple(state_words + [624]), None))
```

随后按服务端已经执行的测量次数推进 RNG，预测接下来的 128 位 key 和 128 位 IV：

```python
for _ in range(consumed_words):
    rng.getrandbits(32)

key = rng.getrandbits(128).to_bytes(16, "little")
iv = rng.getrandbits(128).to_bytes(16, "little")
plaintext = AES.new(key, AES.MODE_CBC, iv).decrypt(ciphertext)
```

仓库构建文件中的 flag 验证结果为：

```text
L3AK{m4x1m1z3_3ntr0py}
```

## 方法总结

- 核心技巧：通过可控量子门改变输出分布，提高 MT19937 部分输出泄漏的熵，再用线性方程恢复完整状态。
- 识别信号：同一非密码学 PRNG 同时负责可观测随机选择和后续密钥生成时，应统计每次接口调用消耗了多少状态字。
- 复用要点：状态恢复不仅要得到 624 个字，还必须精确复现调用顺序、被忽略的输出以及整数端序；任何一次漏算都会使预测的 IV 与服务端不一致。
