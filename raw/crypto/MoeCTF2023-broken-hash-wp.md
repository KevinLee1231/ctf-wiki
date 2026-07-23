# broken_hash

## 题目简述

服务端每轮给出 256 字节随机串 $b$，要求在 60 秒内提交另一个字节串 $d$，满足 $d\ne b$ 且自定义散列值相同。输入、输出都使用 Base64；散列先按 16 字节小端序分块，再以四叉树结构递归压缩。

## 解题过程

叶子压缩函数一次处理四个 128 位块：

```python
def _block_hash(a, b, c, d):
    x, y, z, w = F(a, b, c), F(b, c, d), F(c, d, a), F(d, a, b)
    return (a ^ b ^ c ^ d ^ x ^ y ^ z ^ w) ^ MASK
```

四个输入及四个 $F$ 项最后都只通过异或汇总。$F$ 的参数又按环相邻关系选取，因此 `_block_hash` 对四元组的旋转和镜像不变；特别有

$$
H(a,b,c,d)=H(d,c,b,a).
$$

题目给出的 256 字节恰好是 16 个块。递归的第一层把它分成四组，每组四个块；只需分别反转每组内部的块序，四个叶子散列均不变，根节点收到的四个值及顺序也完全不变。随机输入几乎不可能在四组内都恰好回文，所以变换后还能满足 $d\ne b$。

```python
from base64 import b64decode, b64encode

def collide(encoded: str) -> str:
    data = b64decode(encoded)
    assert len(data) == 256

    blocks = [data[i:i + 16] for i in range(0, len(data), 16)]
    forged = b"".join(
        block
        for group in range(0, 16, 4)
        for block in reversed(blocks[group:group + 4])
    )
    assert forged != data
    return b64encode(forged).decode()
```

完整的交互版本如下：

```python
from argparse import ArgumentParser
from base64 import b64decode, b64encode
from hashlib import md5
from itertools import product
from string import ascii_letters

from pwn import remote

parser = ArgumentParser()
parser.add_argument("host")
parser.add_argument("port", type=int)
args = parser.parse_args()

io = remote(args.host, args.port)

io.recvuntil(b"md5(XXXX+")
suffix = io.recvuntil(b")", drop=True)
io.recvuntil(b" == ")
target = io.recvline().strip().decode()

answer = next(
    bytes(chars)
    for chars in product(ascii_letters.encode(), repeat=4)
    if md5(bytes(chars) + suffix).hexdigest() == target
)
io.sendlineafter(b"Give me XXXX: ", answer)

io.recvuntil(b"this is a random bytes: ")
data = b64decode(io.recvline().strip())

blocks = [data[i:i + 16] for i in range(0, 256, 16)]
forged_blocks = []
for group in range(0, 16, 4):
    forged_blocks.extend(reversed(blocks[group:group + 4]))
forged = b"".join(forged_blocks)

assert forged != data
io.sendlineafter(
    b"give me another bytes with the same hash: ",
    b64encode(forged),
)
io.interactive()
```

服务端验证通过后返回：

```text
moectf{a_hash_FUNCtioN_With_sYMMEtRY_is_vERY_vUlNERa6lE_3iiA0JiuP0DxuuP}
```

## 方法总结

自定义散列不仅要检查单轮扩散，还要检查压缩函数对输入置换是否存在不变量。本题的关键不是寻找数值碰撞，而是利用四元组的二面体对称性构造结构碰撞；递归树中只修改叶子内部顺序，则碰撞会自然传递到根节点。
