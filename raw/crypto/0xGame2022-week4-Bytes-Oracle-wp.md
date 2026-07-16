# week4Bytes Oracle

## 题目简述

服务端生成一个四素数 RSA 模数，给出公钥 $(n,e)$ 和目标密文 $c=m^e\bmod n$。解密接口允许提交任意密文，并返回对应明文的最低一个字节。目标是利用这个 byte oracle 恢复被随机字节包围的 flag。

## 解题过程

RSA 具有乘法同态。对第 $t$ 轮提交

$$
c_t=c\cdot(256^t)^e\bmod n
$$

解密结果就是

$$
r_t=m\cdot256^t\bmod n=m\cdot256^t-q_tn,
$$

其中 $q_t=\left\lfloor 256^tm/n\right\rfloor$。由于 $t\geq1$ 时 $m\cdot256^t$ 的最低字节为零，所以 oracle 返回的字节满足

$$
r_t\equiv-q_tn\pmod{256}.
$$

RSA 模数 $n$ 是奇数，因此它在模 $256$ 下可逆。枚举 $0\leq k<256$ 并建立 $(-kn)\bmod256\mapsto k$ 的表，就能从 oracle 输出求出 $q_t\bmod256$。

令 $x=m/n$，则 $q_t=\lfloor256^tx\rfloor$。连续获得的 $q_t\bmod256$ 正好是 $x$ 的 256 进制小数位。若前 $t$ 位组成整数 `prefix`，则

$$
\frac{\text{prefix}}{256^t}\leq\frac{m}{n}<
\frac{\text{prefix}+1}{256^t}.
$$

不断收窄该区间，直到其中只剩一个整数 $m$。完整脚本如下：

~~~python
import re
import sys

from Crypto.Util.number import long_to_bytes
from pwn import remote

host = sys.argv[1]
port = int(sys.argv[2])
io = remote(host, port)

io.recvuntil(b"4.Quit")
io.sendlineafter(b"> ", b"1")
io.recvuntil(b"n=")
n = int(io.recvline())
io.recvuntil(b"e=")
e = int(io.recvline())
io.recvuntil(b"c=")
c = int(io.recvline())

# oracle_byte = (-q_t * n) mod 256
inverse_byte = {(-k * (n % 256)) % 256: k for k in range(256)}

prefix = 0
scale = 1
for t in range(1, 10000):
    crafted = c * pow(256, t * e, n) % n

    io.sendlineafter(b"> ", b"3")
    io.sendlineafter(b">", str(crafted).encode())
    io.recvuntil(b": ")
    oracle_byte = int(io.recvline().strip(), 16)

    digit = inverse_byte[oracle_byte]
    prefix = prefix * 256 + digit
    scale *= 256

    lower = (prefix * n + scale - 1) // scale
    upper = ((prefix + 1) * n + scale - 1) // scale - 1
    if lower == upper:
        plaintext = long_to_bytes(lower)
        match = re.search(rb"0xGame\{[^}]+\}", plaintext)
        if match is None:
            raise RuntimeError("已恢复明文，但没有找到 flag 格式")
        print(match.group().decode())
        break
else:
    raise RuntimeError("查询次数不足，区间尚未收敛")

io.close()
~~~

恢复结果为：

~~~text
0xGame{RSA|Byte_0racle~~~}
~~~

## 方法总结

最低字节泄漏并不只暴露原明文的一个字节。借助 RSA 乘法同态，可以把明文乘以 $256^t$ 后反复查询；模约减所产生的商会通过最低字节泄漏出来，从而逐位确定 $m/n$ 的 256 进制展开。防御上必须对解密结果使用安全填充并统一错误行为，不能返回任何与明文有关的局部信息。
