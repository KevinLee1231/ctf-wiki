# Weird Message

## 题目简述

附件 `weird_message.txt` 是一段很长、字符集符合 Base64 的文本。题面中的 “dubstep” 和 “Nintendo” 共同指向反复出现的 “drop” 与 “64”，实际结构是多层 Base64 套娃。

## 解题过程

每次解码后，结果仍然只由 Base64 常见字符组成。连续解码 21 层即可得到明文。部分中间层省略了末尾填充，因此脚本要在每轮补齐 `=`：

```python
import base64
from pathlib import Path

data = Path("weird_message.txt").read_bytes().strip()

for layer in range(1, 22):
    data += b"=" * (-len(data) % 4)
    data = base64.b64decode(data)
    print(layer, len(data))

print(data.decode())
```

最终输出：

```text
UMDCTF-{encoded_l00p_f0r_days_on_3nd}
```

该结果的 SHA-256 为 README 给出的 `aed3d9cebf70064b431fdb6f8226f1087c29e9406286298674d1061c9782cea7`。

## 方法总结

多层编码题的重点是建立稳定的循环条件，并处理不规范的 padding，而不是手工复制到在线工具。可以在每层输出长度和前几个字节，确认数据持续收敛且没有误解码；最终再用官方摘要校验精确大小写和数字。
