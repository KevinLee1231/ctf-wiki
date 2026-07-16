# ECC-DH

## 题目简述

服务端扮演 ECDH 中的 Alice，发送曲线基点 $G$ 和公钥 $A=aG$。客户端可以自行选择 Bob 私钥 $b$，提交公钥横坐标，服务端用共享点 $aB$ 的横坐标派生 AES 密钥并加密 flag。题目不需要破解离散对数，只需正确完成椭圆曲线 Diffie-Hellman 协议。

## 解题过程

客户端选择任意 $1\le b<p$，计算：

$$
B=bG
$$

$$
S=bA=b(aG)=a(bG)=aB
$$

服务端只接收 $B$ 的横坐标，并通过曲线方程恢复一个纵坐标。即使恢复的是 $-B$，共享点也只会从 $S$ 变成 $-S$，两者横坐标相同，不影响 `MD5(str(S.x))`。

下面的脚本包含 PoW、点解析、密钥交换和 AES 解密。`Util.py` 使用题目附件中的实现：

```python
from Crypto.Cipher import AES
from hashlib import md5, sha256
from itertools import product
from pwn import remote
from random import randint
from string import ascii_letters, digits

from Util import Curve, Point

HOST = "TARGET"
PORT = 10004

def solve_pow(io):
    io.recvuntil(b"sha256(XXXX+")
    suffix = io.recvuntil(b")", drop=True).decode()
    io.recvuntil(b"== ")
    expected = io.recvline().strip().decode()

    alphabet = ascii_letters + digits
    for chars in product(alphabet, repeat=4):
        prefix = "".join(chars)
        if sha256((prefix + suffix).encode()).hexdigest() == expected:
            io.sendlineafter(b"XXXX: ", prefix.encode())
            return
    raise RuntimeError("PoW 无解")

def recv_point(io, marker, curve):
    io.recvuntil(marker + b"(")
    x, y = map(int, io.recvuntil(b")", drop=True).split(b","))
    return Point(x, y, curve)

io = remote(HOST, PORT)
solve_pow(io)

p = 25321837821840919771
curve = Curve(
    10809567548006703521,
    9981694937346749887,
    p,
)

G = recv_point(io, b"[+] Share G : ", curve)
A = recv_point(io, b"[+] Alice_PubKey : ", curve)

bob_secret = randint(1, p - 1)
B = bob_secret * G
shared = bob_secret * A

io.sendlineafter(b"> ", str(B.x).encode())
io.recvuntil(b"[+] Alice tell Bob : ")
ciphertext = bytes.fromhex(io.recvline().strip().decode())

aes_key = md5(str(shared.x).encode()).digest()
flag = AES.new(aes_key, AES.MODE_ECB).decrypt(ciphertext).rstrip(b"\x00")
print(flag.decode())
io.close()
```

得到：

```text
0xGame{71234da9-baf8-406e-9cc7-d08ceedea945}
```

## 方法总结

ECDH 的目标是让通信双方在不发送私钥的情况下得到同一个共享点。本题给客户端自由选择 Bob 私钥和公钥，因此无需攻击曲线；关键是识别 $bA=aB$ 的交换律关系，并严格复现服务端从共享点横坐标到 AES 密钥的派生过程。
