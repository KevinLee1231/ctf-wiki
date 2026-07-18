# week2CN.NO1

## 题目简述

服务端随机生成同一个 90 字节明文 $m$，使用公钥指数 $e=3$ 和三组独立 RSA 模数分别加密。通过四字符 SHA-256 PoW 后会收到 $(n_i,c_i)$，需要恢复 $m$ 并回传。

## 解题过程

三组密文满足

$$
c_i\equiv m^3\pmod{n_i},\qquad i=0,1,2.
$$

模数两两互素，利用中国剩余定理可得到唯一的

$$
x\equiv m^3\pmod N,\qquad N=n_0n_1n_2.
$$

源码中 $m$ 只有 90 字节，而每个 $n_i$ 约 1024 bit，因此 $m^3<N$。这意味着 CRT 的结果不是发生回绕后的剩余类，而就是整数 $m^3$，对其开精确立方根即可。这是 Håstad 低指数广播攻击。

下面的脚本不保留已经失效的历史靶机地址，运行时填写当前题目给出的连接信息：

```python
from hashlib import sha256
from itertools import product
import string
from pwn import remote

HOST = "challenge.example"
PORT = 00000

def solve_pow(suffix, target):
    alphabet = string.ascii_letters + string.digits
    for chars in product(alphabet, repeat=4):
        prefix = "".join(chars)
        if sha256((prefix + suffix).encode()).hexdigest() == target:
            return prefix
    raise ValueError("PoW 无解")

def crt(remainders, moduli):
    modulus = 1
    for n in moduli:
        modulus *= n
    value = 0
    for c, n in zip(remainders, moduli):
        part = modulus // n
        value += c * part * pow(part, -1, n)
    return value % modulus

def exact_cuberoot(value):
    left = 0
    right = 1 << ((value.bit_length() + 2) // 3 + 1)
    while left + 1 < right:
        middle = (left + right) // 2
        if middle**3 <= value:
            left = middle
        else:
            right = middle
    if left**3 != value:
        raise ValueError("CRT 结果不是完全立方")
    return left

io = remote(HOST, PORT)
io.recvuntil(b"sha256(XXXX+")
suffix = io.recvuntil(b")", drop=True).decode()
io.recvuntil(b"== ")
target = io.recvline().strip().decode()
prefix = solve_pow(suffix, target)
io.recvuntil(b"XXXX :")
io.sendline(prefix.encode())

moduli = []
ciphertexts = []
for i in range(3):
    moduli.append(int(io.recvline().split(b"=")[1]))
    ciphertexts.append(int(io.recvline().split(b"=")[1]))

m = exact_cuberoot(crt(ciphertexts, moduli))
io.sendlineafter(b"CN NO.1:", str(m).encode())
print(io.recvall().decode())
```

仓库服务端保存的静态 flag 为：

```text
0xGame{Chinese_remainder_the0rem_is_great}
```

## 方法总结

同一明文在无填充 RSA 中以小指数 $e$ 被至少 $e$ 个互素模数加密时，应优先检查广播攻击。核心条件是 CRT 后的 $m^e$ 小于所有模数乘积，随后必须验证整数开根结果是完全幂。
