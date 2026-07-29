# cryptoGRAPHy 1

## 题目简述

题目实现了一个面向最短路径查询的图加密方案（Graph Encryption Scheme，GES）。服务随机生成一个含 130 个节点、260 条边的无向图，公开完整密钥和某次查询的加密响应，要求连续 50 次还原从源点到终点的最短路径。

GES 的密钥由两部分拼接：

```text
key = key_SKE || key_DES
```

其中 `key_SKE` 用于 AES-CBC 加密路径节点对，`key_DES` 用于派生字典搜索标签。图加密阶段会把到同一终点的最短路径写成链式记录：

```text
(current, destination)
    -> token(next, destination) || AES-CBC((next, destination))
```

服务已经直接给出密钥，因此本题没有要求攻破 AES 或底层字典加密，核心是读懂响应格式并逆向 `GES.search` 的拼接过程。实现对应论文 [Efficient Graph Encryption Scheme for Shortest Path Queries](https://dl.acm.org/doi/10.1145/3433210.3453099) 中的基本构造；下面已经写出本题实际需要的结构，不依赖论文也能完成恢复。

## 解题过程

### 还原响应结构

`GES.search` 从查询 token 开始反复查表。每条记录的前 32 字节是下一跳 token，后半部分是 AES-CBC 密文：

```python
while True:
    value = DES.search(curr, encrypted_db)
    if value == b"":
        break
    curr = value[:32]
    resp += value[32:]
```

路径中的每个节点对先被编码成短字符串，例如 `b"63,72"`，再使用 16 字节 `key_SKE` 加密。CBC 密文后附 16 字节 IV，而短明文经 PKCS#7 填充后只有一个 AES block，所以每条记录固定占 32 字节：

```text
16 字节密文 || 16 字节 IV
```

因此只需把响应每 32 字节切块，用 `key[:16]` 逐块解密：

```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

def decrypt_block(key_ske: bytes, block: bytes) -> str:
    ct, iv = block[:-16], block[-16:]
    cipher = AES.new(key_ske, AES.MODE_CBC, iv)
    return unpad(cipher.decrypt(ct), AES.block_size).decode()

def reveal_path(key: bytes, response: bytes, source: int) -> list[int]:
    key_ske = key[:16]
    path = [source]

    for offset in range(0, len(response), 32):
        pair = decrypt_block(key_ske, response[offset:offset + 32])
        next_vertex, _ = map(int, pair.split(","))
        path.append(next_vertex)

    return path
```

解密结果的顺序已经是沿最短路径从源点走向终点的下一跳序列。将题目给出的源点放在列表开头，再依次取每个节点对的第一个分量即可。

### 交互流程

完整交互只需要保存服务最初打印的密钥，然后重复读取查询和响应：

```python
key = bytes.fromhex(io.recvline_after(b"[*] Key: ").strip().decode())

for _ in range(50):
    u, v = map(int, io.recvline_after(b"50: ").split())
    response = bytes.fromhex(
        io.recvline_after(b"[*] Response: ").strip().decode()
    )
    answer = " ".join(map(str, reveal_path(key, response, u)))
    io.sendlineafter(b"> Original query: ", answer.encode())
```

服务端会用同一密钥自行解密响应并比较路径；连续 50 次正确即可通过。

## 方法总结

- 核心技巧：从 `GES.search` 的链式记录中识别固定长度 AES-CBC 块，使用公开的 `key_SKE` 逐块恢复下一跳。
- 识别信号：题目直接泄露复合密钥，响应长度是 32 的倍数，源码明确把 `token || ciphertext` 写入加密字典；这说明重点是序列化与数据流，而不是攻击底层密码原语。
- 复用要点：分析组合加密方案时应先拆清各子密钥的用途、记录边界和链式指针方向。拿到明文节点对后，还要确认应取哪个分量以及源点是否已经包含在响应中。
