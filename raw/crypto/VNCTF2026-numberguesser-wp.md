# NumberGuesser

## 题目简述

题目给了一个基于 Python `random` 的猜数服务。服务端用 8 字节随机种子初始化全局 MT19937，先生成 624 个 32-bit hint，再继续从同一个 PRNG 中取 128 bit 作为 AES-CBC key，并用 `seed * 2` 作为 IV 加密 flag。选手最多查询 10 个 hint 的下标，需要从少量输出恢复初始化种子或足够的内部状态，再解密密文。

核心代码如下：

```python
class NumberGuesser:
    def __init__(self, MAX_CHANCES: int = 10, n: int = 624):
        self.seed = os.urandom(8)
        random.seed(self.seed)
        self.hints = [random.getrandbits(32) for _ in range(self.n)]
        self.enc = self.encrypt_flag(flag, self.seed)

    def encrypt_flag(self, flag: str, iv: bytes) -> bytes:
        key = random.getrandbits(128).to_bytes(16, "big")
        ct = AES.new(key, AES.MODE_CBC, iv * 2).encrypt(pad(flag.encode(), AES.block_size))
        return ct
```

这不是常规“收集 624 个输出克隆 MT 状态”的题，因为查询次数只有 10 次。关键在于 Python `random.seed(bytes)` 的初始化过程可逆；通过特定位置的输出反推 MT 初始化数组中的相邻状态，再恢复 64-bit seed。

## 解题过程

官方题解给出的预期参考是 [Python random prediction](https://stackered.com/blog/python-random-prediction/)。攻击脚本的思路是反向 MT19937 的 temper 和初始化过程。

先准备几个逆运算：

```python
def unshiftRight(x, shift):
    res = x
    for _ in range(32):
        res = x ^ (res >> shift)
    return res

def unshiftLeft(x, shift, mask):
    res = x
    for _ in range(32):
        res = x ^ ((res << shift) & mask)
    return res

def untemper(v):
    v = unshiftRight(v, 18)
    v = unshiftLeft(v, 15, 0xefc60000)
    v = unshiftLeft(v, 7, 0x9d2c5680)
    v = unshiftRight(v, 11)
    return v
```

MT19937 每次输出前会对内部状态做 temper，因此拿到 `hint[i]` 后要先 `untemper(hint[i])` 还原为状态字。接着利用 twist 关系中相隔 227 的状态字：

```python
def invertStep(si, si227):
    X = si ^ si227
    mti1 = (X & 0x80000000) >> 31
    if mti1:
        X ^= 0x9908b0df
    X <<= 1
    mti = X & 0x80000000
    mti1 += X & 0x7fffffff
    return mti, mti1
```

实际查询选择的是 `3,4,5,6,230,231,232,233` 这些下标。用它们可以恢复初始化数组中相邻的若干高低位片段，再通过 Python seed 初始化公式反推 8 字节种子。官方 exp 的核心恢复函数如下：

```python
def init_genrand(seed):
    MT = [0] * 624
    MT[0] = seed & 0xffffffff
    for i in range(1, 624):
        MT[i] = ((0x6c078965 * (MT[i - 1] ^ (MT[i - 1] >> 30))) + i) & 0xffffffff
    return MT

def recover_Ji_from_Ii(Ii, Ii1, i):
    ji = (Ii + i) ^ ((Ii1 ^ (Ii1 >> 30)) * 1566083941)
    return ji & 0xffffffff

def recover_kj_from_Ji(ji, ji1, i):
    const = init_genrand(19650218)
    key = ji - (const[i] ^ ((ji1 ^ (ji1 >> 30)) * 1664525))
    return key & 0xffffffff

def recover_Kj_from_Ii(Ii, Ii1, Ii2, i):
    Ji = recover_Ji_from_Ii(Ii, Ii1, i)
    Ji1 = recover_Ji_from_Ii(Ii1, Ii2, i - 1)
    return recover_kj_from_Ji(Ji, Ji1, i)
```

恢复出两个 32-bit seed 分量后拼成 8 字节种子。由于最高位可能存在分支，需要用 `hint[0]` 验证候选：

```python
seed_l = recover_Kj_from_Ii(I_233, I_232, I_231, 233) - 16
seed_h1 = recover_Kj_from_Ii(I_234, I_233, I_232, 234) - 17
seed_h2 = recover_Kj_from_Ii(I_234 + 0x80000000, I_233, I_232, 234) - 17

seed = ((seed_h1 << 32) + seed_l).to_bytes(8)
random.seed(seed)
if random.getrandbits(32) != get(0):
    seed = ((seed_h2 << 32) + seed_l).to_bytes(8)
```

最后用恢复出的 seed 重新模拟服务端。服务端先生成 624 个 hints，再取一次 128-bit 作为 AES key，所以本地也要跳过 624 次：

```python
random.seed(seed)
_ = [random.getrandbits(32) for _ in range(624)]
key = random.getrandbits(128).to_bytes(16, "big")
flag = unpad(AES.new(key, AES.MODE_CBC, seed * 2).decrypt(enc), AES.block_size)
```

## 方法总结

- 核心技巧：Python MT19937 seed recovery，而不是普通的 624 输出状态克隆。
- 识别信号：`random.seed(os.urandom(8))` 后先暴露可索引的 `getrandbits(32)`，再继续用同一个 `random` 生成密钥。
- 复用要点：看到查询次数少但 seed 很短时，应优先检查语言运行时 PRNG 的 seed 初始化是否可逆；恢复 seed 后要严格复现服务端调用顺序，包括已经消耗掉的 624 个 hint。
