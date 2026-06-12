# yet another shuffled MT game

## 题目简述

本题在上一题 GMP MT 输出恢复外增加了 Python `random.shuffle`。解法先从少量 32-bit 输出恢复 Python MT 的 128-bit seed，逆置乱输出序列，再回到 GMP MT 线性恢复和 seed 反推。

## 解题过程

### 关键观察

本题在上一题 GMP MT 输出恢复外增加了 Python `random.shuffle`。

### 求解步骤

这题相比前一题加入了 shuffle 机制，shuffle 调用的是 pyrand。
考虑用第一轮的结果恢复 pyrand 的种子；实测只需要不到 300 个 32 位输出就能恢复 128 位的整
数种子，远小于题目给的 16400 位。
有了 pyrand 的状态，就可以逆向 shuffle。此时问题转化为了前一题的 gmp random 恢复。
需要注意，在生成 pyrand 种子时，它额外调用了几次 gmp random 的函数。
def saveList(lst, filename):
    try:
        with open(filename, 'w') as file:
            json.dump(lst, file)
    except IOError:
        print("An error occurred while saving the list.")


def loadList(filename):
    try:
        with open(filename, 'r') as file:
            lst = json.load(file)
        return lst
    except IOError:
        print("An error occurred while loading the list.")
        return []

from pwn import process, remote
from ast import literal_eval
from magic import crackme
import random
import gmpy2

# io = process(["sage", "mtv2.py"])
io = remote("<challenge-host>", 42102)

io.sendline(str(2**32 - 2).encode() + b" 1 300")
io.recvuntil(b"output:")
output = literal_eval(io.recvline().decode())
seed = crackme(output)[0]
r = random.Random(seed)
for i in range(300):
    assert r.getrandbits(32) == output[i]
print(f"[+] {seed = }")

io.sendline(str(2).encode() + b" 1 20000")
io.recvuntil(b"output:")
output = literal_eval(io.recvline().decode())

n = 20000
indices = list(range(n))
r.shuffle(indices)
real_output = [None for i in range(n)]
for i in range(n):
    real_output[indices[i]] = output[i]
output = real_output

io.sendline(str(2).encode() + b" 1 1")

from gf2bv import LinearSystem
from gf2bv.crypto.mt import MT19937
from tqdm import tqdm

lin = LinearSystem([32] * 624)
mt = lin.gens()
rng = MT19937(mt)
zeros = [mt[0] & 0x7FFFFFFF]
for i in range(2000 - 624):
    rng.getrandbits(32)

rng.getrandbits(128)
rng.getrandbits(32)

for i in tqdm(range(len(output))):
    zeros.append((rng.getrandbits(32) & 1) ^ output[i])

print("COMPUTING INVERSE...")
P = 2**19937 - 20023
I = pow(1074888996 // 12, -1, (P - 1) // 12)
print("COMPUTED INVERSE...")

for sol in lin.solve_all(zeros):
    seed = 0
    for i in sol[1:][::-1]:
        seed = (seed << 32) | i
    if sol[0] == 0x80000000:
        seed |= 1 << 19936

对于恢复 pyrand 种子的部分，我直接调用了上周 Infobahn CTF ’25 的 crypto 出题人的 repo（htt
ps://github.com/Aeren1564/CTF_Library/blob/master/CTF_Library/Cryptography/MersenneTw
ister/python_random_breaker.py），里面有不少 MT 相关的妙妙小工具。
（Infobahn 的那题要求恢复 19936 位的整数种子，这题因为种子较短，不需要所有 624 个输
出）
    print("COMPUTING POWER...")
    seed = gmpy2.powmod(seed, I, P)
    print("COMPUTED POWER...")
    seed, ok = gmpy2.iroot(seed, 12)

    if ok:
        seed -= 2
        print(hex(seed))
        io.sendlineafter(b"secret", int(seed).to_bytes(64,
"big").hex().encode())
        io.interactive()
from python_random_breaker import python_random_breaker

import random
def crackme(outputs):
    breaker = python_random_breaker()
    bit_len = 128
    is_exact = 1
    indices = [i for i in
breaker.get_required_output_indices_for_integer_seed_recovery(bit_len,
is_exact)]
    cur_outputs = [outputs[i] for i in indices]
    # print(indices)
    return (breaker.recover_all_integer_seeds_from_few_outputs(bit_len,
cur_outputs, True))

if __name__ == "__main__":
    seed = random.getrandbits(128)
    rng = random.Random(seed)
    outputs = [rng.getrandbits(32) for i in range(300)]
    print(seed)
    print(crackme(outputs))

### 参考链接补充

参考库用于处理“Python `random` 的少量输出恢复 seed”这一子问题。本题的置乱层不是 GMP 直接输出，而是先用一个 128-bit Python `random.Random(seed)` 对输出序列 `shuffle`，所以解法分两步：

- 先调用 `get_required_output_indices_for_integer_seed_recovery(bit_len, is_exact)` 取出恢复 128-bit seed 所需的输出下标，而不是把全部 300 个输出都纳入搜索。
- 再把这些位置的 32-bit 输出传给 `recover_all_integer_seeds_from_few_outputs(bit_len, cur_outputs, True)`，枚举/恢复候选 Python seed。
- 恢复出 Python seed 后，重新生成同一轮 `shuffle` 置换，把观测到的 GMP 输出恢复成真实顺序。
- 后续线性恢复与 `yet another MT game` 相同，继续利用 GMP MT 的输出关系求回底层状态。

### PDF 外链

- <https://github.com/Aeren1564/CTF_Library/blob/master/CTF_Library/Cryptography/MersenneTwister/python_random_breaker.py>

## 方法总结

- 核心技巧：先破 Python MT shuffle，再破底层 GMP MT。
- 识别信号：输出被 `random.shuffle` 置乱，但 shuffle seed 位数较短。
- 复用要点：恢复置乱源状态后把输出还原为真实顺序，再复用未置乱版本的线性恢复。
