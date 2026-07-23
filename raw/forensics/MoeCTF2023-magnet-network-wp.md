# magnet_network

## 题目简述

附件是一个 BitTorrent 种子和提示。flag 中间部分长度为 24，只使用 `0-9a-zA-Z@!_?`；完整 flag 的 SHA-256 为 `de5d94f22a9b8eab09779102a0fcc9c566880f7807d359da6f27723f3b881584`。目标不是下载种子内容，而是从 torrent 元数据中的 piece 哈希反推六个 4 字节片段。

## 解题过程

`.torrent` 使用 bencode 编码。解析 `info` 字典可见 `piece length` 为 16384，文件顺序是：

```text
1, .pad/0, 3, .pad/1, 5, .pad/2,
2, .pad/3, 6, .pad/4, 4
```

文件 `1、3、5、2、6、4` 各长 4 字节；五个 `.pad` 文件各长 16380 字节。BitTorrent 按拼接后的文件流切 piece，因此前五个 piece 都是“4 字节内容 + 16380 个零字节”，最后一个 piece 只有文件 `4` 的 4 字节内容。

不需要复制庞大的第三方解析器，下面这个最小 bencode 解码器已足够读取本题元数据：

```python
from pathlib import Path

def bdecode(data, pos=0):
    token = data[pos:pos + 1]
    if token == b"i":
        end = data.index(b"e", pos)
        return int(data[pos + 1:end]), end + 1
    if token == b"l":
        result, pos = [], pos + 1
        while data[pos:pos + 1] != b"e":
            value, pos = bdecode(data, pos)
            result.append(value)
        return result, pos + 1
    if token == b"d":
        result, pos = {}, pos + 1
        while data[pos:pos + 1] != b"e":
            key, pos = bdecode(data, pos)
            value, pos = bdecode(data, pos)
            result[key] = value
        return result, pos + 1

    colon = data.index(b":", pos)
    length = int(data[pos:colon])
    start = colon + 1
    return data[start:start + length], start + length

raw = Path("segments.torrent").read_bytes()
meta, end = bdecode(raw)
assert end == len(raw)

info = meta[b"info"]
pieces = [info[b"pieces"][i:i + 20]
          for i in range(0, len(info[b"pieces"]), 20)]
print(info[b"piece length"])
print([entry[b"path"] for entry in info[b"files"]])
print([piece.hex() for piece in pieces])
```

六个 SHA-1 目标依次为：

```text
3acad96ee53545f8ad3c419c5b1965bea234cc6a
b339e2547e1bb5c1e56c2aceab1f7cf5af93d13d
65131dba750fdcb715e3235107041de9c79671ae
d3ba653b9d29544b623902aa07074492793c7ad4
4dd6e892d925a7411cb1677b8b5d8fec6d9308ac
cc83047e9ae392e02b5044283c89f16dc26f328e
```

每段只有四个字符，可以在提示字符集中枚举。前五个目标计算带零填充的 SHA-1，最后一个计算原始四字节的 SHA-1：

```python
from hashlib import sha1, sha256
from itertools import product

alphabet = b"0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ@!_?"
padded_targets = {piece: index for index, piece in enumerate(pieces[:5])}
last_target = pieces[5]
found = {}
zero_pad = b"\x00" * 16380

for chars in product(alphabet, repeat=4):
    candidate = bytes(chars)
    padded_hash = sha1(candidate + zero_pad).digest()
    if padded_hash in padded_targets:
        found[padded_targets[padded_hash]] = candidate
    if sha1(candidate).digest() == last_target:
        found[5] = candidate
    if len(found) == 6:
        break

print(found)
```

piece 顺序对应文件 `1,3,5,2,6,4`，而 flag 要按文件名 `1,2,3,4,5,6` 排列。恢复出的映射为：

```text
1 -> p2p_    2 -> iS_i    3 -> nter
4 -> eSti    5 -> ng_2    6 -> WPIB
```

重排、拼接并用提示中的 SHA-256 校验：

```python
by_piece = [found[i] for i in range(6)]
file_order = [by_piece[0], by_piece[3], by_piece[1],
              by_piece[5], by_piece[2], by_piece[4]]
flag = b"moectf{" + b"".join(file_order) + b"}"
assert sha256(flag).hexdigest() == "de5d94f22a9b8eab09779102a0fcc9c566880f7807d359da6f27723f3b881584"
print(flag)
```

最终得到：

```text
moectf{p2p_iS_intereSting_2WPIB}
```

## 方法总结

Torrent 的 `pieces` 是对整个文件流按固定长度切块后的 SHA-1 串，不是“一文件一哈希”。本题利用 pad 文件把五个 4 字节片段恰好补成完整 piece，再把最后一个片段单独留在尾部。解题关键是同时理解 bencode、文件拼接顺序、piece 边界和最终按文件名重排的区别。
