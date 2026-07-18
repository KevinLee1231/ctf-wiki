# week4ParityOracleBase5

## 题目简述

服务端生成 RSA 密文 $c=m^e\bmod n$，并提供解密 oracle，但只返回明文对 5 的余数。目标是通过选择密文逐位恢复 $m$ 的五进制展开。

## 解题过程

令 $x=m/n\in[0,1)$，第 $k$ 次查询发送

$$
c_k=c\cdot(5^e)^k\bmod n.
$$

其解密结果为 $5^km\bmod n$。设

$$
t_k=\left\lfloor5^kx\right\rfloor,
$$

则

$$
5^km\bmod n=5^km-t_kn.
$$

对 5 取模，第一项为零，所以 oracle 返回值 $r_k$ 满足

$$
r_k\equiv-t_kn\pmod5.
$$

由于 $5\nmid n$，可求逆得到

$$
t_k\bmod5\equiv-r_kn^{-1}\pmod5.
$$

而 $t_k-5t_{k-1}$ 正是 $x$ 的第 $k$ 个五进制小数位，其范围只有 $0$ 到 $4$，所以上式直接给出该位。维护已恢复的五进制前缀 `prefix/scale`，即可用纯整数上下界精确定位 $m$，避免原 WP 直接五等分整数区间造成的累计取整误差。

```python
import re
from Crypto.Util.number import long_to_bytes
from pwn import remote

HOST = "challenge.example"
PORT = 0
io = remote(HOST, PORT)

io.sendlineafter(b">", b"1")
n = int(io.recvline().split(b"=")[1])
e = int(io.recvline().split(b"=")[1])
c = int(io.recvline().split(b"=")[1])

def oracle(ciphertext):
    io.sendlineafter(b">", b"3")
    io.sendlineafter(b">", str(ciphertext).encode())
    line = io.recvline().strip()
    return int(line.rsplit(b" ", 1)[1])

factor = pow(5, e, n)
query = c
inverse_n = pow(n, -1, 5)
prefix = 0
scale = 1

while True:
    query = query * factor % n
    remainder = oracle(query)
    digit = (-remainder * inverse_n) % 5

    prefix = prefix * 5 + digit
    scale *= 5

    # prefix/scale <= m/n < (prefix+1)/scale
    lower = (n * prefix + scale - 1) // scale
    upper = (n * (prefix + 1) - 1) // scale
    if lower == upper:
        message = long_to_bytes(lower)
        match = re.search(rb"0xGame\{[^}\r\n]+\}", message)
        if not match:
            raise ValueError("已恢复明文，但未找到 flag")
        print(match.group().decode())
        break

io.close()
```

服务端每次随机生成前 8 个十六进制字符，其中后 4 位只取 `0-7`，固定后缀为 `-6076-8067-8848-taijinshouji}`。原 WP 的

```text
0xGame{31b52022-6076-8067-8848-taijinshouji}
```

是一次动态样例，不是唯一固定 flag。

## 方法总结

模 5 oracle 不只是“五分区间”：它精确泄露 $m/n$ 的下一位五进制数字。用五进制前缀构造有理区间，并用整数公式计算边界，可以消除浮点和截断误差；攻击代价约为 $\lceil\log_5 n\rceil$ 次查询。
