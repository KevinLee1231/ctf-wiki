# 解不完的压缩包

## 题目简述

附件先用 999 层单成员 ZIP 消耗人工操作，最内层 `cccccccrc.zip` 再把口令拆成多个极短文本成员。真正有价值的两步分别是自动递归解压，以及利用“已知文件长度 + ZIP 目录中的 CRC32”恢复短明文。

## 解题过程

不要把下一层文件名写死为当前循环变量；每次解压后，从本层成员列表中确认唯一的 ZIP，再把它作为下一轮输入。这样即使层号命名有偏差也不会重复解压同一个文件。

```python
from pathlib import Path
from zipfile import ZipFile

workdir = Path("unpacked")
workdir.mkdir(exist_ok=True)
current = Path("999.zip")

for level in range(999):
    with ZipFile(current) as archive:
        zip_members = [name for name in archive.namelist() if name.lower().endswith(".zip")]
        archive.extractall(workdir)

    if len(zip_members) != 1:
        raise ValueError(f"第 {level + 1} 层应只有一个 ZIP，实际为 {zip_members!r}")
    current = workdir / zip_members[0]

print(current)
```

最内层压缩包中的 `pwd1.txt`、`pwd2.txt` 等成员都只有 2 字节。ZIP 中保存了每个成员的未压缩长度与 CRC32；对于如此短的明文，可以枚举候选字符并计算 `zlib.crc32(candidate)`，命中目录记录值时就恢复了原文。这不是寻找 CRC 碰撞来伪造另一个文件，而是利用极小搜索空间做 CRC32 原像枚举。

现成的 ZIP CRC 恢复工具会依次报告成员名、2 字节长度、CRC32 值和枚举出的明文。截图中可见前两个结果分别为 `pwd1.txt -> *m`、`pwd2.txt -> :#`；继续处理其余短成员后，按 `pwd1.txt`、`pwd2.txt` 等文件名顺序拼接得到压缩包密码：

```text
*m:#P7j0
```

用该密码解开最终压缩包，得到：

```text
moectf{af9c688e-e0b9-4900-879c-672b44c550ea}
```

## 方法总结

多层压缩包应根据“本层实际产出的成员”推进，而不是依赖易错的层号假设。CRC32 只有 32 位且不是密码学哈希；当成员长度很短、字符集可控时，ZIP 中公开的 CRC32 足以作为枚举判据。若文件较长或字符集未知，搜索空间会迅速失控，不能把这种方法当作通用 ZIP 解密。
