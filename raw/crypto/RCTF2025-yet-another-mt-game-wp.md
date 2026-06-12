# yet another MT game

## 题目简述

题目依赖 SageMath `random_matrix(Zmod(mod))` 的随机性。不同 modulus 会走 GMP MT 或 Python MT；本题需要从 oracle 输出中恢复 GMP MT 状态，再反推 Sage/GMP 的 seed 以恢复 64 字节 secret。

## 解题过程

### 关键观察

题目依赖 SageMath `random_matrix(Zmod(mod))` 的随机性。

### 求解步骤

在 SageMath 中，random_matrix(Zmod(mod))  调用 randomize  方法生成随机矩阵（http
s://github.com/sagemath/sage/blob/develop/src/sage/matrix/special.py），它调用了两种不同
的 MT：gmp 的 MT 和 Python 的 MT。
在
 时，SageMath 生成
 类型的矩阵（h
ttps://github.com/sagemath/sage/blob/develop/src/sage/matrix/matrix_modn_dense_float.
pyx
）；在
 时，SageMath 生成
mod < 256
Matrix_modn_dense_float
mod < 94906266
 类型的矩阵（
https://github.com/sagemath/sage/blob/develop/src/sage/matrix/matrix_modn_dense_dou
ble.pyx）。这两种矩阵都符合模板类型
（
https://github.com/sagemath/sage/blob/develop/src/sage/matrix/matrix_modn_dense_tem
plate.pxi），它的
 方法将每个元素赋值为
 。
c_random()  定义在 https://github.com/sagemath/sage/blob/develop/src/sage/misc/ra
ndstate.pyx，调用 gmp_urandomb_ui  返回 31 位随机整数，随机数来自 gmp 内置的
MT。
在
 较大时，SageMath 生成
 类型的矩阵，它的
 方法（
Matrix_modn_dense_double
Matrix_modn_dense_template
randomize
current_randstate().c_random() % mod
mod
Matrix_generic_dense
randomize
https://github.com/sagemath/sage/blob/develop/src/sage/matrix/matrix2.pyx）将每个元素
赋值为
，其中
 是
。
 的实现（
https://github.com/sagemath/sage/blob/develop/src/sage/rings/finite_rings/integer_mod_r
ing.py）调用了
 。
random  是 sage.misc.prandom  模块（https://github.com/sagemath/sage/blob/deve
lop/src/sage/misc/prandom.py），也就是 current_randstate().python_random()
，和 Python 内置的 MT 一样。
接下来看 seed  相关的源码（https://github.com/sagemath/sage/blob/develop/src/sage/misc/
randstate.pyx），set_random_seed  用 gmp_randseed  设置了 gmp 的种子；而 pyrand 直
到需要调用时才被初始化，它的初始化 rand.seed(int(ZZ.random_element(1<<128)))  调
用了 ZZ.random_element （https://github.com/sagemath/sage/blob/develop/src/sage/rings/
integer_ring.pyx），也就是 mpz_urandomm(value, rstate.gmp_state, 1<<128)  。所
以，pyrand 的种子是 gmp 的 MT 的输出。
在这题中，secret = os.urandom(64)  是 512 bit 的未知数，即使知道了 pyrand 的种子，也
无法恢复 secret  ，故考虑直接破译 gmp 的 MT。接下来看 gmp 的源码（从 https://gmplib.or
g/ 下载）。gmp 的 MT 和 Python 的 MT 几乎一致，除了初始化的部分：在 rand/randmts.c
可以看到以下公式：
R.random_element()
R
Zmod(mod)
random_element
random.randint(0, mod - 1)
然后 seed2（19937 位整数）被用来初始化 624 个 32 位整数（其中下标 0 的数的低 31 位固定为
0），这些数再旋转 WARM_UP  轮之后就是初始状态。
为了解决这题，只需要请求 19937 个 bit，解线性方程组恢复 gmp random 的初始状态（也就是
seed2 ），再从 seed2  反推 seed  。最后一步只需要解一个 RSA，因为 e  和 phi = p-1
不互质（gcd(1074888996, 2^19937-20024) = 12 ），所以只能先求出部分的逆，再开 12
次根。
seed1 = seed mod (2^19937-20027) + 2
seed2 = (seed1^1074888996) mod (2^19937-20023)
from pwn import process, remote
from ast import literal_eval
import gmpy2

# io = process(["sage", "mtv1.py"])
io = remote("<challenge-host>", 42101)
io.sendline(str(2).encode() + b" 1 19937")
io.recvuntil(b"output:")
output = literal_eval(io.recvline().decode())

from gf2bv import LinearSystem
from gf2bv.crypto.mt import MT19937
from tqdm import tqdm

lin = LinearSystem([32] * 624)
mt = lin.gens()
rng = MT19937(mt)
zeros = [mt[0] & 0x7FFFFFFF]
for i in range(2000 - 624):
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

### 参考链接补充

这些 Sage/GMP 链接支撑的是“随机矩阵输出可以反推 GMP randstate”这一判断：

- `random_matrix()` 的默认 `algorithm` 是 `randomize`，因此题目调用随机矩阵时会进入矩阵类自己的 `randomize()` 实现。
- `matrix_modn_dense_template.pxi` 中，模数较小的 dense modn 矩阵在 `density == 1` 时逐项执行 `self._entries[i] = current_randstate().c_random() % p`，所以矩阵元素直接泄露 GMP randstate 的 31-bit 输出取模。
- `randstate.pyx` 中 `c_random()` 返回 `gmp_urandomb_ui(self.gmp_state, 31)`，`set_random_seed()` 则用 `gmp_randseed(self.gmp_state, seed)` 初始化 GMP 状态。
- Sage 的 `python_random()` 若未显式传 seed，会执行 `rand.seed(int(ZZ.random_element(1 << 128)))`；而 `ZZ.random_element()` 在有上界时走 `mpz_urandomm(value, rstate.gmp_state, ...)`。这说明 Python `random.Random` 的 128-bit seed 也是从同一个 GMP 状态抽出来的。
- 因此本题不能只恢复 Python MT seed，因为 `secret = os.urandom(64)` 与 Python seed 无直接关系；必须从矩阵元素恢复 GMP MT 状态，再按 GMP `randmts.c` 的初始化/输出关系反推可用于解密的状态。

### PDF 外链

- <https://github.com/sagemath/sage/blob/develop/src/sage/matrix/special.py>
- <https://github.com/sagemath/sage/blob/develop/src/sage/matrix/matrix_modn_dense_double.pyx>
- <https://github.com/sagemath/sage/blob/develop/src/sage/matrix/matrix_modn_dense_template.pxi>
- <https://github.com/sagemath/sage/blob/develop/src/sage/misc/randstate.pyx>
- <https://github.com/sagemath/sage/blob/develop/src/sage/matrix/matrix2.pyx>
- <https://github.com/sagemath/sage/blob/develop/src/sage/rings/finite_rings/integer_mod_ring.py>
- <https://github.com/sagemath/sage/blob/develop/src/sage/misc/prandom.py>
- <https://github.com/sagemath/sage/blob/develop/src/sage/rings/integer_ring.pyx>
- <https://gmplib.org/>

### 跨页补回：seed 反推收尾

seed |= 1 << 19936

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

## 方法总结

- 核心技巧：从 Sage/GMP MT 输出建立 GF(2) 线性系统恢复状态。
- 识别信号：Sage `random_matrix`、`Zmod(mod)`、不同 modulus 触发不同随机源。
- 复用要点：先确认具体矩阵类型和 randomize 实现，再处理 seed 初始化的非线性反推。
