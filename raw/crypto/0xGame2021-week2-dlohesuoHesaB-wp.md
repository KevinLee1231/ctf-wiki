# week2dlohesuoHesaB

## 题目简述

题名 `dlohesuoHesaB` 倒序后是 `BaseHousehold`，附件名 `terces.txt` 倒序后是 `secret.txt`。题目先要求整体逆序，随后连续解开若干层 Base 编码，个别中间层还可能再次倒序。

## 解题过程

先将附件内容整体反转：

```python
from pathlib import Path

data = Path("terces.txt").read_bytes()[::-1]
```

后续不必依赖在线工具或外部爆破项目。下面以广度优先方式尝试 Base16、Base32、Base64、Ascii85、Base85 与逆序操作；每个状态只访问一次，发现 `0xGame{...}` 后立即停止：

```python
import base64
import re
from collections import deque
from pathlib import Path

def padded(data, block):
    return data + b"=" * (-len(data) % block)

def decode16(data):
    return base64.b16decode(data.upper())

def decode32(data):
    return base64.b32decode(padded(data.upper(), 8))

def decode64(data):
    return base64.b64decode(padded(data, 4), validate=True)

decoders = {
    "reverse": lambda data: data[::-1],
    "base16": decode16,
    "base32": decode32,
    "base64": decode64,
    "ascii85": base64.a85decode,
    "base85": base64.b85decode,
}

start = Path("terces.txt").read_bytes().strip()[::-1]
queue = deque([(start, ["reverse"] )])
seen = {start}

while queue:
    data, path = queue.popleft()
    match = re.search(rb"0xGame\{[^}\r\n]+\}", data)
    if match:
        print(" -> ".join(path))
        print(match.group().decode())
        break
    if len(path) >= 24:
        continue
    for name, decoder in decoders.items():
        try:
            result = decoder(data.strip())
        except Exception:
            continue
        if result and result not in seen:
            seen.add(result)
            queue.append((result, path + [name]))
else:
    raise ValueError("在限定层数内未找到 flag")
```

当前公开仓库没有保留本题的 `terces.txt` 或最终 flag，官方 Week 2 PDF 也只记录了“逆序后使用 BaseCrack 逐层解码”，因此无法从现存证据准确补写具体层序和 flag。正文保留的是能够对原附件直接运行的本地恢复方法。

## 方法总结

题名、附件名的倒序通常是在提示数据也要反转。多层 Base 题应记录每次变换并限制搜索深度；当原始附件缺失时，应明确证据边界，不能根据其他题目的格式臆造最终结果。
