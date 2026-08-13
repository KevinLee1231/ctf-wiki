# Shy Zipper

## 题目简述

附件是一个结构有效的 ZIP。内部文件只提供 Base64 提示和大量干扰数据，真正的 flag 被编码后追加在 ZIP 的 End of Central Directory（EOCD）记录之后。ZIP 解析器通常在 EOCD 结束处停止，因此尾随字节不会作为归档成员显示。

## 解题过程

EOCD 签名为 `PK\x05\x06`。其固定头长度是 22 字节，偏移 `EOCD + 20` 的两字节小端整数表示 ZIP comment 长度。因此真实结构末尾为：

$$
\text{eocd\_end}=\text{eocd\_offset}+22+\text{comment\_length}
$$

读取此位置之后的尾随数据，找出 Base64 字符串并解码：

```python
import base64
import re
from pathlib import Path

data = Path("shy_zipper.zip").read_bytes()
eocd = data.rfind(b"PK\x05\x06")
if eocd < 0:
    raise ValueError("EOCD not found")

comment_length = int.from_bytes(data[eocd + 20:eocd + 22], "little")
eocd_end = eocd + 22 + comment_length
trailing = data[eocd_end:]

for candidate in re.findall(rb"[A-Za-z0-9+/]+={0,2}", trailing):
    try:
        decoded = base64.b64decode(candidate, validate=True)
    except ValueError:
        continue
    if decoded.startswith(b"grey{"):
        print(decoded.decode())
```

恢复结果：

```text
grey{s0m3_Th1Ng5_4r3_b3tT3r_L3fT_uNz1Pp3d}
```

直接运行 `strings` 也可能看到尾部 Base64，这是非预期捷径；解析 EOCD 能准确解释数据为什么不在 ZIP 成员列表里，也更适用于尾部含二进制噪声的情况。

## 方法总结

- 核心技巧：按 ZIP EOCD 与 comment 长度计算规范结束位置，提取并解码结构之外的隐藏尾随数据。
- 识别信号：归档能打开但内部没有目标、文件尾仍有未被 ZIP 解析器消费的数据、题目强调“outside of structure”。
- 复用要点：优先用 `rfind` 定位最后一个 EOCD 候选，并考虑 comment 长度；不要假设 EOCD 固定等于文件末尾 22 字节。
