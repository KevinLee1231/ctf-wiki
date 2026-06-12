# d3fnv

## 题目简述

本题要求在 FNV 类哈希的 key 未知时恢复可用于伪造哈希的 key。题目灵感来自 Hellman 写的“NSU CRYPTO 2017 第 4 题：FNV2 Hash”，其关键思想是：FNV 哈希并非只能当作黑盒迭代函数看待，也可以把单次哈希视为在未知 key 上求值的小系数多项式。只要能拿到足够多组 token hash，就可以利用小系数结构和格约化恢复与 key 幂向量相关的信息。

## 解题过程

正如 Hellman 解决 FNV2 Hash 时所作的那样，我们需要将 `fnv_hash(value: str)` 视为未知小系数多项式的单次求值操作：

$$𝑓= 128𝑐0 ⋅k𝑛+ 𝑥0 ⋅𝑘𝑛−1 + ⋯+ 𝑥𝑛−1 mod 𝑝$$

`c0` 是 `value` 的第一个字符，而 `xi` 则大多采样自 $[−256, 256]$。因为 `xi` 和 `c0` 实际都非常小，我们可以尝试使用 Nguyen-Stern 算法，或者所谓的正交格方法，来恢复矩阵 $𝑋= 𝑈⋅(𝑐𝑖,0, 𝑥𝑖,0, ⋯, 𝑥𝑖,𝑛−1)𝑖𝑇$。这个矩阵实际上由原系数矩阵经过 LLL 算法规约得到。想要通过它直接恢复原始系数矩阵显然很困难。然而，在实践中我们发现 $𝑋𝑖± 𝑋𝑗$ 往往等于原系数矩阵的第一个和第二个向量（或者仅相差一个负号）。看起来格基规约算法会以高概率将 $𝑣0, 𝑣1$ 拆分成二范数更小的 $𝑋𝑖, 𝑋𝑗$。我们聚焦于如下线性方程：

$$(128 ⋅𝑘𝑛, 𝑘𝑛−1, ⋯, 𝑘, 1) ⋅C = (𝑓0, 𝑓1, ⋯, 𝑓𝑚−1).$$

如果尝试将上述方程中的系数矩阵 `C` 替换为 `X` 并执行一次 `solve_left()`，可以得到什么？实验表明解向量 $𝑘′$ 和原始的 key 向量存在非平凡交集，因此通过暴力枚举恢复 key 的值是可行的。以上现象初看可能有些奇怪，我们需要重新回顾 FNV 哈希函数的结构，以得到一种更容易理解的解法。首先假设矩阵 `X` 的前两个行向量满足如下方程：

$$𝑥⃗1 + 𝑥⃗2 = 𝑐⃗0, 𝑥⃗1 −𝑥⃗2 = 𝑐⃗1$$

fnv 哈希函数的第一步操作如下：

$$(128𝑐𝑖,0 ⋅𝑘 mod 𝑝) ⊕𝑐𝑖,1$$

我们将上式中的异或运算等效视作加上或减去一个小整数。实际上LLL 算法能够更加精确帮助我们表达这一点。利用X 的 $𝑥⃗0, 𝑥⃗1$，我们可以将fnv 哈希函数的第一步改写为：

$$(128𝑘⋅(𝑥𝑖,0 + 𝑥𝑖,1) mod 𝑝) + (𝑥𝑖,0 −𝑥𝑖,1) = 𝑥𝑖,0 ⋅(128𝑘+ 1) + 𝑥𝑖,1 ⋅(128𝑘−1)$$

从 FNV 哈希函数具体结构和 LLL 算法出发，我们可以找到上述实验现象的一种更好的解释，并得到如下求解脚本：

```python
#!/usr/bin/env python3
from sage.all import *
from Crypto.Util.number import long_to_bytes, bytes_to_long
from pwn import remote, process, context
from subprocess import check_output
from re import findall
def flatter(M):
    z = "[[" + "]\n[".join(" ".join(map(str, row)) for row in M) + "]]"
    ret = check_output(["flatter"], input=z.encode())
    return matrix(M.nrows(), M.ncols(), map(int, findall(b"-?\\d+", ret)))
# io = process(['python3', 'task.py'])
io = remote('34.150.83.54', 30510)
# context(log_level = 'debug')
io.recvuntil(b'option >')
io.sendline(b'G')
p = eval(io.recvline().strip(b'p = '))
hlst = []
for i in range(64):
    io.recvuntil(b'option >')
    io.sendline(b'H')
    hashvalue0 = io.recvline()
    hlst.append(eval(hashvalue0.strip(b'Token Hash: ')) ^ 32)
n = 32
m = 64
M = block_matrix(ZZ,[
    [p, 0],
    [matrix(m,1,hlst), 1]
])
M[:,0] *= p + 1
v = flatter(M)
vi = v[:m-(n+1),1:].T
B = block_matrix(ZZ,[
    [p,0],
    [vi,1]
])
B[:,:m-(n+1)] *= 2**32
res = flatter(B)
X = res[:n+1,m-(n+1):]
sol = list(X.change_ring(GF(p)).solve_left(vector(hlst)))
def fnv_hash_(key:int, value:str):
    length = len(value)
    x = (ord(value[0]) << 7) % p
    for c in value:
        x = ((key * x) % p) ^ ord(c)
    x ^= length
    return x
PR = GF(p)["x"]
x = PR.gens()[0]
exit_flag_i = False
exit_flag_j = False
key_pos = []
for i in [-1,1]:
    for j in [-1,1]:
        fpoly = i*128*x**n + j*x**(n-1) - sol[0]
        rr = fpoly.roots()
        if rr != []:
            for r in rr:
                r0 = int(r[0])
                keys = [r0**i % p for i in range(n+1)]
                u = set(sol).intersection(set(keys))
                if len(list(u)) > 1:
                    key = r0
                    exit_flag_i = True
                    exit_flag_j = True
                    key_pos.append(r0)
                    break
        if exit_flag_j:
            break
    if exit_flag_i:
        break
k0,k1 = key_pos
assert (k0 + k1) == p
for _ in range(2):
    io.recvuntil(b'option >')
    io.sendline(b'F')
    token = io.recvline().strip(b'Here is a random token x:').strip(b'\n').decode()
    h0 = fnv_hash_(key_pos[_], token)
    io.recvuntil(b'Could you tell the value of H4sh(x)? ')
    io.sendline(str(h0).encode())
    flags = io.recvline().strip().decode()
    if flags == 'Congratulations! Here is your flag:':
        flag = io.recvline()
        print(flag.strip(b'\n').decode())
```

## 方法总结

- 核心技巧：把 FNV 哈希改写成未知 key 的小系数多项式求值问题，再用 Nguyen-Stern / 正交格思路恢复与系数矩阵相关的短向量。
- 识别信号：哈希迭代中同时存在乘 key、异或小字符值、模大素数等结构时，不要只按随机哈希黑盒处理，应尝试把异或近似为小扰动并建立格模型。
- 复用要点：LLL 得到的矩阵不一定直接等于原始系数矩阵，但行向量的和差可能暴露原矩阵的关键向量；拿到候选 key 后仍需用服务端 `F` 查询验证哈希结果。
