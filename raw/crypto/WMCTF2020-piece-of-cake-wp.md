# piece_of_cake

## 题目简述

本题是类 NTRU 二阶结构与 RSA 组合的交互题，服务端提供 `make_cake()` 和 `eat_cake()` 两个功能。`make_cake()` 中的比特关系允许恢复等价私钥候选 $d_i$，而 `eat_cake()` 需要提交正确的 cake。预期解利用多组 RSA 实例共享小私钥指数的条件：先多次调用 `make_cake()` 得到 14 组 $N_i,d_i$，用 LLL 恢复公共 $e$，再在 `eat_cake()` 中枚举候选 $d$ 分解 $N$ 并恢复 cake。非预期解则利用已知 `cake.bit_length()` 直接规约出等价 $f,g$，求出 `cake % g`。

## 解题过程

**非预期** ：

对于 `eat_cake()`，已知 `cake.bit_length()`，那么直接规约出 **等价** $f,g$，获取 `cake % g` 即可。

**预期** ：

`task.py` 中有两个函数，`make_cake()` 和 `eat_cake()`，两函数中都有类 NTRU 二阶和 RSA 加密，但细看会发现有区别：`make_cake()` 的比特关系使得 cake 能被求出，而 `eat_cake()` 不行；其最后输出的 $ct = cake^k \bmod N$ 的指数也有所不同。通过开头的 `assert` 可以感知到本题与格基规约相关。综合以上几点我们能够推断出做题流程：我们通过不断 `make_cake()` 拿到 $d_i$ 后使用 LLL 求出 $e$，再 `eat_cake()`，通过 $\gcd(2^{ed-1}-1 \bmod N, N)$ 分解 $N$ 后拿到 flag。

很明显这里通过 $f,g,q$ 的关系可以看出，使用 Wiener's attack 是可以求出包含 $f$（即 $d$）的一个集合的。在 `make_cake()` 中，拿到 cake 后通过 `pow(cake, d, N)` 可以确定这个 $d$。根据 $e$ 和 $N$ 的关系，获取 14 组 $d$ 后则可构造论文 [Lattice Based Attack on Common Private Exponent RSA](https://www.ijcsi.org/papers/IJCSI-9-2-1-311-314.pdf) 中的格子拿到 $e$。该论文的关键条件是：多个 RSA 实例使用相同的小私钥指数，且模数规模相近；每个实例满足 $ed_i = 1 + k_i\varphi(N_i)$ 这类 RSA key equation，可把这些关系放进格中，用 LLL 找到短向量恢复公共指数信息。本题把这个模型转化为“多次 `make_cake()` 得到同一公共 $e$ 下的多个 $d_i,N_i$”，因此可以用同类格攻击恢复 $e$。而后以 `eat_cake()` 中所有 $d$ 的候选对 $N$ 进行分解后拿到 cake，即可获取 flag。

exp 如下。脚本中的 LLL 部分使用了 `matrix(ZZ, ...)`，需要在 Sage 环境或提供 Sage 矩阵对象的 Python 环境中运行；远程地址使用占位符替换为实际题目服务。

```python
from Crypto.Util.number import *
from gmpy2 import invert, sqrt, gcd
from string import ascii_letters, digits
from hashlib import sha256
from pwn import *
from pwnlib.util.iters import mbruteforce
context.log_level = 'debug'
CHALLENGE_HOST = '<challenge-host>'
CHALLENGE_PORT = 0  # replace with the challenge port
def proof_of_work(p):
    p.recvuntil("XXX+")
    suffix = p.recv(17).decode("utf8")
    p.recvuntil("== ")
    cipher = p.recvline().strip().decode("utf8")
    proof = mbruteforce(lambda x: sha256((x + suffix).encode()).hexdigest() ==
                        cipher, ascii_letters + digits, length=3, method='fixed')
    p.sendlineafter("Give me XXX:", proof)
def rational_to_contfrac(x,y):
    a = x // y
    pquotients = [a]
    while a * y != x:
        x,y = y, x - a * y
        a = x // y
        pquotients.append(a)
    return pquotients
def convergents_from_contfrac(frac):
    convs = []
    for i in range(len(frac)):
        convs.append(contfrac_to_rational(frac[0:i]))
    return convs
def contfrac_to_rational (frac):
    if len(frac) == 0:
        return (0, 1)
    num = frac[-1]
    denom = 1
    for _ in range(-2, -len(frac) - 1, -1):
        num, denom = frac[_] * num + denom, num
    return (num, denom)
def get_d(q, h, c, N, ct):
    frac = rational_to_contfrac(h, q)
    convergents = convergents_from_contfrac(frac)
    for (i, f) in convergents:
        g = abs(h * f - i * q)
        try:
            cake = (c * f % q * invert(f, g) % g)
            if pow(cake, f, N) == ct:
                return f
        except:
            continue
def make_cake():
    Ns = []
    ds = []
    num = 14
    for i in range(num):
        print("Getting the {} / {} d".format(str(i + 1), str(num)))
        io = remote(CHALLENGE_HOST, CHALLENGE_PORT)
        proof_of_work(io)
        io.sendafter('What\'s your choice?\n', '2\n')
        io.recvline()
        q, h, c = [int(x) for x in io.recvline(keepends = False).decode().split(' ')]
        N = int(io.recvline(keepends = False))
        Ns.append(N)
        ds.append(get_d(q, h, c, N, int(io.recvline(keepends = False))))
        io.close()
    M = 1
    for i in range(num):
        if int(sqrt(Ns[i])) > M:
            M = int(sqrt(Ns[i]))
    B = matrix(ZZ, num + 1, num + 1)
    B[0, 0] = M
    for i in range(1, num + 1):
        B[0, i] = ds[i - 1]
    for i in range(1, num + 1):
        B[i, i] = -Ns[i - 1]
    return abs(B.LLL()[0, 0] // M)
def get_ans(q,h,e, N, ct):
    frac = rational_to_contfrac(h, q)
    convergents = convergents_from_contfrac(frac)
    for (i, d) in convergents:
        p = gcd(int(pow(2, e * d - 1,N) - 1), N)
        if p > 1 and p < N:
            return pow(ct, invert(0x10001, (p - 1) * (N//p - 1)), N)
def eat_cake(e):
    io = remote(CHALLENGE_HOST, CHALLENGE_PORT)
    proof_of_work(io)
    io.sendafter('What\'s your choice?\n', '1\n')
    io.recvline()
    q, h, c = [int(x) for x in io.recvline(keepends = False).decode().split(' ')]
    N = int(io.recvline(keepends = False))
    ct = int(io.recvline(keepends = False))
    ans = get_ans(q ,h , e, N, ct)
    io.recvuntil("Give me your cake:")
    io.sendline(str(ans))
    io.interactive()
    io.close()
e = make_cake()
eat_cake(e)
```

## 方法总结

本题的核心识别信号是“类 NTRU 关系给出小私钥候选，同时多组 RSA 实例共享指数关系”。先用连分数/Wiener 思路从 $h/q$ 中枚举 $d$，再用 `make_cake()` 的可验证输出筛掉错误候选；拿到多组 $N_i,d_i$ 后构造公共私钥指数格并 LLL 恢复 $e$。最后利用 $ed \equiv 1 \pmod{\varphi(N)}$ 推出的 $2^{ed-1} \equiv 1 \pmod p$，通过 $\gcd(2^{ed-1}-1,N)$ 分解模数并解出 cake。
