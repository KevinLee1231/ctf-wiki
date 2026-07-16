# MT19937

## 题目简述

服务端从 $[0,2^{32}-1]$ 随机选择一个整数作为 `random.Random(seed)` 的种子。攻击者可以不限次数调用 `getrandbits(2)`，最后提交 seed；猜对后返回 flag。每次调用只泄露一个 32 位 MT19937 输出的最高 2 位，其余 30 位被丢弃，因此不能直接套用“收集 624 个完整输出并逐个 untemper”的常规恢复方法。

## 解题过程

### 将截断输出建模为线性方程

MT19937 的内部状态由 624 个 32 位整数构成，共 $624\times32=19968$ 位。twist、temper 以及从每个 32 位输出截取最高 2 位，都可以视为 $\mathbb{F}_2$ 上的线性变换。连续请求 9984 次 `getrandbits(2)` 后，恰好得到 $9984\times2=19968$ 个观测位，可写成：

$$
sT=b
$$

其中 $s$ 是未知的 19968 位初始状态行向量，$b$ 是按输出顺序拼接的 19968 位观测向量，$T$ 是只由输出位置决定的线性变换矩阵。

无需手工推导 $T$。令状态仅第 $i$ 位为 1、其余位为 0，通过 `setstate()` 注入 CPython 的 `Random`，再执行与题目相同的 9984 次 `getrandbits(2)`；所得输出就是第 $i$ 个基向量经过线性变换后的结果，也就是 $T$ 的第 $i$ 行。遍历全部 19968 个基向量后，在 SageMath 的 $GF(2)$ 上求解 `T.solve_left(B)` 即可恢复完整状态。

该矩阵规模为 $19968\times19968$，构造成本较高，但它只与 Python 版本和采样方式有关，与本轮随机 seed 无关，实战中可以预计算后缓存。

### 从 CPython 状态逆推 seed

恢复状态后还不能直接把 `state[0]` 当 seed。CPython 对整数 seed 使用 `init_genrand(19650218)` 生成固定初态，再经过 `init_by_array()` 的两轮混合；本题 seed 不超过 32 位，所以初始化 key 只有一个 32 位元素。

脚本先用乘数 `1566083941` 逆转第二轮混合在索引 622、623 处的状态，再结合固定初态和第一轮乘数 `1664525` 反推出唯一的 32 位 seed。独立使用多个已知 seed 复测该逆推函数，均能还原原值。

原稿引用的 [MT19937 状态恢复笔记](https://seandictionary.top/mt19937/) 详细讨论了 `getrandbits()` 的截断规则、黑盒构造线性矩阵和 CPython `init_by_array()`；这些必要信息已在上文结合本题参数展开，链接仅保留为来源和延伸阅读。

完整 SageMath 脚本如下：

```sage
__import__('os').environ['TERM'] = 'xterm'
from pwn import *
from random import Random
from tqdm import trange
# context(log_level = 'debug')
# ---------------------------
def construct_a_row(RNG):
    row = []
    for _ in range(19968//2):
        bits = RNG.getrandbits(2)
        row += list(map(int, bin(bits)[2:].zfill(2)))
    return row
T = []
for i in trange(19968):
    state = [0]*624
    temp = "0"*i + "1"*1 + "0"*(19968-1-i)
    for j in range(624):
        state[j] = int(temp[32*j:32*j+32], 2)
    RNG = Random()
    RNG.setstate((3,tuple(state+[624]),None))
    T.append(construct_a_row(RNG))
T = Matrix(GF(2),T)
io = process(['python', 'task.py'])
output = []
for i in range(19968//2):
    io.sendlineafter(b'>> ', b'G')
    output += [int(io.recvline().strip())]
b = [int(j) for i in output for j in bin(i)[2:].zfill(2)]
assert len(b) == 19968
B = vector(GF(2),b)
s = T.solve_left(B)
state = []
for i in range(624):
    state.append(int("".join(list(map(str,s)))[32*i:32*i+32],2))
# ---------------------------
def _int32(x):
    return int(0xFFFFFFFF & x)
def _re_init_by_array_part(index, mt, multiplier):
    return _int32((mt[index] + index) ^^ (mt[index - 1] ^^ mt[index - 1] >> 30) * multiplier)
def _init_genrand(seed, mt):
    mt[0] = seed
    for i in range(1, 624):
        mt[i] = _int32(1812433253 * (mt[i - 1] ^^ mt[i - 1] >> 30) + i)
def re_init_by_array(mt):
    tmp = [_re_init_by_array_part(i, mt[:-1], 1566083941) for i in [622, 623]]
    original_mt = [0] * 624
    _init_genrand(19650218, original_mt)
    predict_seed = _int32(tmp[-1] - _int32((tmp[-2] ^^ (tmp[-2] >> 30)) * 1664525 ^^ original_mt[-1]))
    return predict_seed
mt = state+[624]
seed = re_init_by_array(mt)
print(seed)
io.sendlineafter(b'>> ', b'C')
io.sendlineafter(b'Guess the seed: ', str(seed).encode())
io.interactive()
```

仓库的 `secret.py` 仅含本地默认值 `flag{test}`，PDF 也没有保存比赛实例最终输出，因此无法从离线材料确定真实比赛 flag。脚本提交恢复出的 seed 后，服务端会自行输出当前实例的 flag。

## 方法总结

当 MT19937 只泄露零散或截断位时，应先判断总泄露位数是否达到 19968，并利用生成、旋转、temper 与位截取在 $\mathbb{F}_2$ 上的线性性质建立方程。本题还要区分标准 MT19937 初始化与 CPython `random.Random(seed)` 的 `init_by_array()` 路径；恢复内部状态只是第一步，逆初始化得到原 seed 才能通过最终检查。
